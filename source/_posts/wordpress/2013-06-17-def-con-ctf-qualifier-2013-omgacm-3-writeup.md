---
title: DEF CON CTF Qualifier 2013 OMGACM 3 Writeup
author: ray
layout: post
permalink: /def-con-ctf-qualifier-2013-omgacm-3-writeup/
categories:
  - Uncategorized
---
DEF CON CTF Qualifier 2013 OMGACM 3 Writeup

## Problem Description

The remote host offers a racing game:

    % nc grandprix.shallweplayaga.me 2038
    Use 'l' and 'r' to move. Don't crash.
    Press return to start
    

We send a line feed to start the game and receive a 5&#215;9 board:

    |-----|
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |  u  |
    |-----|
    

We can send &#8220;l\n&#8221; to move left, &#8220;r\n&#8221; to move right, or &#8220;\n&#8221; to stand.  
At the end of the round, all obstacles fall. It is obvious that we need to avoide those obstacles.

## Solution

One way to view the problem is that we are forced to move forward one step each round.  
We can choose one of the three grids in front us. This is a special case of the dynamic pathfinding problem.

We can apply a breadth-first search to find a safe path from `u` to some cell in the first row.  
During the main loop of the search, we record for each cell how to reach it from the orignal grid.  
As decisions are made once per round, we can record only the first step for each path.

Moreover, we find that there will be a finishing line made up by a row of `=` not far beyond the 1000th round.  
It seems that `=` does not represent an obstacle.

    Round 1062 s
    |-----|
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |=====|
    |u    |
    |-----|
    
    Round 1063 s
    |-----|
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |     |
    |u====|
    |-----|
    
    Round 1064 s
    You won this game. Push a key to play again
    
    Round 1065 r
    User won game
    The key is: all our prix belong to you
    
    Round 1066 r
    

So our criterion to check whether a grid represents an obstacle is: `c != ' ' &amp;&amp; c != 'u' &amp;&amp; c != '='`.

## Programs

Here is the program with BFS:

    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    using namespace std;
    
    #define FOR(i, a, b) for (int i = (a); i &lt; (b); i++)
    #define REP(i, n) FOR(i, 0, n)
    #define SET(i, v) memset(i, v, sizeof i)
    
    const int N = 11, M = 7;
    char a[N][M+1], first[N][M], buf[4096];
    
    bool isObstacle(char c)
    {
      return c != &#039; &#039; &amp;&amp; c != &#039;u&#039; &amp;&amp; c != &#039;=&#039;;
    }
    
    bool cmp(int y, int yy)
    {
      return abs(y-M/2) &lt; abs(yy-M/2);
    }
    
    char work()
    {
      int t = 0, x, y;
      REP(i, N) {
        REP(j, M + 1) {
          a[i][j] = buf[t++];
          if (a[i][j] == &#039;u&#039;)
            y = j;
        }
        a[i][M] = 0;
      }
    
      const char op[] = &quot;lsr&quot;;
      SET(first, 0);
      queue&lt;pair &gt; q;
      q.push(make_pair(9, y));
      while (! q.empty()) {
        x = q.front().first;
        y = q.front().second;
        q.pop();
        if (x == 1) break;
        int ys[] = {y-1, y, y+1};
        sort(ys, ys + 3, cmp);
        REP(i, 3) {
          int yy = ys[i];
          if (! first[x-1][yy] &amp;&amp; ! isObstacle(a[x-1][yy])) {
            first[x-1][yy] = x == 9 ? op[yy-y+1] : first[x][y];
            q.push(make_pair(x-1, yy));
          }
        }
      }
    
      const int cols[M-2] = {3,2,4,1,5};
      REP(i, M-2)
        if (first[1][cols[i]])
          return first[1][cols[i]];
      return 's';
    }
    
    int main()
    {
      int sock_fd;
      struct hostent *host;
      struct sockaddr_in serv_addr = {};
      if ((host = gethostbyname("grandprix.shallweplayaga.me")) == NULL)
        return 1;
      if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
        return 1;
      serv_addr.sin_family = AF_INET;
      serv_addr.sin_port = htons(2038);
      serv_addr.sin_addr = *((struct in_addr *)host-&gt;h_addr);
      if (connect(sock_fd, (struct sockaddr *)&amp;serv_addr, sizeof serv_addr) == -1)
        return 1;
    
      // Press return to start
      read(sock_fd, buf, sizeof buf);
      puts(buf);
      write(sock_fd, "\n", 1);
    
      int round = 0, len;
      char choice[2] = {0, '\n'};
      while (len = read(sock_fd, buf, sizeof buf - 1), buf[len] = '', len &gt; 40) {
        puts(buf);
        choice[0] = work();
        printf("Round %d %c\n", ++round, choice[0]);
        write(sock_fd, choice, sizeof choice);
      }
    }
    

During the contest, we use another approach to solve this task by using a heuristic to find the &#8220;safest&#8221; column: the one with the least obstacles in the inverted triangle behind it.  
And then plan a path to move to the safest column while dodging obstacles nearby.

    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #define MAXDATASIZE 4096
    #include
    #include
    using namespace std;
    
    const int N = 11;
    const int M = 7;
    
    char A[N][M+1];
    char buf[MAXDATASIZE];
    
    int x, y;
    
    char path[N];
    
    int c[M+1], s[M+1];
    
    void cnt(int p, int y, int r)
    {
        if (A[p][y] != ' ') return ;
        if (p == 1) {
            c[r]++;
            return ;
        }
        for (int j = -1; j  0 &amp;&amp; y + j &lt; M-1) {
                cnt(p - 1, y + j, r);
            }
    }
    
    char op[4]=&quot;lsr&quot;;
    
    bool dfs(int p, int y, int r)
    {
        if ((A[p][y] != &#039; &#039;) &amp;&amp; (A[p][y] != &#039;u&#039;) &amp;&amp; (A[p][y] != &#039;=&#039;))
            return 0;
        if (p == 5) {
            return y == r;
        }
        for (int j = -1; j  0 &amp;&amp; y + j c[j];
    }
    
    void work()
    {
        //puts(buf);
        memset(c, 0, sizeof c);
        int t = 0;
        for (int i = 0 ; i &lt; N; i++) {
            for (int j = 0 ; j &lt;= M; j++)
                A[i][j] = buf[t++];
            A[i][M] = 0;
            //puts(A[i]);
        }
    
        x = 9;
        for (int j = 1; j &lt; 6; j++)
            if (A[9][j] == &#039;u&#039;) y = j;
    
        for (int i = 1; i &lt; 6; i++) cnt(5, i, i);
        for (int i = 1; i &lt; 6; i++)  s[i] = i;
        //sort(s + 1, s + 6, cmp);
        int k;
        for (int i = 1; i &lt; 6; i++)
            for (int j = i+1; j  c[s[j]]) {
                    k = s[i]; s[i] = s[j]; s[j] = k;
                }
        for (int i = 1; i h_addr);
        bzero(&amp;(serv_addr.sin_zero),8);
        if (connect(sock_fd, (struct sockaddr *)&amp;serv_addr,sizeof(struct sockaddr))==-1){
            perror("connect error");
            exit(1);
        }
    
        send(sock_fd,"\n",1,0);
        iLength=recv(sock_fd,buf,sizeof(buf),0);buf[iLength] = 0;
        puts(buf);
        iLength=recv(sock_fd,buf,sizeof(buf),0);buf[iLength] = 0;
        puts(buf);
    
        int cnt = 0;
        char r[3] = "";
        r[1]='\n';
        do {
            work();
            r[0] = path[9];
            printf("Round %d %s", ++cnt, r);
            send(sock_fd,r,2,0);
            iLength=recv(sock_fd,buf,sizeof(buf),0);
            buf[iLength]=0;
            puts(buf);
        } while (iLength &gt; 40);
    }