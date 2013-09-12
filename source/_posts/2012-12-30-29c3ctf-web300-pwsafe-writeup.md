---
title: 29c3ctf web300 pwsafe writeup
author: hqdvista
layout: post
permalink: /29c3ctf-web300-pwsafe-writeup/
categories:
  - CTF
  - writeup
---
This is a rather straight-forward web problem with a little cryptography trick and it&#8217;s solved under the joint effort of fqj and me. The initial page consists of login form and register form. After registration user can login in to a pastebin like page in which he or she can edit and save text. Those are nearly all the functionalities.

At first we tried sqli on form, but doesn&#8217;t work. Further investigation on potential xss also doesn&#8217;t work. Then we turned to focus on http headers. It seems there are some patterns in &#8220;session&#8221; item of cookie. After comparing several accounts&#8217; cookie with different names and from different computers, fqj found the cookie is actually

    954a33ddafa959cf59247cd21b4cc163 + md5(username) + md5(ip)
    

wow, very useful discovery. The &#8220;ip&#8221; can be forged by X-Forwarded-For header.

Then, how to use this important pattern? Inferred from the statement of this problem, administrator account may be needed. After fuzzing and crawling we found juicy uris, /admin and /server-status. Based on the return of /admin we figure out that the administrator account username is &#8220;admin&#8221;, and accessing /server-status shows connections to /admin, which contains ip &#8220;1.2.3.4&#8243;. Why not give it a try?

So construct session cookie with username &#8220;admin&#8221; and ip &#8220;1.2.3.4&#8243;, access login page, and key is there <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_smile.gif' alt=':)' class='wp-smiley' />