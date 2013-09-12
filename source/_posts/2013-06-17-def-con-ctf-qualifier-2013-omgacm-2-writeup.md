---
title: DEF CON CTF Qualifier 2013 OMGACM 2 Writeup
author: ray
layout: post
permalink: /def-con-ctf-qualifier-2013-omgacm-2-writeup/
categories:
  - Uncategorized
---
DEF CON CTF Qualifier 2013 OMGACM 2 Writeup

Text adventures are one of the oldest types of computer games and form a subset of the adventure genre.  
After walking around for a while, we find that we need to go all the way north and reach a room with two jugs.

`look red jug`, `look blue jug` and `look inscription` gives the detail of a water-pouring problem with two jugs and  
we are required to give a solution in limited time and trials. One insight is that once we decide to pour water from the red jug to the blue jug,  
there will be no sense to pour water in the opposite direction (from the blue jug to the red jug) in later actions.  
What we can do is emptying the blue jug, filling the red jug and pouring water from the red jug to the blue jug.  
We can thus simulate the two cases (from red to blue and from blue to red) separately and take the case with the smaller actions.

NB. we need to send a bunch of actions at once to reduce time spent on network transmission.  
It seems that we can only send two packets at a mission. Here is the program:

    require 'socket'
    require 'set'
    
    $sock = TCPSocket.new 'diehard.shallweplayaga.me', 4001
    
    def send d
      $sock.puts d
      msg = $sock.recv(1024)
      print msg
      msg
    end
    
    def recv
      msg = $sock.recv 1024
      print msg
      msg
    end
    
    def pour blue, red, ins
      r = b = c = 0
      while b != ins &amp;&amp; r != ins
        c += 1
        if b == blue
          b = 0
        elsif r &gt; 0
          bb = [r+b, blue].min
          r -= bb-b
          b = bb
        else
          r = red
        end
      end
      c
    end
    
    #res = send ''
    #while res[/Exits: (.*)/]
      #exits = $1.split.select {|c| c != 's' }
      #res = send(exits.member?('n') ? 'n' : exits[0])
    #end
    send 'nnnnnnenwn'.chars.map {|c| "#{c}\n" }.join.chomp
    
    loop do
      exchange = false
    
      res = send "get red jug\nget blue jug\nlook red jug\nlook blue jug\nlook inscription"
      res += recv until res[/A red jug holds 0 of (\d+) gallons/]
      red = $1.to_i
      res += recv until res[/A blue jug holds 0 of (\d+) gallons/]
      blue = $1.to_i
      res += recv until res[/To get to the next stage put (\d+) gallons/]
      ins = $1.to_i
    
      sred, sblue = 'red', 'blue'
      res1 = pour blue, red, ins
      res2 = pour red, blue, ins
      if res1 &gt; res2
        exchange = true
        red, blue = blue, red
        sred, sblue = sblue, sred
      end
    
      r = b = 0
      msg = ''
      while b != ins &amp;&amp; r != ins
        if b == blue
          b = 0
          msg += "empty #{sblue} jug\n"
        elsif r &gt; 0
          bb = [r+b, blue].min
          r -= bb-b
          b = bb
          msg += "pour #{sred} jug into #{sblue} jug\n"
        else
          r = red
          msg += "fill #{sred} jug\n"
        end
      end
      msg += "put #{b == ins ? sblue : sred} jug onto scale\n"
      msg += "drop #{b == ins ? sred : sblue} jug\n"
      msg += "n"
    
      res = send msg
      res += recv until res[/You (see|find)/]
      if res['You find yourself in a solid granite']
        send "look key\nget key"
        break
      end
    end
    

Check out http://ascii.io/a/3641 to understand the entire flow of the program.

And the very program we used in the contest:

    #!/usr/bin/env python
    
    import os, sys, socket, random, subprocess
    from subprocess import Popen, PIPE
    
    import os, sys
    
    def solve(a, b, c):
        x = y = 0
        n = 2
        flag = True
        while x == 0 and y == 0:
            for i in range(1, n):
                t = i * a - (n - i) * b
                if t == c:
                    x = i
                    y = n - i
                    break
                elif t == -c:
                    flag = False
                    x = n - i
                    y = i
                    a, b = b, a
                    break
            n += 1
        va = vb = 0
        ret = ''
        A = B = ''
        if flag:
            A = 'red jug'
            B = 'blue jug'
        else:
            A = 'blue jug'
            B = 'red jug'
        #print '%d x %d - %d x %d = %d, %d -&gt; %s, %d -&gt; %s' % (x, a, y, b, c, a, A, b, B)
    
        while x or y:
            if va == 0:
                ret += 'fill %s\n' % A
                x -= 1
                va = a
                #print va, vb
            elif vb == b:
                ret += 'empty %s\n' % B
                y -= 1
                vb = 0
                #print va, vb
            else:
                t = b - vb
                if t &gt; va:
                    t = va
                va -= t
                vb += t
                ret += 'pour %s into %s\n' % (A, B)
                #print va, vb
    
        if va != c and vb != c:
            vb += va
            #print va, vb
            ret += 'pour %s into %s\n' % (A, B) 
    
        if va == c:
            ret += 'put %s onto scale\n' % A
            ret += 'drop %s\nn' % B
        elif vb == c:
            ret += 'put %s onto scale\n' % B
            ret += 'drop %s\nn' % A
        else:
            print 'Error'
    
        return ret
    
    
    
    s = socket.socket()
    s.connect(('diehard.shallweplayaga.me', 4001))
    
    r = ['n', 'n', 's', 's', 'n', 'n', 's', 's', 'w', 'e', 'w', 'w', 's', 'n', 'e', 'e', 'w', 'w', 's', 'n', 'n', 'w', 'e', 'n', 'e', 'w', 'e', 'e', 'w', 'n', 'n', 'e', 'w', 's', 's', 'e', 'e', 'n', 's', 'w', 'e', 'w', 'w', 's', 'n', 'n', 'n', 's', 's', 'e', 'w', 'e', 'e', 'n', 'n', 'n', 'n', 's', 's', 'n', 'n', 's', 'n', 'n', 'n', 'e', 'w', 'e', 'n', 's', 'w', 's', 's', 's', 'n', 's', 's', 'n', 's', 'n', 's', 's', 'n', 'n', 's', 's', 's', 'n', 'n', 'n', 'n', 's', 'n', 'n', 's', 's', 'n', 'n', 's', 's', 's', 's', 'n', 'n', 'n', 's', 's', 's', 's', 'n', 's', 'w', 'w', 'w', 's', 's', 'e', 'w', 'n', 'n', 'e', 'n', 'e', 'e', 'n', 's', 'w', 'e', 'w', 'w', 's', 'n', 'n', 's', 'w', 'e', 'n', 'n', 'e', 'w', 's', 'w', 'e', 'n', 'e', 'e', 'e', 'w', 'w', 'e', 'w', 'w', 's', 'e', 'w', 'e', 'w', 'e', 'w', 'e', 'w', 's', 'n', 'e', 'w', 'e', 'e', 'n', 's', 'w', 'w', 'w', 's', 's', 'e', 'w', 'n', 's', 'e', 'e', 'n', 's', 'w', 'w', 'n', 'n', 's', 'w', 'e', 'n', 's', 'w', 'e', 'w', 'e', 'n', 's', 's', 'e', 'w', 'n', 'w', 'e', 'n', 'e', 'n', 'e', 'w', 'e', 'e', 'n', 'n', 's', 's', 'w', 'e', 'w', 'e', 'w', 'e', 'w', 'e', 'n', 'n', 'n', 's', 's', 'n', 's', 's', 'w', 'w', 'w', 'e', 'w', 'e', 's', 'n', 'e', 'e', 'w', 'e', 'n', 'n', 'n', 'n', 'n', 's', 's', 'n', 's', 'n', 's', 'n', 's', 'n', 's', 'n', 's', 's', 's', 'n', 'n', 'n', 's', 's', 'n', 'n', 's', 's', 'n', 's', 'n', 'n', 's', 's', 's', 'n', 's', 'n', 'n', 's', 's', 's', 'n', 's', 'n', 's', 'n', 's', 'n', 's', 'w', 'w', 's', 'n', 'n', 'n', 's', 's', 's', 'n', 's', 'n', 's', 'n', 'n', 'w', 'e', 'w', 'e', 'w', 'e', 's', 'e', 'e', 'w', 'e', 'w', 'e', 'n', 's', 'w', 'w', 'n', 'w', 'e', 'n', 'e', 's', 'n', 'e', 'e', 'w', 'w', 'e', 'w', 'e', 'w', 'n', 'n', 's', 'n', 'e', 'w', 's', 'w', 'e', 's', 'w', 's', 'w', 'e', 's', 'w', 's', 'e', 'w', 'e', 'w', 'w', 'e', 'e', 'e', 'n', 'n', 'n', 'n', 'n', 'n', 'e', 'n', 'w', 'e', 's', 'w', 's', 'n', 'e', 'n', 'w', 'e', 's', 'w', 'e', 'w', 'e', 'n', 's', 'n', 'w', 'n']
    
    r = 'nnnnnnenwn'
    
    print s.recv(1000)
    
    def send(content):
        if not content: 
            return
        print 'send ...' + content
        global s
        s.send(content + '\n')
        data = s.recv(4096)
        data = data.strip()
        if data:
            print data
        return data
    
    def interactive():
        while True:
            cmd = raw_input('cmd: \n')
            if cmd == 'quit':
                return
            send(cmd)
    
    for i in r:
        send(i)
    
    send('get red jug')
    send('get blue jug')
    send('fill blue jug')
    send('pour blue jug into red jug')
    send('empty red jug')
    send('pour blue jug into red jug')
    send('fill blue jug')
    send('pour blue jug into red jug')
    send('put blue jug onto scale')
    #send('drop blue jug')
    send('drop red jug')
    send('n')
    send('look')
    
    #send(raw_input('cmd?\n'))
    
    fill_red = 'A red jug holds 0 of '
    fill_blue = 'A blue jug holds 0 of '
    target_str = 'To get to the next stage put '
    
    while True:
        st = -1
        ed = -1
        d = send('look red jug\nlook blue jug\nlook inscription')
        while st == -1 or ed == -1:
            st = d.find(fill_red)
            ed = d.find(' ', st + len(fill_red))
            if st == -1 or ed == -1:
                dd = s.recv(4096)
                dd = dd.strip()
                if dd:
                    print dd
                d += dd
                continue
            red = int(d[st + len(fill_red): ed])
    
        st = -1
        ed = -1
        while st == -1 or ed == -1:
            st = d.find(fill_blue)
            ed = d.find(' ', st + len(fill_blue))
            if st == -1 or ed == -1:
                dd = s.recv(4096)
                dd = dd.strip()
                if dd:
                    print dd
                d += dd
                continue
            blue = int(d[st + len(fill_blue): ed])
    
        st = -1
        ed = -1
        while st == -1 or ed == -1:
            st = d.find(target_str)
            ed = d.find(' ', st + len(target_str))
            if st == -1 or ed == -1:
                dd = s.recv(4096)
                dd = dd.strip()
                if dd:
                    print dd
                d += dd
                continue
            target = int(d[st + len(target_str):ed])
    
        print 'solving ... %d %d %d' % (red, blue, target)
        #p1 = Popen(["./solve.out"], stdin=PIPE, stdout=PIPE)
        #p1.stdin.write('%d %d %d' % (red, blue, target))
        #p1.stdin.close()
        #result = p1.stdout.read() 
        result = solve(red, blue, target)
    
        d = send('get red jug\nget blue jug\n' + result.strip())
    
        while d.find('The scale balances perfectly and a door opens to the next room!') == -1:
            dd = s.recv(4096)
            if dd:
                print dd.strip()
            d += dd
        if d.find('You find yourself in a solid granite chamber filled with hexadecimal') &gt; -1:
            dd = send('look key\nget key')
            interactive()
        #interactive()