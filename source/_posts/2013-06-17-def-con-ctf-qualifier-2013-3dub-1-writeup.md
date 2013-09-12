---
title: DEF CON CTF Qualifier 2013 3dub 1 Writeup
author: DeAdCaT___
layout: post
permalink: /def-con-ctf-qualifier-2013-3dub-1-writeup/
categories:
  - writeup
---
This is just a appetizer.

URL is

    http://badmedicine.shallweplayaga.me:8042/
    

When you see the webpage, you see some login form and you can login as anyone but not admin.

    badmedicine
    
    admin login disabled
    

The cat is in the cookie, you will see your username is in someway(some hash?) become the cookie, then I got the idea to change the cookie which represent admin.

I use firefox and burpsuite to deal with this idea and then I got the key.

It’s too easy and no law to see(lol, you should understand this piece of English via Chinese)