---
title: Codegate 2013 bin 300 writeup
author: scdeny
layout: post
permalink: /codegate-2013-bin-300-writeup/
categories:
  - CTF
  - writeup
---
Bin300 is a win32 EXE, run it like this:

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/bin300.png" alt="" width="245" height="113" class="aligncenter size-full wp-image-393" />][1]

It seems like need a password.

At 004029c0, wo found the password checking.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/passchk.png" alt="" width="615" height="97" class="aligncenter size-full wp-image-394" />][2]

Just brute jump out of the checking, the function then released a file named “8c12…..exe.unprotected.exe”.

The file is stored in the end of the Bin300.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/eof.png" alt="" width="380" height="74" class="aligncenter size-full wp-image-395" />][3]

Launch the unprotected EXE but we get the alert:

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/err.png" alt="" width="572" height="162" class="aligncenter size-full wp-image-396" />][4]

Check the PE header, we found the Version Setting is 7.1, and then correct it to 5.1.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/ver.png" alt="" width="271" height="59" class="aligncenter size-full wp-image-397" />][5]

As the CodeGate give the hint:

[Bin 300 Hint] Routine that makes the &#8220;Note&#8221;

[Bin 300 Hint2] Find out notes that are created in somewhere where is not 1,2,3 difficulty level,

We found the routine which makes the “Note”:

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/makenotes.png" alt="" width="509" height="166" class="aligncenter size-full wp-image-398" />][6]

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/default.png" alt="" width="530" height="256" class="aligncenter size-full wp-image-399" />][7]

After case 1,2,3, there is a default case, where the GraphBuffer2 is filled from note_default. It must be noted that there are 2 graph buffers.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/graph_buff1.png" alt="" width="358" height="67" class="aligncenter size-full wp-image-405" />][8]  
In other cases, the GraphBuffer1 is filled. And the GraphBuffer1 is the actual output buffer.

We launch the program in OllyDbg, at the “switch” point we modify the “Difficulty” to 4(or any other value but not 1,2,3) then the “case default” can be covered.

At the “case default”, modify the code “pOutGraph = pGraphLine2&#95;1 &#8211; 1;” into “pOutGraph = pGraphLine &#8211; 1;” to change the output buffer to GraphBuffer1. At the asm level, it’s change the “mov ecx, [ebp+pGraphLine2&#95;1]” into “lea ecx, [edi]”.

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/out.png" alt="" width="369" height="109" class="aligncenter size-full wp-image-400" />][9]

F9……

Wow, the flag is printed.

EYE IS PRECIOUS

[<img src="http://www.blue-lotus.net/wp-content/uploads/2013/03/graph.png" alt="" width="480" height="443" class="aligncenter size-full wp-image-401" />][10]

 [1]: http://www.blue-lotus.net/wp-content/uploads/2013/03/bin300.png
 [2]: http://www.blue-lotus.net/wp-content/uploads/2013/03/passchk.png
 [3]: http://www.blue-lotus.net/wp-content/uploads/2013/03/eof.png
 [4]: http://www.blue-lotus.net/wp-content/uploads/2013/03/err.png
 [5]: http://www.blue-lotus.net/wp-content/uploads/2013/03/ver.png
 [6]: http://www.blue-lotus.net/wp-content/uploads/2013/03/makenotes.png
 [7]: http://www.blue-lotus.net/wp-content/uploads/2013/03/default.png
 [8]: http://www.blue-lotus.net/wp-content/uploads/2013/03/graph_buff1.png
 [9]: http://www.blue-lotus.net/wp-content/uploads/2013/03/out.png
 [10]: http://www.blue-lotus.net/wp-content/uploads/2013/03/graph.png