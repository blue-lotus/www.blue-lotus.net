---
title: PHDay CTF 2012 Quals PWN 400
author: fqj
layout: post
permalink: /phday-ctf-2012-quals-pwn-400/
categories:
  - CTF
  - writeup
---
PWN400是一个用python编写的应用，运行在服务端上。

首先用 eval(&#8220;a=&#8221; + raw_input()) 的形式做了一个jail让我们无法import其他包的东西

然后在Reader函数用一个@guard修饰器限制参数必须是字符串，Reader函数内也要参数必须是那几个log文件的文件名之一。

大体的样子是这样的：

    def reader(open_file=open, XXXXXX不记得了):
         @guard(filename=str)
         def _Reader(filename):
              XXXX不记得了
         return _Reader
    Reader=reader()
    

然后在Reader函数定义完之后，在&#95;&#95;builtins&#95;&#95;里删除了open、eval、type等等的builtin函数。然后删除了&#95;&#95;builtins&#95;&#95;等危险的全局变量，然后从每一个object中删除了&#95;&#95;closures&#95;&#95;

所以说如果你要用open，那没有。要用Reader读取其他文件不行。要import不行。

这个问题的方案是，他删除了&#95;&#95;closures&#95;&#95;但没有删除func_closure

Reader.func&#95;closure[0].cell&#95;contents.func&#95;closure[0].cell&#95;contents(&#8216;/etc/passwd&#8217;).read()

用这个就可以了。

func_closure[X]是一个cell到一个object，cell似乎就是python中用来记引用的东西。

首先Reader是一个被修饰的函数，第一层闭包外面就是原来的Reader函数和修饰器的参数。

原来的Reader函数再外面的闭包就有open_file=open这个东西，就是open函数，所以用来打开/etc/passwd读取就好了。