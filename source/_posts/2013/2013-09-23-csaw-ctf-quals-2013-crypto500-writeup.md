---
title: CSAW CTF Quals 2013 crypto500 writeup
author: hellok
layout: post
permalink: /csaw-ctf-quals-2013-crypto500-writeup/
comments: true
categories:
  - CTF
  - writeup
---

## Problem Description
We’ve found the source to the Arstotzka spies rendevous server, we must find out their new vault key. Source code is [here](http://shell-storm.org/repo/CTF/CSAW-2013/Crypto/slurp-500/slurp.py).

So we have to crack sha512 puzzle and calc correct hash. At first i was trying to calculate sEphemeralPriv out, but it seems impossible. after some try, I realized that you can just use "n^m mod === something".

## Code

~~~
#!/usr/bin/python
import os
import sys
import time
import struct
import socket
import telnetlib
import random,hashlib,base64
from hashlib import sha512,sha1

def hashToInt(*params):
    sha=sha512()
    for el in params:
        sha.update("%r"%el)
    return int(sha.hexdigest(), 16)


def gs(num):
    return hashlib.sha1(num).digest()

def crack(sha_p):
    print "crack:",sha_p
    ss=""
    ret=0
    for keylen in range(10,40):
        for add_1 in range (0,140):
            for add_2 in range (10,140):
                for add_3 in range (0,155):
                    for add_4 in range (0,155):
                        for add_5 in range (0,155):
                            new_msg=sha_p+chr(add_1)+chr(add_2)+chr(add_3)+chr(add_4)+chr(add_5)
                        if gs(new_msg)[-3:]=="\xff\xff\xff":
                            print "got one %s"%len(new_msg)
                            if len(new_msg)==21:
                                ss=new_msg
                                print "WE CRACK THE PUZZLE %d\n"%len(ss)
                                ret=1
                                return (ret,ss)

return (ret,ss)

#s = socket.create_connection(('128.238.66.222',7788))
s = socket.create_connection(('127.0.0.1',7788))
se=""
for i in range(1,100):
    ss=s.recv(1000)
    print ss
    sha_p=ss[len(ss)-16:]
    new_msg=crack(sha_p[-16:])
    if new_msg[0]:
        se=new_msg[1]
        print "send:",se,hashlib.sha1(se).digest()[-3:]
        break


s.send(se)
#raw_input()
N = 59244860562241836037412967740090202129129493209028128777608075105340045576269119581606482976574497977007826166502644772626633024570441729436559376092540137133423133727302575949573761665758712151467911366235364142212961105408972344133228752133405884706571372774484384452551216417305613197930780130560424424943100169162129296770713102943256911159434397670875927197768487154862841806112741324720583453986631649607781487954360670817072076325212351448760497872482740542034951413536329940050886314722210271560696533722492893515961159297492975432439922689444585637489674895803176632819896331235057103813705143408943511591629

index=28483644508750028902258833085453121291738558908844640378204850915473006274236033891815596646870094954832384471913373171061099388958373536383247454431214837805096635029244738399662911119306089493137122381562794483801525427601711736403424013966624957471463169984161438593701202381673774894874606950326837142168801614196218030413825361835813060325642766963973454456577907567314093695138863251603368180581162185039224604662844750924047132242103613509637088222461132023037724162558095265336717907895786691004801515830247270579025866384571194244368350362690250445425121639616294300827554006418861422621428030154329451920623#17095359415354811031956139176232822755293333580176796651092816713929142596363160518183074582333639666374490434453616522976528693318889544211598801485465809374370977363040240176466544504447562622451988654983833567639785357067606037932165650485806709538575591881306056180364137612491852807151445333929946180083335184143784040034635507293616742432550220001472750029503246943474764484598756871323795898352813684410952415776064654304191774216122857994311730048413954488815342894838841918071841785821830683677234982197021652871796420412165927837273704381564043458200253387672509280296033910910416083249362331740208987494679

cEphemeral=1
send_num1=str(hex(index))[2:][:-1]
send_num2=str(hex(cEphemeral))[2:]

s.send(struct.pack("H",len(send_num1)))
s.send(send_num1)#send numberc index-->base number

print "1:",s.recv(72)
print "2:",s.recv(60)

s.send(struct.pack("H",len(send_num2)))
s.send(send_num2)#send number cEphemeral

salt=int(s.recv(128),16)
sEphemeral=long(int(s.recv(514),16))
                          
#print "3:salt:",salt
#print "4:sEphemeral:",sEphemeral
tmp=hashToInt(salt,"")
storedKey = pow(index,tmp , N)
cq_sEphemeral=(3 * storedKey) % N
tmp1=(sEphemeral-cq_sEphemeral)%N


#agreedKey_withouthash = (cEphemeral * index^[sha512(salt, password) * slush])^sEphemeralPriv mod Nwhoami
#   cEphemeral=1  -->
#agreedKey_withouthash = (index^[sha512(salt, password) * slush])^sEphemeralPriv mod N
#cause cEphemeral^^3 mod N =1
#so if sha512(salt, password) * slush * sEphemeralPriv mod 3 == 0
#agreedKey_withouthash = 1
#so we don't need to know sEphemeralPriv

slush = hashToInt(cEphemeral, sEphemeral)
salt=hashToInt(index)
agreedKey=hashToInt(1L)
gennedKey=hashToInt(hashToInt(N) ^ hashToInt(index), hashToInt(index), salt,int(cEphemeral), long(sEphemeral), agreedKey)
send_num3=str(hex(gennedKey))[2:][:-1]
s.send(struct.pack("H",len(send_num3)))
s.send(send_num3)


print s.recv(1000)
print s.recv(1000)#-->flag recv
print s.recv(1000)
~~~
{:lang="python"}
