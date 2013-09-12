---
title: hack_u_too bin500 writeup
author: hellok
layout: post
permalink: /hack_u_too-bin50/
categories:
  - writeup
---
bin500是典型的逆向分析题。题目:[hardcore][1]

包含简单的反调试器，算法逆向

<!--more-->

程序如下：

1.anti调试器。通过简单的补丁等可绕过

2.KEY生成分为3部分

第一部分

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/11.png" alt="" width="649" height="731" class="alignnone size-full wp-image-249" />][2]

得出：

arg1 length 7

arg2 length 23

第二部分

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/21.png" alt="" width="634" height="771" class="alignnone size-full wp-image-250" />][3]

得出参数1和参数2个映射关系，和参数2的前4个字母。

其中4个变量关系如下：

v13|v14=0&#215;77

v13^v14=0&#215;46

v14^v15=0&#215;45

v15^v16=0x1c

v15|v15=0x7c

手动计算，或者暴力都好。

最终有2组解v13v14v15v16组成字符串为：w1th或s5pl（感谢aay提醒有2组解，同时告诉我们暴力很靠谱）

第三部分

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/3.png" alt="" width="624" height="481" class="alignnone size-full wp-image-251" />][4]

buildmemory:

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/41.png" alt="" width="475" height="522" class="alignnone size-full wp-image-257" />][5]

这里说下buildmemory的算法

buildmemory输入1个char

他会在内存上构建如下块

    001AFBB8 33 36 34 00 33 31 32 00 364.312.
    
    001AFBC0 32 36 30 00 32 30 38 00 260.208.
    
    001AFBC8 31 35 36 00 156.
    

变幻1:去除中间的\X00,

010333AC 31 35 36 32 30 15620

变幻2:

每5位循环交换位置，这里延伸了典型的一个变幻

    010333AC 32 35 30 36 31 25061
    

内存块为全局变量，所以外面也可使用

构建内存块结束后和a24557397004443变量比较，可以看到每5位读取一次

do while循环5次，分别读取：

    42343 .......
    
    14941 .......
    
    45743 .......
    
    25031 .......
    
    57083 .......
    

根据上面的映射关系

    15620变为25061
    
    answer变为42343
    

所以answer为32443

324为我们需要的input*3的结果。

324/3=108(见buildmemory算法)

所以input为108,chr(108)为‘l’，下4个分别为ov3~

这时题目提示：

Congrats! You did it!

If you find a correct username, then flag is password.

发现资源文件里面可以找到注册名

k0hyacu

最终有2组解

k0hyacu fr0m&#95;hacky0u&#95;w1th_l0v3~

k0hyacu fr0m&#95;hacky0u&#95;s5pl_l0v3~

组1可成功提交

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/hardcore.exe
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/11.png
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/21.png
 [4]: http://www.blue-lotus.net/wp-content/uploads/2012/12/3.png
 [5]: http://www.blue-lotus.net/wp-content/uploads/2012/12/41.png