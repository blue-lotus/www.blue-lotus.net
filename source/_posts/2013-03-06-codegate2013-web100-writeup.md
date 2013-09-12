---
title: Codegate2013 web100 writeup
author: yzm
layout: post
permalink: /codegate2013-web100-writeup/
categories:
  - writeup
tags:
  - Codegate2013
---
# Web100

*yuzeming@gmail.com*

一个登陆页面，需要以admin登录。  
核心代码如下

    <?php
    $ps = mysql_real_escape_string($ps);
    $ps = hash("whirlpool",$ps, true);
    $result = mysql_query("select * from users where user_id='$id' and user_ps='$ps'");
    ?>

可以注意到hash函数的第三个参数为true，使得hash函数返回的是二进制串。任何字符都可能出现在字符串内。  
如果`'='`出现在hash后的字符串内可能导致注入。

原理是当这个字符串带入查询后，查询串为`where user_ps='XXXX'='YYY'`。

*   第一步运算 `user_ps='XXXX'`得到一个int值。由于不相等应该等于0.
*   第二步运算 int值与字符串比较，会把字符串转换为int，转换结果也等于0。

这绕过了密码检验。

可以使用admin/364383登录这个系统。