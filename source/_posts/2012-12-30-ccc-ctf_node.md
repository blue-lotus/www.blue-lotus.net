---
title: 29c3ctf node writeup
author: zTrix
layout: post
permalink: /ccc-ctf_node/
categories:
  - CTF
  - writeup
---
This a web problem, using node.js as the programming language. It only gives a portal showing a login form.

When I started on this problem, there already have been some discoveries made by my team mates(Thanks to Jay and hqdvista). They found this url: http://94.45.252.237:1024/robots.txt , as indicated, we got http://94.45.252.237:1024/pages.js.

After beautifying using http://jsbeautifier.org, I reviewed the code, and it shows that there exists another page called auth.js, but http://94.45.252.237:1024/pages.js is incorrect. Dive into the page.js code again, it&#8217;s filtered out by RE &#8220;/^\/auth&#46;js/&#8221;, so we tried http://94.45.252.237:1024//auth.js . Wow, it works!

After some comprehension work, we figured out that, the check in auth.js will try to authenticate admin using 2 methods, the first is to check username/password, the second is to check useragent/ip. But we have none of them to even give a try.

Later, according to auth.js, we found this url: http://94.45.252.237:1024/stats.txt , which contains a list of login records, but without password. Review the code of auth.js, we can find that if we use the admin useragent and ip, we can get admin access. I wrote a script to try all the ua/ip pair in the stats.txt, resulting the first one is the admin, so that we can access admin page using the command below:

curl -D head.txt -s -A &#8216;Mozilla/5.0 (compatible; MSIE 6.0; Windows NT 5.1)&#8217; -H &#8216;X-Forwarded-For: 8.8.8.8&#8242; http://94.45.252.237:1024/admin

However, we just got the same stats list file as previous one, but the content is a bit different. So I reviewed the code of its generation, in http://94.45.252.237:1024/pages.js, we found that, when rendering admin page, the first local variable passed in is flag, so the flag is in admin page! But we still need to find out which one is.

After searching the internet, we know that express.js use jade html template, so the admin page should be admin.jade. Then we tried http://94.45.252.237:1024/admin.jade , hooray! It reveals that the flag is the string after “Admin &#8211; ”.

Happy cracking!