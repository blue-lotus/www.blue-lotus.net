---
title: secuinside CTF Quals 2013 bigfile of secret writeup
author: ray
layout: post
permalink: /secuinside-ctf-quals-2013-bigfile-of-secret-writeup/
categories:
  - Uncategorized
---
The file to download is huge, but we can use an HTTP range request to download the last part of the file. `curl` has a dedicated option to specify the `Range` header:

    curl -r -500 http://119.70.231.180/secret_memo.txt
    

[1] http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.35