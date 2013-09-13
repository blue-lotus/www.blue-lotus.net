---
title: ASIS CTF memdump writeup
author: kelwin
layout: post
permalink: /asis-ctf-memdump-writeup/
comments: true
categories:
  - CTF
  - writeup
---
This is my first time to solve memory forensic challenge. I learn to use [volatility][1] from this [post][2].

Indicated from the memory dump strings, we know the system is `Ubuntu 12.04` with the kernel of `vmlinuz-3.5.0-23-generic`. After building a profile (the step by step procedure is [here][3]), we can use the commands in volatility.

    $ vol.py --profile=LinuxUbuntu1204x64 -f mem.dump linux_pslist
    ...
    0xffff88000acf4500 udevd        8112    0       0      0x000000000d06e000 2013-08-26 12:35:50 UTC+0000
    0xffff88000d54ae00 asis-ctf     9425    1000    1000   0x000000000c8c9000 2013-08-26 12:48:54 UTC+0000
    0xffff88000acf2e00 nano         15584   1000    1000   0x000000000d677000 2013-08-26 13:13:42 UTC+0000
    ...
    

We noticed that there is a process called `asis-ctf`, which seems to provide the flag. Then we dump the executable file from memory of the process.

    $ vol.py --profile=LinuxUbuntu1204x64 -f mem.dump linux_proc_maps -p 9425
    Volatile Systems Volatility Framework 2.3_beta
    Pid      Start              End            Flags               Pgoff Major  Minor  Inode      File Path                    
    9425 0x0000000000400000 0x0000000000401000 r-x                   0x0    252      0     393333 /home/netadmin/asis-ctf      
    9425 0x0000000000600000 0x0000000000601000 r--                   0x0    252      0     393333 /home/netadmin/asis-ctf      
    9425 0x0000000000601000 0x0000000000602000 rw-                0x1000    252      0     393333 /home/netadmin/asis-ctf
    ...
    $ vol.py --profile=LinuxUbuntu1204x64 -f mem.dump linux_dump_map -p 9425 -D output
    $ hexdump -C -n 10 task.9425.0x600000.vma
    00000000  7f 45 4c 46 02 01 01 00  00 00                    |.ELF......|
    $ cat task.9425.0x600000.vma task.9425.0x601000.vma > asis-ctf
    $ file asis-ctf
    asis-ctf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), stripped
    

Here we get the ELF file, after doing some RE, we got the flag by runing the script below:

    table_s = """42 49 55 52 4c 41 57 4e 64 5f 69 37 69 31 3e 63
    6b 65 6c 33 3b 34 3d 65 3f 65 6f 63 47 31 75 36
    72 66 42 62 4a 65 75 39 49 66 48 34 4d 32 4a 34
    4e 37 4e 32 4d 35 55 65 50 37 82 32 84 61 52 35
    83 39 85 61 53 34 89 39 8b 64 26"""
    table = []
    for c in table_s.replace("\n", " ").split(" "):
        n = int("0x" + c, 16)
        table.append(n)
    
    flag = ""
    for i in range(0x25):
        a = 2 * i
        c = table[a] - i - 1
        flag += chr(c)
    print flag
    
    $ python asis-ctf.py
    ASIS_cb6bb012a8ea07a426254293de2bc0ef
    

The ELF file asis-ctf I got from the mem.dump is still not able to run, that&#8217;s why RE is still needed. Does anyone have an idea to extract an runnable asis-ctf from the memory? Please tell me;-)

 [1]: https://www.volatilesystems.com/default/volatility
 [2]: http://blog.lse.epita.fr/articles/59-ebctf-2013-for100.html
 [3]: http://code.google.com/p/volatility/wiki/LinuxMemoryForensics
