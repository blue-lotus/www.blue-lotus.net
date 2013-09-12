---
title: ebctf 2013 crypto writeup
author: zTrix
layout: post
permalink: /ebctf-2013-crypto-writeup/
categories:
  - CTF
  - writeup
---
The problem offers 2 encrypted message and 1 secret key for one of them. And it&#8217;s our mission to decrypt the other message.

Plain message is divided into 16-bytes blocks, the encryption use sha256 to generate encryption cipher and xor the message to get encryption message.

So if we can get the first cipher used for encrypting the first message block, the rest can be calculated.

Use the secret key provided, we can decrypt one of the messages and it turns out to be an email.

> From: Vlugge Japie To: Baron van Neemweggen  
> ** Subj: Weekly update  
>  
> Boss,  
>  
> Sorry, I failed to get my hands on the information you requested.  
> Please don&#8217;t tell the bureau &#8211; I&#8217;ll have it next week, promise!  
>  
> Vlugge Japie</p> 
So it&#8217;s very likely the other message should also be an email, and if this is true, we can get the first 16 bytes of plain text: &#8220;From: Vlugge Jap&#8221;. Use xor, we can get the cipher: xor(cipher, &#8220;From: Vlugge Jap&#8221;)

The solution is straightforward now.

    import hashlib, string, sys
    
    def xor(a, b):
        l = min(len(a), len(b))
        return ''.join([chr(ord(x) ^ ord(y)) for x, y in zip(a[:l], b[:l])])
    
    def h(x):
        x = hashlib.sha256(x).digest()
        x = xor(x[:16], x[16:])
        return x
    
    inp = open('msg002.enc').read()
    msg = inp.decode('base64')
    k = xor(msg[:16], 'From: Vlugge Jap')
    
    out = ''
    for i in xrange(0, len(msg), 16):
        out += xor(msg[i:i+16], k)
        k = h(k + str(len(msg)))
    
    print out