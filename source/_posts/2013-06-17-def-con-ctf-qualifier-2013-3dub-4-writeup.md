---
title: DEF CON CTF Qualifier 2013 3dub 4 Writeup
author: DeAdCaT___
layout: post
permalink: /def-con-ctf-qualifier-2013-3dub-4-writeup/
categories:
  - writeup
---
The webpage is like this

    Admin Files
    
    usernames.txt
    passwords.txt
    Login Here
    

You can view the usernames.txt

and the URL is

    http://rememberme.shallweplayaga.me/getfile.php?filename=usernames.txt&amp;accesscode=60635c6862d44e8ac17dc5e144c66539
    

Guessing that the accesscode is one kind of hash of username.txt, after test, we can see

    MD5 ("usernames.txt") = 60635c6862d44e8ac17dc5e144c66539
    

Then we can view other things in this site by building the valid URL

By this way, we can see the getfile.php

    Acces granted to getfile.php!
    
    $value = time();
    $filename = $_GET["filename"];
    $accesscode = $_GET["accesscode"];
    if (md5($filename) == $accesscode){
    echo "Acces granted to $filename!";
    
    srand($value);
    if (in_array($filename, array('getfile.php', 'index.html', 'key.txt', 'login.php', 'passwords.txt', 'usernames.txt'))==TRUE){
    $data = file_get_contents($filename);
    if ($data !== FALSE) {
    if ($filename == "key.txt") {
    $key = rand();
    $cyphertext = mcrypt_encrypt(MCRYPT_RIJNDAEL_128, $key, $data, MCRYPT_MODE_CBC);
    echo base64_encode($cyphertext);
    }
    else{
    echo nl2br($data);
    }
    
    }
    else{
    echo "File does not exist";
    }
    }
    else{
    echo "File does not exist";
    }
    
    }
    else{
    echo "Invalid access code";
    }
    ?&gt;
    

And there is a key.txt can view by this URL

    http://rememberme.shallweplayaga.me/getfile.php?filename=key.txt&amp;accesscode=65c2a527098e1f7747eec58e1925b453
    

the context is

    Acces granted to key.txt!
    
    TEu4LOi+D8CU/+fjK6RUj3CnBuqjfTYA8IgWPNXFEV3R1bDvGLwDA3+1Ew9tdrFqbjonjRUebZBXFL6LdP69wQ==
    

the encrypt process can be seen in the getfile.php

So the way to solve this porblem is refresh the

    http://rememberme.shallweplayaga.me/getfile.php?filename=key.txt&amp;accesscode=65c2a527098e1f7747eec58e1925b453
    

and write a native phpfile having

    echo time();
    

so you will see a time, you should set it as a seed to generate the $key.

At that time, I got the encrypt key is

    TEu4LOi+D8CU/+fjK6RUj3CnBuqjfTYA8IgWPNXFEV3R1bDvGLwDA3+1Ew9tdrFqbjonjRUebZBXFL6LdP69wQ==
    

and the time is 1371387200

We should search +-1000 around 1371387200.

So I write this piece of code

    $i = 1;
    while($i &lt; 2000) 
    {
        $value = 1371387200-1000+$i;
        $data = &quot;TEu4LOi+D8CU/+fjK6RUj3CnBuqjfTYA8IgWPNXFEV3R1bDvGLwDA3+1Ew9tdrFqbjonjRUebZBXFL6LdP69wQ==&quot;;
        $data = base64_decode($data);
        srand($value);
        $key = rand();
        $plaintext = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $key, $data, MCRYPT_MODE_CBC);
        echo $plaintext;
        $i++;
    }
    

run it in your own computer and use Command + f or Ctrl + f to find

    The key is
    

You will see the key~