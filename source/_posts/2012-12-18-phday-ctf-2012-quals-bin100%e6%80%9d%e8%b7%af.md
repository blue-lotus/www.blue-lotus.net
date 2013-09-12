---
title: PHDay CTF 2012 Quals BIN100思路
author: scdeny
layout: post
permalink: /phday-ctf-2012-quals-bin100%e6%80%9d%e8%b7%af/
categories:
  - CTF
  - writeup
---
进入实验室后，第二次参加CTF比赛，第一次做出题来，总算入门了，下面总结一下比赛心得，主要讲述一下我做题的思路，以供入门新手借鉴，总结经验，锻炼队伍。

IDA打开binary后，习惯性的F5，可以看到一个很简单的main函数

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/main.jpg" alt="" width="492" height="227" class="aligncenter size-full wp-image-198" />][1]

这个main函数主要完成了三个功能：

1.把“HoppaKey”字符串拷贝到一个0&#215;400长的缓冲区里，猜想其与Key有关，命名其为Flag；

2.取参数0赋给全局变量，参数0自然是binary的全路径，那么可以将这个全局变量命名为pSelfPathName；

3．将5个函数的地址赋给了5个全局变量，将5个函数命名为f1-f5，5个函数指针命名为pf1-pf5。

然后main打印了错误字符串后就奇怪的结束了，那就根据这些简单的线索来进一步分析吧

**第一线索：**

既然我们怀疑Flag缓冲区与KEY有关，那么查看引用Flag的地方，发现只有main引用了Flag，

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/flag.jpg" alt="" width="468" height="194" class="aligncenter size-full wp-image-196" />][2]

**第二线索：误入歧途，最复杂的代码不一定是KEY的藏身处**

进行不下去，那么转向下一疑点pSelfPathName，查看引用pSelfPathName的地方，

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/path.jpg" alt="" width="454" height="134" class="aligncenter size-full wp-image-200" />][3]

发现f5引用了pSelfPathName，赶紧跳到f5函数去分析，

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/f5.jpg" alt="" width="601" height="455" class="aligncenter size-full wp-image-195" />][4]

F5的代码好像干了点别的事，貌似很很有戏，其实这个时候已经中计了，最后证明这个函数对我们一点帮助都没有，其唯一功能就是误导我们。

看这个函数的代码，它在自身binary中找到一个名叫load_me的资源，并把kernel32.dll中的LoadLibraryA，GetProcAddress两个函数的地址写了进去，然后改写了一堆内存，最后调用了偏移3376处的一个函数，

越看越像那么回事，继续分析下去，用eXeScope打开binary，

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/load_me.png" alt="" width="658" height="317" class="aligncenter size-full wp-image-197" />][5]

找到名叫load_me的资源，用显眼的“MZ”一眼看出这个资源竟然是个未加密的PE文件，赶紧将其导出为loadme.exe，用IDA打开，跳到偏移3376的地方，虚拟地址为401930，F5函数

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/3376.png" alt="" width="619" height="373" class="aligncenter size-full wp-image-193" />][6]

这个函数更乱，可能跟f5函数改写的内存有关。。。。。。

双击执行试试，弹出了个框

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/dudu.png" alt="" width="185" height="158" class="aligncenter size-full wp-image-194" />][7]

It’s much easier dudu！！！被鄙视了，思路不对？

**第三线索：**

那回到main函数中的第三条线索，依次再浏览一下f1-f4，4个函数都干了些什么，发现都只是一些简单的数组运算，而且除了main以外没有任何其他地方引用了这4个函数的地址，很奇怪，这几个函数都没被调用，那他们到底是干嘛的呢？

**柳暗花明：原来只是个填空题**

这个时候陈钦同学过来说他注意到了好多nop指令，这一下提醒了我，刚开始也看到了main函数里的两串各6个字节的nop指令，没引起注意，现在回想，会不会是出题者故意将一些代码用nop覆盖呢？

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/nop.png" alt="" width="587" height="403" class="aligncenter size-full wp-image-199" />][8]

6个字节正好是call dword ptr [mem]（FF 15 ADDR），原来就是个填空题？难道是从f1-f5中选择两个函数填到这两个地方？

用OD加载binary，断在nop前面（40130A），在nop处写入汇编代码：call [00403378]（从pf1至pf5都试一遍）

[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/pf1-5.png" alt="" width="440" height="81" class="aligncenter size-full wp-image-201" />][9]

结果都不能通过中间的两次判断，直接跳到wrong（打印错误信息的代码）去了，f5直接就弹出了一堆“计算器”窗口，应该可以忽略f5了。

看看这两次判断要求什么条件：返回值al != 0 && fLoadMe != 0。f1-f5谁影响了这两个标志？

根据401309的代码可以看到这里把字符串“HoppaKey”作为参数压入了栈里。

f1 8=>16 ret 0 8字节字符串 转换为 16字节字符串

f2 32 ret 1 要求输入32字节字符串

f3 16=>32 ret 0 or 1; fLoadMe = (int)f5; 16字节字符串 转换为 32字节字符串

f4 >=8 ret 1 要求输入8字节以上字符串

貌似只有f3最接近，但输入字符串只有8字节，不满足16字节要求，f3就会返回0

**F5只是传说，不要迷恋5哥**

这时陈钦同学再次发现f4里也有nop串，粗心的我竟然一路F5，没留意一眼汇编代码，原来还有更多的nop串，为了不错杀一个换人，直接搜索nop串，发现总共有4个nop串，其中main中2个，图已贴出，在f1，f4里还各有一个nop串，这个4个函数应该各就各位了，拿出OD试吧。。。。。。

最终尝试出main中第1个nop串填充call f4，f4->f1，f1->f3，main中第2个nop串call f2。

不断的尝试，走了很多弯路，最后总算大功告成。

**希望咱们队伍以后多多交流，快快成长，早日杀进前十！！！**

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/main.jpg
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/flag.jpg
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/path.jpg
 [4]: http://www.blue-lotus.net/wp-content/uploads/2012/12/f5.jpg
 [5]: http://www.blue-lotus.net/wp-content/uploads/2012/12/load_me.png
 [6]: http://www.blue-lotus.net/wp-content/uploads/2012/12/3376.png
 [7]: http://www.blue-lotus.net/wp-content/uploads/2012/12/dudu.png
 [8]: http://www.blue-lotus.net/wp-content/uploads/2012/12/nop.png
 [9]: http://www.blue-lotus.net/wp-content/uploads/2012/12/pf1-5.png