---
title: DEF CON CTF Qualifier 2013 3dub 5 Writeup
author: lovelydream
layout: post
permalink: /def-con-ctf-qualifier-2013-3dub-5-writeup/
categories:
  - Uncategorized
---
This task was solved by *aay* soon after the the problem came up.ym.

The initial intention of the organizer is that the vulnerability is in session and the cookie is decrypt byte per byte so we can post some username like &#8216;admiX&#8217; and brute force the ciphertext of the last letter to get the admin&#8217;s session.However there&#8217;s another quick way to achieve the goal.

As we visited the page, corresponding to the experience of solving the 3dub problem before, we tried to login as admin.But the ruby program had a runtime error.We thought that we should decode and construct the cookie.

From the backtrace, we can get some code

    get '/' do
      haml :index
    end
    
    post '/' do
      if params[:username] == 'admin'
        raise "can't log in as admin, be smarter"
    end
    
    cipher = make_cipher
    cipher.encrypt
    cipher.key = CRYPTO_KEY
    iv = cipher.random_iv
    

That says the POST variable username is checked and cannot be admin.The key point is that we can use **username[]** to bypass since Array is a totally different thing from String.So just modify the form&#8217;s field from *username* to *username[]* and it takes effect.The problem is solved so far.

However, the fact that really confuse me is that since username[] is an Array, why the program manages to run correctly by username[] which replaces username.

I happened to find some clue later.Just changed the *username* field to *username[]* and added a new POST field *username[][]* and then submitted it.The program had another error: **ArgumentError at / odd number of arguments for Hash**, and the backtrace with some new code.

    end
    
    cipher = make_cipher
    cipher.encrypt
    cipher.key = CRYPTO_KEY
    iv = cipher.random_iv
    
    plaintext = Hash[*params.sort.flatten].to_param
    
    ciphertext = cipher.update plaintext
    ciphertext &lt;&lt; cipher.final
    
    cookies[:iv] = iv
    cookies[:ciphertext] = ciphertext
    cookies[:verification] = params[:verification]
    

Now we can have a clear view on how the cookies are generated.And pay attention to the following code:

    plaintext = Hash[*params.sort.flatten].to_param
    

The params list is sorted and then flattened which means that the inner square brackets [] are token out so that ['admin'] are changed into &#8216;admin&#8217;.