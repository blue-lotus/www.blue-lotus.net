---
title: EBCTF 2013 Quals for100 writeup
author: cbmixx
layout: post
permalink: /ebctf-2013-quals-for100-writeup/
categories:
  - Uncategorized
---
Just do as description

`rev(steg.unhideBin(unhex(rot13(un7z(carve(xor(unhex(rot13(unrar(ebCTF_Teaser_FOR100.rar))))))))))`

*   `unrar` and `un7z` needs the password, got it(`12` for `unrar`, `21` for `un7z`) by brute forcing.
*   `xor` refers to [XOR cipher][1] using 1 byte key. got it(`chr(66)`) by brute forcing.
*   `carve` the 7z file concatenated to a png file
*   `steg.unhideBin` was found in [RobinDavid / LSB-Steganography][2]

![Flag][3]

 [1]: http://en.wikipedia.org/wiki/XOR_cipher
 [2]: https://github.com/RobinDavid/LSB-Steganography
 [3]: http://www.blue-lotus.net/wp-content/uploads/2013/06/flag.png