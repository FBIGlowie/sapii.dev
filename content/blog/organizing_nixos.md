+++  
date = '2024-11-21T00:37:34-05:00'  
draft = false  
title = 'Organizing NixOS '  
+++

# Organizing NixOS 

Hello, the goal of this blog is being some guidelines for new NixOS users based on my experiences on **"making a modular config"**. You can of course organize it anyway you want. However, for most people it may cause decision paralysis 😭.

## Where you are

Let's assume you have a base configuration of already setup with flakes and home manager. We'll also assume you want multiple computers using one configuration, such as a home lab and your desktop.

```bash
[user@nixos]$ ls /etc/nixos

flake.nix 
flake.lock 
configuration.nix 
hardware-configuration.nix
home.nix
```

## Separating 

You will definitely want folders that separate your home configurations from each other as well as from your systems. You also want to separate where you keep your modules and pkgs. 

### Module Separation 
- Firstly, create a folder for your modules like ```/etc/nixos/modules ```.


- Separate home specific modules from system ones ( because home manager provides some options not available in system config and vice versa ). Folders like  ```modules/home``` for home configurations and  ``` modules/nixos``` or  ```modules/system``` for system configurations.

- It's also good idea to have a folder for modules every system needs like ```modules/global```. Here I like to keep a file like ```global.nix``` or functions that I add to ```lib``` to be used in any file.


### Home Separation
- Make a folder for home configurations like ```/etc/nixos/home``` or ```/etc/nixos/users```.

- Since every user will be on a system, it's probably best to make a connection between systems and users. You can name the folders like ```/home/docker@homelab``` or ```/home/docker-homelab``` so you don't end up with 4 folders for one user yet all for separate systems 😭.

- If you do decide to use home manager standalone via ```home-manager.lib.homeManagerConfiguration``` in the flake or any other way, it's quite similar but you can specify a location say in your ```/home``` and use different tools for rebuilding user config.

- Inside each home folder, you can make a folder for dotfiles and either manage them via GNU Stow or leave them unorganized and symlink them info place with  a option like ```home.file.".bashrc".source = ./dotfiles/bash/bashrc;```.

- For servers, such as a home lab, it's better to a more much "declarative" user management so I recommend you either don't use home manager or don't use it standalone.

### System Separation 

- You can separate system definitions inside a ```/etc/nixos/systems``` or ```/etc/nixos/nixos``` folder, we'll go with ```systems/```.

- You can just use the hostname for every folder like ```systems/desktop``` and ```systems/homelab```.

### Secrets

- Just keep secrets in a folder like ```/etc/nixos/secrets``` and use whatever tool you want.

### Pkgs

- I'd say just have a folder named ```pkgs``` and put stuff there.

## Closing thoughts 

It may take you multiple restructures and refactors to make a "perfect" config. It's ample you first start defining variables like system and hostnames then pivot to defining options in modules.

**Remember that nixos requires a default.nix to exist in every folder and that every file should be committed to local git repo, otherwise you get file not found error upon rebuild**

These guidelines here are what I followed for my config structure and have worked for me so I hope they work for you.

Now go get started!!!



