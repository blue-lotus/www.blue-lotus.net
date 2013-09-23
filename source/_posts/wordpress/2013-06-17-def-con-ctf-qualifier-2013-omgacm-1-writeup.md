---
title: DEF CON CTF Qualifier 2013 OMGACM 1 Writeup
author: ray
layout: post
permalink: /def-con-ctf-qualifier-2013-omgacm-1-writeup/
categories:
  - Uncategorized
---
We are given several 8-puzzles and are required to give a sequence of actions that leads

from the initial state to the goal state.

Each state of an 8-puzzle can be represented as a permutation of 9 such as 123456780,

which can be expressed as a Lehmor code in the factorial number system.

Given that there is a bijection from 0 ~ n!-1 to a number coded in the factorial number system,

we can encode the state of the 8-puzzle compactly.

Thus we translate the puzzle into a pathfinding problem (from the initial state to the goal state).

Many algorithms apply and we just take one of the simplest: breadth-first search.

On a separate note, we need to send the whole sequence of actions at once to reduce time spent on network transmission.

Here is the heart of the program (in C++ 11):

    #include 
    #include 
    #include 
    #include 
    using namespace std;
    
    const int factorial[] = {1,1,2,6,24,120,720,5040,40320};
    
    #define FOR(i, a, b) for (int i = (a); i &lt; (b); i++)
    #define REP(i, n) FOR(i, 0, n)
    
    enum Direction {U, R, D, L};
    
    struct Node
    {
      int a[9], g;
      int zero() const {
        return int(find(a, a + 9, 0) - a);
      }
      Node up() {
        int p = zero();
        if (p = 6) throw 0;
        Node res = *this;
        swap(res.a[p], res.a[p+3]);
        res.g = g + 1;
        return res;
      }
      Node left() {
        int p = zero();
        if (p % 3 == 0) throw 0;
        Node res = *this;
        swap(res.a[p], res.a[p-1]);
        res.g = g + 1;
        return res;
      }
      bool operator==(const Node &amp;rhs) const {
        return equal(a, a + 9, rhs.a);
      }
    };
    
    int closed[362880];
    
    const Node dst = {{1,2,3,4,5,6,7,8,0}};
    int lehmorCode(const Node &amp;x)
    {
      int h = 0;
      REP(i, 9)
        REP(j, i)
          if (x.a[j] &gt; x.a[i])
            h += factorial[8-j];
      return h;
    }
    
    int bfs(const Node &amp;src)
    {
      queue q;
      fill_n(closed, 362880, -1);
      int dst_code = lehmorCode(dst);
    
      Node orig = dst;
      q.push(orig);
      closed[dst_code] = -2;
    
      while (! q.empty()) {
        Node curr = q.front();
        q.pop();
        if (curr == src) return curr.g;
        int dir = 0;
        function fs[] = {&amp;Node::up, &amp;Node::right, &amp;Node::down, &amp;Node::left};
        for (auto f : fs) {
          dir++;
          try {
            Node succ = f(curr);
            int h = lehmorCode(succ);
            if (closed[h] == -1) {
              q.push(succ);
              closed[h] = dir - 1;
            }
          } catch (int) {
          }
        }
      }
      return -1;
    }
    
    void print(Node x)
    {
      for(;;) {
        switch (closed[lehmorCode(x)]) {
        case U:
          x = x.down();
          puts("up");
          break;
        case R:
          x = x.left();
          puts("right");
          break;
        case D:
          x = x.up();
          puts("down");
          break;
        case L:
          x = x.right();
          puts("left");
          break;
        case -2:
          return;
        }
      }
    }
    
    int main()
    {
      Node src;
      REP(i, 9)
        scanf("%1d", &amp;src.a[i]);
      int len = bfs(src);
      printf("%d\n", len);
      if (len &gt; 0)
        print(src);
    }
    

And the communication module written in Ruby:

    require 'socket'
    
    s = TCPSocket.new 'pieceofeight.shallweplayaga.me', 8273
    
    src = ''
    rest = 0
    
    while line = s.gets
      puts line
      if line =~ /|  \d/
        xs = line.scan(/\|  ./).map {|x| x[-1] }
        if xs.find {|x| x =~ /\d/ }
          src &lt; 0
          rest -= 1
        else
          IO.popen './acm1', 'r+' do |h|
            h.puts src
            h.close_write
            xs = h.read.lines.to_a
            s.puts xs[1..-1].join
            rest = xs[0].to_i
          end
        end
        src = ''
      end
      if line =~ /The key is/
        File.open('flag') {|hh| hh.puts line }
      end
    end