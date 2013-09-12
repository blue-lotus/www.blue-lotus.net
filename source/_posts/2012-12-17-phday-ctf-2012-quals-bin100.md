---
title: PHDay CTF 2012 Quals BIN100
author: hellok
layout: post
permalink: /phday-ctf-2012-quals-bin100/
categories:
  - CTF
  - writeup
---
bin100为2进制分析题  
<!--more-->

我们直接运行程序发现

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/1-300x60.png" alt="" width="300" height="60" class="alignnone size-medium wp-image-171" />][1]

提示我们出错

载入IDA。发现主要5个函数。

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/2.png" alt="" width="262" height="82" class="alignnone size-full wp-image-173" />][2]

函数我们分别重命名为f1->f5

读代码。5个函数功能大概如下,f1到f4函数为字符串变幻。f5函数实际最终调用了exec(calc)并且死循环。

f1 input 8 return 0

f2 input 32 return 0

f3 input>=16 xor first 16 return 1

f4 8>==xor return 0

f5 call calc exec return 0

但源程序这5个函数都没调用，只是取了地址

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/4-258x300.png" alt="" width="258" height="300" class="alignnone size-medium wp-image-174" />][3]

同时结合题目提示，得知此题是要我们把NOP给填充补全

这里我们填上call address即可

一共4处NOP需要填充

最终，邓大侠完美找到了填充顺序

顺序为

f4&#8211;>f1&#8211;>f2&#045;&#8211;||>f3

重新运行程序即可得到KEY

需要注意个是中间有个资源释放和本题无关。

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/5.jpg" alt="" width="185" height="158" class="alignnone size-full wp-image-178" />][4]

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/1.png
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/2.png
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/4.png
 [4]: http://www.blue-lotus.net/wp-content/uploads/2012/12/5.jpg