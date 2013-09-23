---
title: Codegate2013 web300 writeup
author: hqdvista
layout: post
permalink: /codegate2013-web300-writeup/
categories:
  - CTF
  - writeup
---
This problem first presents a letter, in which the author asks Sherlock to find someone related to hacking case. The letter gives a blog address as clue to find the wanted one.

By analyzing the blog js code we found the js/secret.js rather suspicious. By unpacking and decrypting it we get the following code:

    eval $(document).ready(function(){var cnt=0;$('a.S').click(function(){cnt++;if(cnt==10)   {$('#popup').bPopup({contentContainer:'.content',loadUrl:'./d56b699830e77ba53855679cb1d252da.php'});cnt=0}})});
    

, which discloses a secret login page url: http://58.229.122.17:2218/d56b699830e77ba53855679cb1d252da.php (md5 of &#8220;login&#8221;). It can be accessed by clicking ten times on &#8220;Grey&#8221; logo.

Then we find a post form at http://58.229.122.17:2218/contact.php, we use sqlmap to identify if there is possibility of sqli. Luckily there is.

    python sqlmap.py -u "http://58.229.122.17:2218/contact.php" --data="your_name=1&amp;your_email=1@1.com&amp;question=2&amp;your_message=1&amp;contact_submitted=send"
    

sqlmap identifies time-based blind sqli on parameter &#8220;question&#8221;. We then try to dump possible databases. Time-based blind sqli is slow and sensitive to network traffic load, so remember not to press your network connection when do time-based blind sqli.

Databases &#8220;the\_grey&#8221; is found, with two table &#8211; &#8220;contact&#8221; and &#8220;user&#8221;. &#8220;user&#8221; contains three columns &#8220;no&#8221;, &#8220;id&#8221;, md5 &#8220;password&#8221; and five rows. Using this information we login the blog and in page http://58.229.122.17:2218/my\_page.php?user=victor we get the one who intend to hack Hound Co.,Ltd. and time the company is hacked.

That&#8217;s the key.