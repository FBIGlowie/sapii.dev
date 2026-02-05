+++
date = '2025-11-30T18:39:41-05:00'
draft = true
title = 'Unicode Shenanigans'
+++

# Intro to ASCII

If you have ever taken a computer science class, you probably remember about ASCII characters.
While it officially stands for "**American Standard Code for Information Interchange**", you can think of it as standard that linking numbers to letters (and control codes), officially called "**character encodings**". It encoded each letter as a seven-bit integer (0 to 127 values). 33 of the characters were control characters, which handled functions like **line feeds** and **carriage returns** for printers and their teletype systems.
### Early teletype system:
![](/images/Teletype.jpg "")
{{< cc-source
  title="Teletype-IMG_7287.jpg"
  creator="Rama & Musée Bolo"
  licensce="CC BY-SA 2.0"
  url="https://commons.wikimedia.org/w/index.php?curid=36769003"
  enable=true
>}}


# Intro to Unicode

Unicode itself came as an extension to ASCII. It still supports the same character encodings as ASCII (the literal numbers) as ASCII but is aimed to support all of the world's writing systems by using up to 4 **bytes** to store encoding numbers. As of UTF-8, it supports more than a million code points (sequence of bits, aka numbers). It has characters from almost every language and introduces special characters to append to character bytes to support the formatting of special characters, like ligatures for Arabian and the topic of this blog, **Diacritics**.


# Invisible characters

Before we get to Diacritics, we must learn about some introduction from Unicode, **the invisible and non-printable characters**. Going to [invisible-characters.com](https://invisible-characters.com/), you get a list of all the these types of characters. Some of them are like the simple Space, which you can highlight:
```
 
```
Or the Zero Width Space, which can't be highlighted:
```
​
```
