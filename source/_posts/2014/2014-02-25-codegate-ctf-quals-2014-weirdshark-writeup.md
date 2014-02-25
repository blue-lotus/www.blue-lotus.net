---
title: Codegate CTF Quals 2014 weirdshark writeup
author: zTrix
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-weirdshark-writeup/
comments: true
categories:
  - CTF
  - writeup
---

A pcap file is given here for analysis. check type using `file`

    # file weird_shark.pcap_f5f1e42dd398f18c43af89ba972b3ee7
    weird_shark.pcap_f5f1e42dd398f18c43af89ba972b3ee7: pcap-ng capture file - version 1.0

Open the file using wireshark, but no luck, wireshark reports malformed file format and refuse to open it.

So we need to extract the packets inside manually.

<!-- more -->

Soon I got the file format document [here](https://www.winpcap.org/ntar/draft/PCAP-DumpFileFormat.html), the file format is really simple, it's organized in blocks, and according to my comprehension, each block contains a single network frame packet.

According to `general block structure` section in the pcap document, we can easily get the block type, length, and content.

And as for the packet, there are several levels of network protocol headers, a brief hex view shows that the network traffic are HTTP requests and responses, so the protocol stack should be IP + TCP + HTTP

    | 6 + 6 bytes MAC addr + 2 bytes (ethertype 08 00) | 20 bytes IP header | 20 bytes TCP header | HTTP Header + HTTP Body |

the hard thing here to do manually pcap parse is to assemble TCP packets into byte stream, which require a good understanding of TCP control sequence. But we can assume that the network condition is good, no packet loss or retranssmission happens, just assemble them one by one and see what happens.

use the following python code to extract all http content

```python
import os, sys, struct

f = open('weird_shark.pcap_f5f1e42dd398f18c43af89ba972b3ee7').read()
total = len(f)

# skip the first two blocks, which seems broken, 0x80 and 0x9c are the block sizes respectively
index = 0x80 + 0x9c

w = open('http-content.bin', 'w')

while index < total:
    block_type = f[index:index+4].encode('hex')
    block_size = struct.unpack('<I', f[index+4:index+8])[0]
    captured_len = struct.unpack('<I', f[index+20:index+24])[0]
    packet_len = struct.unpack('<I', f[index+24:index+28])[0]
    print index, block_size, packet_len, captured_len, block_type
    w.write(f[index+28+54:index+28+packet_len])
    w.flush()
    index += block_size

w.close()
```

The result seems very promising,  just concat the packet contents one by one really works! which indicates the network condition is really good. There are several http requests inside

    GET / HTTP/1.1
    GET /favicon.ico HTTP/1.1
    GET /mario.png HTTP/1.1
    GET /favicon.ico HTTP/1.1
    GET /obama.bmp HTTP/1.1
    GET /codegate.jpg HTTP/1.1
    GET /multiple.pdf HTTP/1.1
    GET /grayhash.jpg HTTP/1.1

And to my surprise, the flag is not in `codegate.jpg`, but in `multiple.pdf`

FLAG = `FORENSICS_WITH_HAXORS`
