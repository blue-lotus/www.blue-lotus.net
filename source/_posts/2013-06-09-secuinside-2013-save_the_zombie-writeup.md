---
title: Secuinside 2013 save_the_zombie writeup
author: fish
layout: post
permalink: /secuinside-2013-save_the_zombie-writeup/
categories:
  - CTF
  - writeup
---
Two binaries are provided for this task, which are *Apache2* and *cli.exe*. *Apache2* is an ELF running under Linux, which is obviously packed, therefore static analysis seems impossible.

Really? Let&#8217;s statically analyze the core-dump. Run *Apache2*, let gdb attach to the process, and execute **generate-core-file**. A core-dump of the target process is generated, which contains all unpacked instructions. Now it&#8217;s time for IDA <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_biggrin.gif' alt=':D' class='wp-smiley' /> .

A quick analysis told us that the *Apache2* is a daemon listening on a server, waiting for zombies to connect back and register themselves (with a special header field in HTTP request). Their IPs and ports are logged in a plain text file /t/a. Look at the other executable *cli.exe*, and we found it to be the actual backdoor program that runs on compromized zombies. It could accept commands from attackers to list any directories or echo the content of any files. Hence we should find the zombie and ask it to read the flag file for us. But&#8230; where is the zombie?

It doesn&#8217;t take too much time before we found the *Apache2* is running on the specified server, and http://54.214.248.168/t/a is the zombies-list. So let&#8217;s write a script, which is extremely easy, and make it connect to that machine, list all files in the current working directory, and read the key file out.

We eventually got the key: **Night Of The Living Dead**