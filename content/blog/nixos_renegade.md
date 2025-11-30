+++
date = '2025-11-29T12:30:05-05:00'
draft = false
title = 'Nixos on Libre Renegade'
+++

# Nixos on Libre Renegade

Over thanksgiving I spent quite the time trying to setup Nixos on the Libre Board Renegade and I'd like to share some things here.

## Existing Documentation

So the only pieces of existing documentation I could find is the existing [nixos wiki](https://wiki.nixos.org/wiki/NixOS_on_ARM/Libre_Computer_ROC-RK3328-CC) page as well as [this blog post](https://pc-hass.de/blog/nixos-on-renegade/). The wiki page details the basic configuration setting up a standard NixOS image for aarch64 however the blog post says that none of the prebuilt images would work ( likely due to the a different U-Boot ) and so it solves this by writing the official Libre Computer U-Boot to the sdImage.

## The Configuration
### Base Configuration
```
{
  description = "NixOS SD card image for Libre Renegade (ROC-RK3328-CC)";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
  };
  outputs = { self, nixpkgs, ... }@inputs:
    let
      pkgs = import nixpkgs { system = "aarch64-linux"; };
      inherit (nixpkgs) lib;
    in
    {
      nixosConfigurations = {
        renegade = nixpkgs.lib.nixosSystem {
          system = "aarch64-linux";
          modules = [
            "${nixpkgs}/nixos/modules/installer/sd-card/sd-image-aarch64.nix"
            ];
        };
      };
      packages = {
        # For the target system
        aarch64-linux.renegade = self.nixosConfigurations.renegade.config.system.build.sdImage;

        # For your build system (x86_64)
        x86_64-linux.renegade = self.nixosConfigurations.renegade.config.system.build.sdImage;
      };

    };
}
```

This is a really simple system configuration flake that outputs the sdImage as a package, however **won't boot due to not having the correct U-Boot**. Build with
```
nix build .#renegade 
```

### Adding U-Boot
The blog post provides a really handy way to do this. After adding this, the image is bootable, this U-Boot image also enables USB, meaning the systemd service described in the wiki can be ommited ( it actually fails to start even ).

```
  sdImage.postBuildCommands = let
    rk3328-uboot = fetchurl {
      url = "https://boot.libre.computer/ci/roc-rk3328-cc";
      hash = "sha256-Y5yMmU/LyMVJi94vrYDpBM+4qwUfILwP+gVUToAXteA=";
    };
  in
  ''
    dd if=${rk3328-uboot} of=$img conv=fsync,notrunc bs=512 seek=64
  '';
```
{{< cc-source
  title="Nixos on Renegade"
  creator="Gereon Schomber"
  license="CC BY 4.0"
  url="https:/pc-hass.de/blog/nixos-on-renegade/"
  enable=true
>}}

### Adding bootloader
U-Boot uses a special text file almost like GRUB stored at `/boot/` for the boot entries, make sure to enable this if you are to rebuild on the Libre Renegade itself. ( I ended up rebuilding and garbage collecting without this enabled and it deleted the kernel, bricking my system ).   
```
boot.loader.generic-extlinux-compatible.enable = true;
```
### Configuring the fan 

Now the Libre Renegade has a special case you can buy called the "LoveRPi Active Cooling Case", which itself comes with a fan. Now the fan is pretty loud, even if you switch to the 3.3v pins instead of 5v. There was a [PR](https://github.com/libre-computer-project/libretech-wiring-tool/commit/f2509bb1f7705d6061782881367a622820cbce59) to the libretech-wiring-tool adding DeviceTree files specifically to enable PWM on pins 12. PWM is used to control the fan speed. NixOS already provides configuration options to enable DeviceTree files via `hardware.deviceTree.overlays`.

```
let
  pkgs = import nixpkgs { system = "aarch64-linux"; };
  repo = pkgs.fetchFromGitHub {
    owner = "libre-computer-project";
    repo = "libretech-wiring-tool";
    rev = "0078dc04ccb4daabc303d5cec1a4f4222db5b924";
    hash = "sha256-ZZvMetODFPamzqaMrf/EMvCS56ri0iKNG2UpKpOgQ4U=";
  };
  pwm-2-dts-file = "${repo}/libre-computer/roc-rk3328-cc/dt/pwm-2.dts";
  pwm-2-fan-dts-file = "${repo}/libre-computer/roc-rk3328-cc/dt/pwm-2-fan.dts";
in
{ config, pkgs, ... }:
{
  # Enable device tree support
  hardware.deviceTree.enable = true;

  # Apply the overlays
  hardware.deviceTree.overlays = [
    {
      name = "pwm2";
      dtsFile = pwm-2-dts-file;
    }
    {
      name = "pwm2-fan"; 
      dtsFile = pwm-2-fan-dts-file;
    }
  ];

  # Ensure the kernel has pwm-fan and pwm support
  boot.kernelModules = lib.mkAfter [
    "pwm-fan"
    "pwm-rockchip"
  ];
}

```
**The fan PWM declaration needs to be below the PWM 2 one because it depends on it.**

### Controlling the fan

Since the fan speed will be set based on you CPU temperature, you need to read that first. The common path for this is at `/sys/class/hwmon/`, however the paths after this differ, and you need to find them. The file for the CPU temp is `temp1_input` and the PWM file is at `/pwm1`.
I have this incredibly simple python script derived from the [PR](https://github.com/libre-computer-project/libretech-wiring-tool/pull/15#issuecomment-3537427128), there is also another [project](https://github.com/angelicadvocate/renegade-fan-control/tree/main?tab=readme-ov-file) that aims to have enable fan control, you can use their [script](https://github.com/angelicadvocate/renegade-fan-control/blob/main/fan_control.sh), it also provides [pictures](https://github.com/angelicadvocate/renegade-fan-control/blob/main/LibreRenegadePWM-img1.png) for where to connect the fan pins to the Renegade pins.
#### The Script:

```
from pathlib import Path
import time
fancontrol = Path("/sys/class/hwmon/hwmon0/pwm1")
cpu_temp = Path("/sys/class/hwmon/hwmon1/temp1_input")

cpu_handle = open(cpu_temp, "r")
fan_handle = open(fancontrol, "w")



while True:
    temp = int(cpu_handle.read()) / 1000
    cpu_handle.seek(0)
    fan_speed = 0
    if temp <= 50:
        fan_speed = 0
    elif temp <= 55:
        fan_speed = 80
    elif temp <= 65:
        fan_speed = 128
    elif temp <= 75:
        fan_speed = 192
    else:  # temp > 75
        fan_speed = 255
    fan_handle.write(str(fan_speed))
    fan_handle.seek(0)
    print(f"CPU Temp: {temp}, setting fan speed: {fan_speed}")
    time.sleep(1)
```
The service definition for this:

```
  systemd.services.simple-fan-control = {
    enable = true;
    wantedBy = [ "default.target" ];
    description = "Incredibly Simple python fan control script";
    serviceConfig = {
      Restart = "always";
      RestartSec = 5;
      ExecStart = ''${pkgs.python3}/bin/python ${fancontrol-script}'';
    };
  };
```
Beware, the script will start at boot but the PWM or CPU files won't exist yet, so it will fail and keep restarting until they do.
### Closing thoughts

I love this little thing, it serves as a good K3S control plane server. Make sure to enable binfmt in your system's nix configuration so you can cross compile.

# Full Configuration

```
{
  description = "NixOS SD card image for Libre Renegade (ROC-RK3328-CC)";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
  };
  outputs =
    { self, nixpkgs, ... }@inputs:
    let
      pkgs = import nixpkgs { system = "aarch64-linux"; };
      repo = pkgs.fetchFromGitHub {
        owner = "libre-computer-project";
        repo = "libretech-wiring-tool";
        rev = "0078dc04ccb4daabc303d5cec1a4f4222db5b924";
        hash = "sha256-ZZvMetODFPamzqaMrf/EMvCS56ri0iKNG2UpKpOgQ4U=";
      };
      pwm-2-dts-file = "${repo}/libre-computer/roc-rk3328-cc/dt/pwm-2.dts";
      pwm-2-fan-dts-file = "${repo}/libre-computer/roc-rk3328-cc/dt/pwm-2-fan.dts";
      fancontrol-script = pkgs.writeTextFile {
        name = "fancontrol-script";
        text = ''
          from pathlib import Path
          import time
          fancontrol = Path("/sys/class/hwmon/hwmon0/pwm1")
          cpu_temp = Path("/sys/class/hwmon/hwmon1/temp1_input")

          cpu_handle = open(cpu_temp, "r")
          fan_handle = open(fancontrol, "w")



          while True:
              temp = int(cpu_handle.read()) / 1000
              cpu_handle.seek(0)
              fan_speed = 0
              if temp <= 50:
                  fan_speed = 0
              elif temp <= 55:
                  fan_speed = 80
              elif temp <= 65:
                  fan_speed = 128
              elif temp <= 75:
                  fan_speed = 192
              else:  # temp > 75
                  fan_speed = 255
              fan_handle.write(str(fan_speed))
              fan_handle.seek(0)
              print(f"CPU Temp: {temp}, setting fan speed: {fan_speed}")
              time.sleep(1)
        '';
      };
      inherit (nixpkgs) lib;
    in
    {
      nixosConfigurations = {
        renegade = nixpkgs.lib.nixosSystem {
          system = "aarch64-linux";
          modules = [
            "${nixpkgs}/nixos/modules/installer/sd-card/sd-image-aarch64.nix"
            (
              {
                config,
                lib,
                pkgs,
                ...
              }:
              {
                systemd.services.simple-fan-control = {
                  enable = true;
                  wantedBy = [ "default.target" ];
                  description = "Incredibly Simple python fan control script";
                  serviceConfig = {
                    Restart = "always";
                    RestartSec = 5;
                    ExecStart = ''${pkgs.python3}/bin/python ${fancontrol-script}'';
                  };
                };
                sdImage.postBuildCommands =
                  let
                    rk3328-uboot = builtins.fetchurl {
                      url = "https://boot.libre.computer/ci/roc-rk3328-cc";
                      sha256 = "Y5yMmU/LyMVJi94vrYDpBM+4qwUfILwP+gVUToAXteA=";
                    };
                  in
                  ''
                    dd if=${rk3328-uboot} of=$img conv=fsync,notrunc bs=512 seek=64
                  '';
                boot.loader.generic-extlinux-compatible.enable = true;
                # Enable device tree support
                hardware.deviceTree.enable = true;

                # Apply the overlays
                hardware.deviceTree.overlays = [
                  {
                    name = "pwm2";
                    dtsFile = pwm-2-dts-file;
                  }
                  {
                    name = "pwm2-fan";
                    dtsFile = pwm-2-fan-dts-file;
                  }
                ];

                # Ensure the kernel has pwm-fan and pwm support
                boot.kernelModules = lib.mkAfter [
                  "pwm-fan"
                  "pwm-rockchip"
                ];

              }
            )

          ];
        };
      };
      packages = {
        # For the target system
        aarch64-linux.renegade = self.nixosConfigurations.renegade.config.system.build.sdImage;

        # For your build system (x86_64)
        x86_64-linux.renegade = self.nixosConfigurations.renegade.config.system.build.sdImage;
      };

    };
}

```
