---
title: UFOCTF2013 Web100 FlagStore
author: lovelydream
layout: post
permalink: /ufoctf2013-web100-flagstore/
categories:
  - Uncategorized
---
It is a very interesting problem and exhausted our much time and energy.

There are 3 levels.All about sqli.

The first level only needs username and password.After trying for several hours with some common sqli like &#8216; or &#8217;1&#8242;=&#8217;1 and so on but all failed,I accidentally found that &#8221; or &#8220;1&#8243;=&#8221;1 made sense.And @fqj use sqlmap to find the true username and level1_password in the database.(sqlmap also told that the database is sqlite).

The second level needs username,level1&#95;password and level2&#95;password.*Note that you must back to level 1 and use true username and password to get to level2 again or you&#8217;ll never go ahead anymore*.And seeking for more hours,I find that &#8220;%&#8221; in sqlite can match strings(including empty strings) and &#8220;&#95;&#8221; can match single char.In this level,&#8221;%&#8221; is just the payload.And we use &#8220;&#95;&#8221; to brutefoce the true password.

The third level needs level3&#95;password and a confirm for level3&#95;password.And for payload &#8216; or password like &#8216;%&#8217; &#8212; the site told us that password3.1 passed but password3.2 failed.In confirm blank a mistake happened but we know that the sqli took effect.That is enough,by using &#8220;_&#8221; we can bruteforce the true password(if the site told us only password 3.2 failed then it makes sense).

@H.Shao wrote the script to find the password for level2 and level3 and finally got the FLAG.There is trap in level3,because there is &#8220;&#95;&#8221; in the level3&#95;password and this time &#8220;&#95;&#8221; doesnt represent any single char but just &#8220;_&#8221;.