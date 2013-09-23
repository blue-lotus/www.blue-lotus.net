---
title: 'PlaidCTF 2013 Crypto compression(250)  writeup'
author: douniwan5788
layout: post
permalink: /plaidctf-2013-crypto-compression250-writeup/
categories:
  - CTF
  - writeup
---
We managed to get the source code for an encryption service running at

54.234.224.216 :4433 OK

    #!/usr/bin/python
    import os
    import struct
    import SocketServer
    import zlib
    from Crypto.Cipher import AES
    from Crypto.Util import Counter
    
    # Not the real keys!
    ENCRYPT_KEY = '0000000000000000000000000000000000000000000000000000000000000000'.decode('hex')
    # Determine this key.
    # Character set: lowercase letters and underscore
    PROBLEM_KEY = 'XXXXXXXXXXXXXXXXXXXX'
    
    def encrypt(data, ctr):
        aes = AES.new(ENCRYPT_KEY, AES.MODE_CTR, counter=ctr)
        return aes.encrypt(zlib.compress(data))
    
    class ProblemHandler(SocketServer.StreamRequestHandler):
        def handle(self):
            nonce = os.urandom(8)
            self.wfile.write(nonce)
            ctr = Counter.new(64, prefix=nonce)
            while True:
                data = self.rfile.read(4)
                if not data:
                    break
    
                try:
                    length = struct.unpack('I', data)[0]
                    if length &gt; (1&lt; CTR非常依赖Counter的唯一性。这是因为，如果你重用一个Counter，那么对两个密文分组的异或就将是相应的明文分组的异或。
    

由于zlib压缩第一个字节的明文也能猜到，于是考虑加密上是否有漏洞，但检查后方发现使用没有问题。

后经@zhugejw老师提醒此题名为compression开始重点关注zlib，看了lz77算法发现长度大于3Byte的重复串会被压缩，结合CTR流密码可以知道zlib压缩后的数据长度,找到解题方向：跟加密无关，通过构造数据根据返回的长度，猜解PROBLEM_KEY。如果发送的串与KEY有3位以上相同就会被压缩，于是先写了个脚本爆破4位，然后按位猜解。

    import socket,struct
    
    skt=socket.socket()
    skt.connect(("54.234.224.216",4433))
    nonce= skt.recv(8)
    print repr(nonce)
    
    fi = open('pyj.txt', 'a')
    
    data = list("ABCDEFGHIJKLMNOPQRST")
    
    for f1 in 'abcdefghijklmnopqrstuvwxyz_':
        for f2 in 'abcdefghijklmnopqrstuvwxyz_':
            for f3 in 'abcdefghijklmnopqrstuvwxyz_':
                for f4 in 'abcdefghijklmnopqrstuvwxyz_':
                    try:
                        data[0] = f1[0]
                        data[1] = f2[0]
                        data[2] = f3[0]
                        data[3] = f4[0]
                        sd=''.join(data)
                        print sd
    
                        skt.send(struct.pack('I',len(sd)))
                        skt.send(sd)
    
                        ciphertextlen= struct.unpack('I',skt.recv(4))[0]
                        ciphertext = skt.recv(ciphertextlen)
                        if ciphertextlen &gt;fi, ciphertextlen, repr(ciphertext)
                            fi.flush()
                    except:
                        print 'except'
                        skt.close()
                        skt.connect(("54.234.224.216",4433))
    skt.close()
    fi.close()
    

测试时发现即使不发送数据返回的长度也比20个Byte全不同的KEY小，于是可以判断KEY中本身就有重复

暴力跑出了  
> ime ays cri e_s *so met me*  
等串

以此为基准按位猜解跑出 crimes_pays ，但由于key本身含有重复，会影响按位猜解，于是卡住了，最终由@Slipper大神人肉猜解出KEY：  
`crime_sometimes_pays`

ps:其实对于lz77还是不太明白，个人理解4Byte的数据压缩为3Byte才会导致长度减小，爆破时不知道为什么有3Byte重复就长度减小了，请大牛们指点