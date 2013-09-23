---
title: secuinside CTF Quals 2013 PE_Time writeup
author: LittleFater
layout: post
permalink: /secuinside-ctf-quals-2013-pe_time-writeup/
categories:
  - CTF
  - writeup
---
PE_Time.exe is a traditional Windows Form Application.

Once executed, a window named ***MyEdit*** will be created and a **Window Procedure** located at 0x004010F0 will be registered to handle all the messages sent to this window.

![enter image description here][1]

After looking into the Window Procedure, we can easily find a piece of code that deal with the **WM_COMMAND** message &#8212; the message sent when the user selects a command item from a menu, when a control sends a notification message to its parent window, or when an accelerator keystroke is translated.

![enter image description here][2]

Looking down along those code, we find it use the **GetWindowTextA()** API to obtain a text that typed into an EditBox controls, and then do some conversion to each byte of the input text.

![enter image description here][3]

A similar Loop has been used to deal with each byte of the input text, here is a pseudocode for each Loop:

    while(LoopCount--)
    {
        InputText[i] -= LoopCount;
        InputText[i] ^= XorKey;
    }
    

After converting all the four bytes, it will compare it with a string ***&#8220;C;@R&#8221;***. Once the string is match, we will get a congratulation.

![enter image description here][4]

So the key must be what we input and it is very easy to be worked out:

    #givemekey.py
    str = 'C;@R'
    keystr = ''
    
    key = ord(str[0])
    for i in range(1, 6):
        key = key ^ 3
        key = key + i
    keystr = keystr + chr(key)
    
    key = ord(str[1])
    for i in range(1, 5):
        key = key ^ 4
        key = key + i
    keystr = keystr + chr(key)
    
    key = ord(str[2])
    for i in range(1, 4):
        key = key ^ 5
        key = key + i
    keystr = keystr + chr(key)
    
    key = ord(str[3])
    for i in range(1, 3):
        key = key ^ 6
        key = key + i
    keystr = keystr + chr(key)
    
    print keystr
    

Here is the key:

![enter image description here][5]

 [1]: http://www.blue-lotus.net/wp-content/uploads/2013/05/1.jpg
 [2]: http://www.blue-lotus.net/wp-content/uploads/2013/05/2.jpg
 [3]: http://www.blue-lotus.net/wp-content/uploads/2013/05/31.jpg
 [4]: http://www.blue-lotus.net/wp-content/uploads/2013/05/5.jpg
 [5]: http://www.blue-lotus.net/wp-content/uploads/2013/05/6.jpg