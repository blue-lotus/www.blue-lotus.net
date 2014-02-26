---
title: Codegate CTF Quals 2014 Clone Technique writeup
author: yhy
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-clone-technique-writeup/
comments: true
categories:
  - CTF
  - writeup
---

Clone_Techique is a 250 points reverseme. The given program is a win32 PE. 

After load it in IDA pro, we can see the main function contains a loop which calls `CreateProcessW` many many times, that explains the title "clone technique". Before `CreateProcessW`, command string is prepared by `wsnprintf`, in which add some arguements when create new process. Except that, nothing special in main function.

    if ( (unsigned int)count <= 0xD0000000 )
    {
        count = v1;
        dword_409754 = v6;
        GetModuleFileNameW(0, &Filename, 0x12Cu);
        do
        {
            if ( (unsigned int)dword_409758 > 0x190 )
            return 0;
            ++dword_409758;
            wsprintfW(&CommandLine, L"\"%s\" %u %u %u", &Filename, v1, v6, dword_409758);
            CreateProcessW(0, &CommandLine, 0, 0, 0, 0, 0, 0, &StartupInfo, &ProcessInformation);
            WaitForSingleObject(ProcessInformation.hProcess, 0xFFFFFFFFu);
            GetExitCodeProcess(ProcessInformation.hProcess, &ExitCode);
            v1 = ExitCode;
            v6 = sub_401280(ExitCode ^ v6, ExitCode % 0x1E);
        }
        while ( ExitCode );
        result = 0;
    }

It took me sometime to figure out where to store the flag. Finally I notice a strange string near the const string used by `wsprintfW`.


    .data:00407030 magic_string    db 0Fh,'帪9=^?▃h',0Ch,'=嫮判{',9,'4叮g],0,0
    .data:00407030                                         ; DATA XREF: sub_401160+90o
    .data:0040704C ; const WCHAR aSUUU
    .data:0040704C aSUUU:                                  ; DATA XREF: real_main+128o
    .data:0040704C                 unicode 0, <"%s" %u %u %u>,0
    .data:00407068                 align 10h

Using Xref, IDA lead me to a interesting function. Noticing the `GetCommandLineW` call and `WriteProcessMemory` call, I believe it should be the key functin. 

But there is some anti-decompile technique disable IDA to decompile this function. 

    add     esp, 80h
    sub     esp, 80h

By filling the line shown above with nop. The IDA successfully decompile it.

    v0 = GetCommandLineW();
    v5 = CommandLineToArgvW(v0, &pNumArgs);
    if ( pNumArgs == 4 )
    {
        Buffer = _wtoi(v5[1]);
        v7 = _wtoi(v5[2]);
        dword_409758 = _wtoi(v5[3]);
    }
    else
    {
        Buffer = 0xA8276BFAu;
        v7 = 0x92F837EDu;
        dword_409758 = 1;
    }
    memset(sub_401070(&magic_string, Buffer, v7), 0, 28u);
    Buffer ^= 0xB72AF098u;
    v7 ^= v7 * Buffer;
    lpBaseAddress = (char *)&unk_409748 + 4;
    WriteProcessMemory((HANDLE)0xFFFFFFFF, (char *)&unk_409748 + 4, &Buffer, 4u, &NumberOfBytesWritten);
    lpBaseAddress = (char *)&unk_409750 + 4;
    return WriteProcessMemory((HANDLE)0xFFFFFFFF, (char *)&unk_409750 + 4, &v7, 4u, &NumberOfBytesWritten);

It is clear that the program compute the magic string with arguments. And then memset the new string with 0 immediately. The new string is very likely to be the flag. But we must know the correct arguements. Next step we need to find out the all the arguements when createprocess

Using procmon, it is easy to monitor all newlly created process and log them. Export them all into XML file, and do some scripting, and then we get all the arguements when create process.

![1]

Now it very close to the flag, but for me the pain just begins. Not familiar with debugging multi-process in windows, I decide to totally reverse the compute function. Luckily, with help of IDA it's not that diffcult, but still took me some time. 

The most frustrating part is the ROL function provided by IDA don't behave the same with the reverseme. Using a debugger, I set a breakpoint at the key function and compare the results of my own code and that of reverseme. 

With a rewritten key function, just enumerate all the parameters we dumped. Look over the 800+ results, we find:

    And Now His Watch is Ended

That's the flag!

 [1]: http://www.blue-lotus.net/images/2014/procmon.png
