---
title: Codegate CTF Quals 2014 Automata writeup
author: Aluex
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-automata-writeup/
comments: true
categories:
  - CTF
  - writeup
---

First download the executive file automata. Run it and the program requires two things, a command and a code.

Open the file with IDA Pro and locate the references of the string 
	
	[=] Welcome to Automata System [=]
	
Decompile the context code and find that the program first pass the command and a address to a sub procedure located at 0x1339 (PIE is on).

In this procedure the program calculates the value of the string in base 37 and fill the address given with 3 int values:

	+ v1 = 42 - v2 - v3
	+ v2 = value_of_the_command % 17 + 1
	+ v3 = (value_of_the_command & 0xF) + 1

Here we know that the length of the code is 43 and its components.

Back to the main procedure, the program immediately check if the code given contains only 3 possible chars (whose ascii value mod 16 = 1,2,3 respectively and later we used '1','2', and '3') and the numbers of them (equals v1, v2 and v3 respectively).

After this step, the program will output

	[*] Verifying your code
 
Following comes the automata part.

The program then creates 3 pipes and uses fcntl() to mark the readding ends (whose file descriptors are fd+1, fd+3, and fd+5, where fd is the descriptor of network, or 4) with O_NONBLOCK.
Then it calls fork() to create a child process keeps reading from fd+1 and fd+3.
The child also calls a function at 0x11b2, which output a percentage calculated by the argument over 43, i.e., the length of the code.
Then the child calls 0x127f with the address of code, a index, three file descriptors and a signal.

In the function it first checks if it has come to the end of the code. If so, send the signal passed by the argument. Otherwise, add 1 to the index and write to one of the 3 file descriptors according to the char pointed by the index.

The left part of program creats another 7 child processes. Therefore we have an automata whose states are processes and the transfer function is implied by the pipes.

To find the transfer function, we attached gdb to the main process and watch the address where the argument of pipe() lies.

The pipe info is like this (numbers in the brackets are the state indices and those outside the brackets indicates the corresponding code):

	(1)<--(Start)
	(1)<--(1)1,2
	(2)<--(1)3
	(8)<--(2)3
	(3)<--(2)1
	(2)<--(3)1
	(5)<--(2)2
	(2)<--(5)2
	(2)<--(6)3
	(4)<--(3)2
	(3)<--(4)2
	(8)<--(3)3
	(6)<--(4)3
	(5)<--(4)1
	(4)<--(5)1
	(8)<--(5)3
	(7)<--(6)1,2
	(6)<--(7)1,2
	(8)<--(7)3
	(8)<--(8)1,2,3

Then we work out a figure (See Figure 1).

![1]
Figure 1

We also found that the program calls system() at function 0x124e, therefore the goal is to enter this address.

Trace back and it seems that only when SIGUSR1 is received can the program execute the key function.
 
Therefore we should make sure that after 43 codes are executed, the state is at any node other than the 8-th state.

Seems that for most of the states, code '3' will lead to 8-th state. Fortunately, the cycle connecting state 2, 3, 4, and 6 containing code '3' and we use it to easily construct the code correspounding to the command.

The command is filtered by strtok(), which split the command at white chars (' ', '\t', '\n', '\r','\"','\'').

Then we want to get a shell by system command, and we tried

	bash<&4>&4
	sh<&4>&4
	dash<&4>&4

It failed. Maybe because the organizer keeps monitor bash for security reasons.

	ls>&4
shows there is a file called key.

After all, we come up an idea to represent space with $(IFS)

	cat${IFS}key

And the flag is got.


 [1]: http://www.blue-lotus.net/images/2014/graph-automata1.png
