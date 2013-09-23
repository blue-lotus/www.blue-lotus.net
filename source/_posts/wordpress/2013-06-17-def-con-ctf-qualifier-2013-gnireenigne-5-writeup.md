---
title: DEF CON CTF Qualifier 2013 gnireenigne 5 Writeup
author: fish
layout: post
permalink: /def-con-ctf-qualifier-2013-gnireenigne-5-writeup/
categories:
  - CTF
  - writeup
---
This is a x64 binary running on Linux. It binds to the first IP of eth0 and accepts connections at port 6823. The client should submit data in the following manner:

    4 bytes length | the first block | the second block
    

The two blocks have identical sizes, and the first field indicates the total size of two blocks. The total size should be between 48 and 1024.

After that, our input will be checked against two hashing algorithms. The first one is a standard MD5, and the MD5 values of two blocks cannot be identical, otherwise no key will be printed. The second one is an unknown hashing algorithm. When we found a collision in the second hashing algorithm, we could finally get the key.

We spent some time to fully reverse that algorithm:

    unsigned char BOX_[256] = { 0x07, 0x00, 0x00, 0x00, 0x0c, 0x00, 0x00, 0x00,
        0x11, 0x00, 0x00, 0x00, 0x16, 0x00, 0x00, 0x00,
        0x07, 0x00, 0x00, 0x00, 0x0c, 0x00, 0x00, 0x00,
        0x11, 0x00, 0x00, 0x00, 0x16, 0x00, 0x00, 0x00,
        0x07, 0x00, 0x00, 0x00, 0x0c, 0x00, 0x00, 0x00,
        0x11, 0x00, 0x00, 0x00, 0x16, 0x00, 0x00, 0x00,
        0x07, 0x00, 0x00, 0x00, 0x0c, 0x00, 0x00, 0x00,
        0x11, 0x00, 0x00, 0x00, 0x16, 0x00, 0x00, 0x00,
        0x05, 0x00, 0x00, 0x00, 0x09, 0x00, 0x00, 0x00,
        0x0e, 0x00, 0x00, 0x00, 0x14, 0x00, 0x00, 0x00,
        0x05, 0x00, 0x00, 0x00, 0x09, 0x00, 0x00, 0x00,
        0x0e, 0x00, 0x00, 0x00, 0x14, 0x00, 0x00, 0x00,
        0x05, 0x00, 0x00, 0x00, 0x09, 0x00, 0x00, 0x00,
        0x0e, 0x00, 0x00, 0x00, 0x14, 0x00, 0x00, 0x00,
        0x05, 0x00, 0x00, 0x00, 0x09, 0x00, 0x00, 0x00,
        0x0e, 0x00, 0x00, 0x00, 0x14, 0x00, 0x00, 0x00,
        0x04, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00,
        0x10, 0x00, 0x00, 0x00, 0x17, 0x00, 0x00, 0x00,
        0x04, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00,
        0x10, 0x00, 0x00, 0x00, 0x17, 0x00, 0x00, 0x00,
        0x04, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00,
        0x10, 0x00, 0x00, 0x00, 0x17, 0x00, 0x00, 0x00,
        0x04, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00,
        0x10, 0x00, 0x00, 0x00, 0x17, 0x00, 0x00, 0x00,
        0x06, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00,
        0x0f, 0x00, 0x00, 0x00, 0x15, 0x00, 0x00, 0x00,
        0x06, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00,
        0x0f, 0x00, 0x00, 0x00, 0x15, 0x00, 0x00, 0x00,
        0x06, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00,
        0x0f, 0x00, 0x00, 0x00, 0x15, 0x00, 0x00, 0x00,
        0x06, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00,
        0x0f, 0x00, 0x00, 0x00, 0x15, 0x00, 0x00, 0x00};
    
    void calc_hash(unsigned char* input, unsigned int size, unsigned char* output)
    {
        unsigned int* BOX = (unsigned int*)BOX_;
        unsigned char *buffer_ptr = NULL;
    
        unsigned char IV[] = {0x68, 0x69, 0x20, 0x6D, 0x79, 0x20, 0x6E, 0x61, 0x6D, 0x65, 0x20, 0x69, 0x73, 0x20, 0x69, 0x74};
    
        unsigned int counter_1 = 0;
        unsigned int ctr = 0;
    
        ctr = size + 1;
        while((ctr & 0x3f) != 0x38)
        {
            ++ctr;
        }
        buffer_ptr = (unsigned char*)malloc(ctr + 8);
        memcpy(buffer_ptr, input, size);
        *(buffer_ptr + size) = 0x80;
        unsigned int input_size_2 = size + 1;
    
        unsigned int buffer_A[2000] = {0};
    
        unsigned int string_1, string_2, string_3, string_4, string_5, ctr_3, string_6;
    
        while(ctr > input_size_2)
        {
            *(buffer_ptr + input_size_2) = 0;
            ++input_size_2;
        }
    
        unsigned int a_ = size << 3;
        unsigned char* b_ = buffer_ptr + ctr;
        *b_ = (unsigned char)a_;
        *(b_ + 1) = (unsigned char)(a_ >> 8);
        *(b_ + 2) = (unsigned char)(a_ >> 16);
        *(b_ + 3) = (unsigned char)(a_ >> 24);
    
        unsigned int c_ = size >> 0x1d;
        unsigned char* d_ = buffer_ptr + ctr + 4;
        *d_ = (unsigned char)c_;
        *(d_ + 1) = (unsigned char)(c_ >> 8);
        *(d_ + 2) = (unsigned char)(c_ >> 16);
        *(d_ + 3) = (unsigned char)(c_ >> 24);
        input_size_2 = 0;
    
        while(ctr > input_size_2)
        {
            // 0x40248f
            unsigned int ctr_2 = 0;
            while(ctr_2 <= 15)
            {
                // 0x402498
                unsigned char* e_ = buffer_ptr + input_size_2 + (ctr_2 << 2);
                unsigned int c = *(e_);
                unsigned int d = *(e_ + 1);
                d = d << 8;
                d = d | c;
                c = *(e_ + 2) << 16;
                d = d | c;
                c = *(e_ + 3) << 24;
                c = c | d;
                buffer_A[ctr_2] = c;
                ++ctr_2;
            }
            string_1 = *(unsigned int*)IV;
            string_2 = *(unsigned int*)(IV + 4);
            string_3 = *(unsigned int*)(IV + 8);
            string_4 = *(unsigned int*)(IV + 12);
    
            ctr_2 = 0;
            while(ctr_2 <= 63)
            {
                // 0x402535
                if(ctr_2 <= 0xf)
                {
                    string_5 = (~string_2 & string_4) | (string_3 & string_2);
                    ctr_3 = ctr_2;
                }
                // 0x402558
                else if(ctr_2 <= 0x1f)
                {
                    string_5 = (~string_4 & string_3) | (string_4 & string_2);
                    ctr_3 = ((ctr_2 << 2) + ctr_2 + 1) & 0xf;
                }
                // 0x402588
                else if(ctr_2 <= 0x2f)
                {
                    string_5 = (string_3 ^ string_2) ^ string_4;
                    ctr_3 = (ctr_2 * 3 + 5) & 0xf;
                }
                else
                {
                    // 0x4025b0
                    string_5 = (~string_4 | string_2) ^ string_3;
                    ctr_3 = ((ctr_2 << 3) - ctr_2) & 0xf;
                }
    
                // 0x4025ce
                string_6 = string_4;
                string_4 = string_3;
                string_3 = string_2;
                unsigned int ebx = string_1 + string_5;
                double sin_result = sin((double)(ctr_2 + 1));
                double d = sin_result * sin_result;
                double e = sqrt(d) * 4.294967296e9;
                unsigned long y = ebx + (unsigned long)e + buffer_A[ctr_3];
                ebx = (y << (BOX[ctr_2] & 0xff));
                d = sin((double)(ctr_2 + 1));
                d = d * d;
                e = sqrt(d) * 4.294967296e9;
                unsigned int x1 = string_5 + string_1 + (unsigned long)e + buffer_A[ctr_3];
                unsigned int x2 = 0x20 - BOX[ctr_2];
                string_2 = string_2 + ((x1 >> (x2 & 0xff)) | ebx);
                string_1 = string_6;
                ++ctr_2;
            }
            unsigned int x = *(unsigned int*)IV;
            x += string_1;
            *(unsigned int*)IV = x;
            x = *(unsigned int*)(IV + 4);
            x += string_2;
            *(unsigned int*)(IV + 4) = x;
            x = *(unsigned int*)(IV + 8);
            x += string_3;
            *(unsigned int*)(IV + 8) = x;
            x = *(unsigned int*)(IV + 12);
            x += string_4;
            *(unsigned int*)(IV + 12) = x;
            input_size_2 += 0x40;
        }
    
        free(buffer_ptr);
        memcpy(output, IV, 16);
        return;
    }
    

According to [Wikipedia][1], it is a revised MD5. The IV is changed to &#8220;hi my name is it&#8221;. To find a collision, you may download and compile [md5coll][2] by Patrick Stach, and execute it with new IV.

    fish@fish-Linux-Mint-13 ~/defconctf/tastycloud $ ./md5coll-optimized 0x6d206968 0x616e2079 0x6920656d 0x74692073
    block #1 done
    block #2 done
    unsigned int m0[32] = {
    0xf5ff5286, 0xd5187d3b, 0x30dd0e80, 0xca85cc02, 
    0x9747a14e, 0xc832351a, 0xcd333352, 0x16602937, 
    0x0632ecc1, 0x62b34dff, 0x83aee8f3, 0xd2c11b1d, 
    0x098950f8, 0x37dbacfa, 0x85abc37e, 0x1aac555a, 
    0x7ac11e49, 0x050463bb, 0xd3c58401, 0xd4d5dd63, 
    0xc6761a4f, 0x0dd022c4, 0x16696944, 0xe02da950, 
    0x747aeff5, 0x487cf5cc, 0xaae46a9a, 0x3e9ed485, 
    0xfd2968ee, 0x025bcaa4, 0x68f2289f, 0x4d353e49, 
    };
    
    unsigned int m1[32] = {
    0xf5ff5286, 0xd5187d3b, 0x30dd0e80, 0xca85cc02, 
    0x1747a14e, 0xc832351a, 0xcd333352, 0x16602937, 
    0x0632ecc1, 0x62b34dff, 0x83aee8f3, 0xd2c19b1d, 
    0x098950f8, 0x37dbacfa, 0x05abc37e, 0x1aac555a, 
    0x7ac11e49, 0x050463bb, 0xd3c58401, 0xd4d5dd63, 
    0x46761a4f, 0x0dd022c4, 0x16696944, 0xe02da950, 
    0x747aeff5, 0x487cf5cc, 0xaae46a9a, 0x3e9e5485, 
    0xfd2968ee, 0x025bcaa4, 0xe8f2289f, 0x4d353e49, 
    };
    

So here is the final Python script to get the key.

    import sys, os, time, struct, socket
    
    HOST = "tastycloud.shallweplayaga.me"
    PORT = 6283
    
    data1 = "\x86\x52\xff\xf5\x3b\x7d\x18\xd5\x80\x0e\xdd\x30\x02\xcc\x85\xca\x4e\xa1\x47\x97\x1a\x35\x32\xc8\x52\x33\x33\xcd\x37\x29\x60\x16\xc1\xec\x32\x06\xff\x4d\xb3\x62\xf3\xe8\xae\x83\x1d\x1b\xc1\xd2\xf8\x50\x89\x09\xfa\xac\xdb\x37\x7e\xc3\xab\x85\x5a\x55\xac\x1a\x49\x1e\xc1\x7a\xbb\x63\x04\x05\x01\x84\xc5\xd3\x63\xdd\xd5\xd4\x4f\x1a\x76\xc6\xc4\x22\xd0\x0d\x44\x69\x69\x16\x50\xa9\x2d\xe0\xf5\xef\x7a\x74\xcc\xf5\x7c\x48\x9a\x6a\xe4\xaa\x85\xd4\x9e\x3e\xee\x68\x29\xfd\xa4\xca\x5b\x02\x9f\x28\xf2\x68\x49\x3e\x35\x4d"
    data2 = "\x86\x52\xff\xf5\x3b\x7d\x18\xd5\x80\x0e\xdd\x30\x02\xcc\x85\xca\x4e\xa1\x47\x17\x1a\x35\x32\xc8\x52\x33\x33\xcd\x37\x29\x60\x16\xc1\xec\x32\x06\xff\x4d\xb3\x62\xf3\xe8\xae\x83\x1d\x9b\xc1\xd2\xf8\x50\x89\x09\xfa\xac\xdb\x37\x7e\xc3\xab\x05\x5a\x55\xac\x1a\x49\x1e\xc1\x7a\xbb\x63\x04\x05\x01\x84\xc5\xd3\x63\xdd\xd5\xd4\x4f\x1a\x76\x46\xc4\x22\xd0\x0d\x44\x69\x69\x16\x50\xa9\x2d\xe0\xf5\xef\x7a\x74\xcc\xf5\x7c\x48\x9a\x6a\xe4\xaa\x85\x54\x9e\x3e\xee\x68\x29\xfd\xa4\xca\x5b\x02\x9f\x28\xf2\xe8\x49\x3e\x35\x4d"
    
    s = socket.socket()
    s.connect((HOST, PORT))
    size = len(data1) + len(data2)
    
    s.send(struct.pack("I",size))
    s.send(data1)
    s.send(data2)
    
    print s.recv(1024)
    

Run the script, we get the key: **pringlelingus and redbull without a cause**

This is one of the few challenges that we didn&#8217;t manage to solve during the game. It&#8217;s a pity as we finally noticed the final hashing algorithm is a revised MD5, however we have no time to construct a collision. It&#8217;s all my fault that I didn&#8217;t realized it was a MD5 as soon as possible <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_razz.gif' alt=':P' class='wp-smiley' /> . Many thanks to NWMonster, Slipper and cq for their precious help.

 [1]: http://en.wikipedia.org/wiki/Md5#Pseudocode
 [2]: http://dl.packetstormsecurity.net/crypt/md/md5coll.c