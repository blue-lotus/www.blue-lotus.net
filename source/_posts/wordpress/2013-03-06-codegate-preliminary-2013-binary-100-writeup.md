---
title: Codegate Preliminary 2013 Binary 100 Writeup
author: fish
layout: post
permalink: /codegate-preliminary-2013-binary-100-writeup/
categories:
  - CTF
  - writeup
---
We got 8938765cdf97403089c008e4d4b63d52.exe, which is a .Net program. It asks for a series of numbers, and we could reasonably guess that the flag will be shown if we input the correct key.

By using Reflector, we found that this assembly is obfuscated. But the good news is that only a few portion is obfuscated. Looking around the disassembled code, we found something interesting:

    public void TransFormable(string Data)
    {
        if (Data.Length == 0x10)
        {
            if (this.xorToString(AESCrypt.Encrypt(this.r.Text, KeyValue)) == this.lowkey)
            {
                MessageBox.Show(AESCrypt.Decrypt(this.StringToXOR(this.a.ByteTostring_t(this.c)), KeyValue));
            }
            else
            {
                MessageBox.Show("Do you know ?  " + AESCrypt.Decrypt(this.StringToXOR(this.a.ByteTostring_t(this.d)), KeyValue));
            }
        }
    }
    

Knowing that *this.r* is the TextBox, the logic of this part of code could be: checking if the encryption result of our input equals to *this.lowkey*. If they are equal, *this.c* will be decrypted and prompted (that should be the flag!). Alright, then let&#8217;s try decrypting *this.c*. Create a new .Net project in Visual Studio, copy necessary data and methods to that project and decrypt that array. Luckily, we got the flag: **code9ate2013 Start**.