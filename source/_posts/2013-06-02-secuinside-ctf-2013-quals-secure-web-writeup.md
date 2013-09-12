---
title: secuinside CTF 2013 quals secure web writeup
author: hqdvista
layout: post
permalink: /secuinside-ctf-2013-quals-secure-web-writeup/
categories:
  - writeup
---
This problem consists a so plugin and web application. H.Shao analyzes the so plugin, and found out it was acting as a WAF, intended to filter out dangerous actions

      if ( strstr(haystack, "php") )//no php
      {
        ap_log_cerror("mod_dontwebhack300.c", 130, 3, 0, v13, "be my guest. lol xD");
        *v15 = 1;
      }
      haystack = strstr(haystack, "filename");//must have filename
      if ( !haystack )
        return 0;
      v9 = get_filename(haystack);
      ap_log_cerror("mod_dontwebhack300.c", 138, 3, 0, v13, v9);
      if ( *v15 == 1 || check_bad_php_string(v13, src) == 1 )// forbid dangerous functions
        return 120000;
    

Disallowed functions:

    v11 = "passthru";
    v12 = "fpassthru";
    v13 = "system";
    v14 = "`";
    v15 = "execl";
    v16 = "open";
    v17 = "popen";
    v18 = "escapeshellcmd";
    v19 = "eval";
    v20 = "proc_open";
    v21 = "get_contents";
    

However, some mistakes occurred in the server configuration, causing the WAF working abnormally, thus exposing the application to the outside, which means one can upload any php webshell with full functionalities to the server . Given the source code of upload page:

    $uploaddir = '/var/www/uploads/' . md5($_SERVER["REMOTE_ADDR"]) . '/';
    
    if(is_dir($uploaddir) == false)
        mkdir($uploaddir);
    $uploadfile = $uploaddir . basename($_FILES['data']['name']);
    
    if (move_uploaded_file($_FILES['data']['tmp_name'], $uploadfile)) {
        echo "Success\n";
    } else {
        print "failed\n";
    }
    

We could figure out the uploaded directory is md5(client_ip). Then we uploaded a webshell and connect back a shell. We found the server had gcc installed with linux version 3.8.0.

We first thought we might need root, so we tried several local root exploit but failed, also we didn&#8217;t found many other teams&#8217; exploit, so this may not be a right way.

At this time the wandering aay found the home directory, and the flag file was just there&#8230; problem solved this way.