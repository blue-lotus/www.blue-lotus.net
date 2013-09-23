---
title: PHDay CTF 2012 Quals misc200
author: chenjj
layout: post
permalink: /phday-ctf-2012-quals-misc200/
categories:
  - CTF
  - writeup
---
此题题目为如下一张全黑的图片：  
[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/square-300x300.gif" alt="" title="square" width="300" height="300" class="alignnone size-medium wp-image-206" />][1]

很容易猜想到图片可能是背景颜色遮盖了图片内容，也有可能是图片的header等一些信息损坏，导致图片显示有问题。  
对于第一种猜想，用PS打开图片，就可以看到图片的各个像素点都是一样的颜色。  
对于第二种猜想，Mr.刘在学习gif的数据格式，校验每位是否错误，这种方法很费力费时。我采用的是，用Python读取图片像素，生成新的图片。

代码如下：  
[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/code1-300x68.png" alt="" title="code" width="300" height="68" class="alignnone size-medium wp-image-213" />][2]

打开新生成图片，显示结果如下：  
[<img src="http://www.blue-lotus.net/wp-content/uploads/2012/12/new1-300x300.gif" alt="" title="new" width="300" height="300" class="alignnone size-medium wp-image-209" />][3]

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/square.gif
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/code1.png
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/new1.gif