---
title: SIGINT 2013 crypto300 satisfaction
author: slipper
layout: post
permalink: /sigint-2013-crypto300-satisfaction/
categories:
  - Uncategorized
---
<http://slipper.tk/writeup/2013/07/11/sigint-2013-crypto-satisfaction/>

# [SIGINT 2013 crypto300 satisfaction][1]

Description

    Something fishy going on:
    you need this
    188.40.147.108 2000
    This flag does not have a SIGINT_ prefix!
    

[server.rb][2]

[bf.rb][3]

The server only runs properly RSA signed brainfuck code.

    def valid?(code, cert)
      return false unless code =~ /\A[\[\]+\-\. ]+\Z/
      return false unless cert =~ /\A\d+\Z/
      sig = cert.to_i
      crc = crc32(code)
      test = crc.to_bn.mod_exp($d, $n)
      return sig == test
    end
    
      if(valid_cert)
        puts "got valid code: #{code.inspect}"
        ruby_code = bfdo(code)
        puts "got some ruby code: #{ruby_code.inspect}"
        client.puts eval(ruby_code)
    

If the brainfuck code which we submit only contains `+-[].`, then the sever will calculate the crc of the code and check if it matches the signature.  
Then how does `crc32()` work? The code in the script seems different from classic [CRC32][4].

    def crc32(str)
      mask = 0x04C11DB7
      crc = 0
      str.bytes.each do |byte|
        byte.to_s(2).rjust(8,"0").each_char do |bit|
          h_crc_bit = ( (( crc &amp; 0x80_00_00_00) != 0) ? 1 : 0)
          h_byte_bit = bit.to_i
          if  h_crc_bit == h_byte_bit
            crc = ((crc &lt;&lt; 1)^mask) &amp; 0xffffffff
          else
            crc = (crc &lt;&lt; 1) &amp; 0xffffffff
          end
        end
      end
      return crc
    end
    

Standard CRC32 is the remainder of a polynomial division of the target string. In practice it employs the finite field GF(2). XOR is equivalent to division in GF(2).  
As a hash function, CRC32 is easy to collision.  
In the code above, XOR was taken when `h_crc_bit == h_byte_bit`.  
However, in standard CRC32, it should be `h_crc_bit == 1 &amp;&amp; h_byte_bit == 1`  
Maybe the new crc32 is more likely to collision.

To study this algorithm, I rewrite it in C++.

    unsigned int CRC32_bit(unsigned char data[], int len)
    {
            unsigned int r = 0;
            unsigned int s;
            for (int i = 0; i &lt; len; i++) {
                    s = data[i] &lt;&lt; 24;
                    for (int j = 0; j &lt; 8; j++) {
                            if (!((r ^ s) &amp; 0x80000000))
                            //instead of if (r &amp; 0x80000000)
                                    r = (r &lt;&lt; 1) ^ POLY;
                            else
                                    r = r &lt;&lt; 1;
                            s = s &lt;&lt; 1;
                    }
            }
            return r;
    }
    

In this problem, it seems that we can't sign the code because we don't know the constants in `./rsa_keys.rb`.  
The basic idea is very simple. If we keep the `crc=0` or `crc=1`, then signature is always the same with crc(0 or 1). Now we just need to find something excutable whose crc is 0 or 1.

First we should write a brainfuck program whose output is a legal ruby code, then append some useless character to the code to make its crc be 0 or 1.

A simple brainfuck programmer is like this

    def bf(s):
            p = 0
            q = 0
            K = 48
            ret = &#039;&#039;
            now = 0
            for c in s:
                if ord(c) &lt; K:
                        if (now != 0):
                                ret += &#039;&lt;&#039;
                                now = 0
                        while (p  ord(c)):
                                p -= 1
                                ret += '-'
                else:                                                                                                                                                                             
                        if (now != 1):                                                                                                                                                            
                                ret += '&gt;'                                                                                                                                                        
                                now = 1                                                                                                                                                           
                        while (q  ord(c)):
                                q -= 1
                                ret += '-'
                ret += '.'
        return ret
    

My code is kind of ugly but it does work. (The brainfuck code length is limited to 1000)

Given a crc sum and a byte we can calculate the next crc sum directly.

    inline unsigned int extend(unsigned int r, unsigned char s)
    {
            return (r &lt;&gt; 24) ^ s];
    }
    

`crc_table` is a pre-calculated table.

If the crc sum is 0, then \`r <> 8;  
x |= (i ^ s) << 24;  
if (extend(x, s) == r) {  
//some code  
}  
}  
}

But charset that we can use is limited, so we don't need to worry about too many solutions.

I used a bidirectional search algorithm to speed up my search in my code.  
[crc32.cpp][5]

It works well even I only use `+-` to generate collisions.

    import subprocess
    from time import sleep
    from socket import *
    
    ...
    
    if __name__ == '__main__' :
            s = socket(AF_INET, SOCK_STREAM)
            s.connect(('188.40.147.108', 2000))
            print s.recv(1024)
            #raw = bf("""client.puts File.readlines("/etc/passwd")#""")
            #raw = bf("""client.puts File.readlines("./the_flag.rb")#""")
            raw = bf("""client.puts File.readlines("./rsa_keys.rb")#""")
    
            p = subprocess.Popen("./crc32", stdin = subprocess.PIPE, stdout = subprocess.PIPE)
            p.stdin.write(str(len(raw)) + "\n")
            p.stdin.write(raw + "\n")
            suffix = p.stdout.read()
    
            payload = raw + suffix
    
            print len(payload)
            print payload
    
            s.send(payload + "\n")
            s.send("1\n")
            sleep(5)
            print s.recv(4096)
    

[go.py][6]  
Then I read [./the_flag.rb][7] and [./rsa_keys.rb][8].

The FLAG is `goozbartouuu`

 [1]: http://slipper.tk/writeup/2013/07/11/sigint-2013-crypto-satisfaction/
 [2]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/server.rb
 [3]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/bf.rb
 [4]: http://en.wikipedia.org/wiki/Crc32
 [5]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/crc32.cpp
 [6]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/go.py
 [7]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/the_flag.rb
 [8]: https://github.com/5lipper/CTF-Challenges/blob/master/SIGINT2013/crypto/satisfaction/rsa_keys.rb