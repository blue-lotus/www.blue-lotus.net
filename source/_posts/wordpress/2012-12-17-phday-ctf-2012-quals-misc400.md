---
title: PHDay CTF 2012 Quals Misc400
author: kelwin
layout: post
permalink: /phday-ctf-2012-quals-misc400/
categories:
  - CTF
  - writeup
---
这道题给了一个50x50x50的三维迷宫，让你求出从给定入口到给定出口之间的路径。  
<!--more-->

  
三维迷宫是一层一层提供的，一共50层，每层大致长这个样子：

![maze_layer_example][1]

求解迷宫很简单，使用最简单的BFS算法求解并提交路径。提交完之后，发现服务器没有传回key，而是又传来一道新的迷宫题，使用相同的脚本继续解，一直到连续解完16个迷宫题，出现：

![maze_win][2]

紧接着就没有任何数据了，马上想到key隐藏在前面16个三维迷宫里面。在和Fish讨论和官方Hint（&#8221;A point of view dows matter&#8221;）的提示下，就想到迷宫的路径可能是某种字母的形状。于是打印从迷宫入口点所在的面看过去的平面图，发现果然是非常工整的字母！

![maze_key][3]

一开始将I误认为了1，所以无法得分。

 [1]: http://www.blue-lotus.net/wp-content/uploads/2012/12/maze_layer_example.jpg
 [2]: http://www.blue-lotus.net/wp-content/uploads/2012/12/maze_win.jpg
 [3]: http://www.blue-lotus.net/wp-content/uploads/2012/12/maze_key.png