---
title: DEF CON CTF Qualifier 2013 OMGACM 4 Writeup
author: james0zan
layout: post
permalink: /def-con-ctf-qualifier-2013-omgacm-4-writeup/
categories:
  - Uncategorized
---
# Problem Description

Each problem gives you a circuit board, which will have a dimension, a set of nulls (that trace cannot go through), a feed point, and a set of antenna points.

Your goal is to draw a trace for each antenna point that starts at the feed point, ends at the antenna point and does not intersect other traces.

The distance of these traces must also be equal for all antenna points.

# Our Solution

Since we intuitively classify this problem as NP-Complete, we resort to iterative deepening depth-first search for solving it.

In our algorithm, we choose to &#8220;grow&#8221; the traces from antenna points, which means that all the traces simultaneously start from their respective antenna point and go one step further one by one.

This strategy forces that all the traces will be equal length with each other, thus many unnecessary states are avoided.

And if one trace encounters with another trace before reaching the feed point, we can simply merge them into one to eschew intersection.

More specifically, in order to record the current state, we use:

1.  Nine bool variables for each point: one for the point and 8 for adjacent edges; 
2.  One pair < int, int > (one for 1 and the other for sqrt 2) variable for each point to store the distance from this point to the corresponding antenna point if it has been covered by a trace; 
3.  Current &#8220;growing&#8221; point for each trace; 
4.  Which trace’s turn is it to grow in this iteration; 
5.  The final distance, if one of the traces has reached the feed point; 

And we also use a set of pruning tricks to accelerate the algorithm.

1.  Iterative deepening on length of each path; 
2.  Limit the depth of dfs to be at most 30. (This ought to have been another iterative deepening argument, but we hard code it with an empirical number for simplifying the code); 
3.  Preprocess the minimal distance from each point to the feed point and combine this information with iterative deepening threshold for further pruning; 
4.  If one of the trace has already reached the feed point, use the distance for pruning. 

Here is another tip:

The intersections will not only happen at points but also in small squares, you may need to double check this.

# Code

    #include <stdio.h>
    #include <stdlib.h>
    #include <map>
    #include <set>
    #include <vector>
    #include <queue>
    #include <string.h>
    using namespace std;
    
    typedef unsigned long long llu;
    #define MAX 100
    #define mp make_pair
    #define pb push_back
    
    bool Null[MAX][MAX], Null2[MAX][MAX];
    int XY[MAX][MAX];
    int N, M, eN, fX, fY, P, UP;
    int dy[] = {1, 0, -1, 1, -1, 1, 0, -1};
    int dx[] = {1, 1, 1, 0, 0, -1, -1, -1};
    int verse[] = {8, 7, 6, 5, 4, 3, 2 , 1};
    set<vector<bool> > S;
    pair<int, int> dp[] = {
        mp(0, 1),
        mp(1, 0),
        mp(0, 1),
        mp(1, 0),
        mp(1, 0),
        mp(0, 1),
        mp(1, 0),
        mp(0, 1)
    };
    
    struct node {
        vector<bool> t;
        vector<pair<int, int> > len;
        int cnt;
        pair<int, int> ans;
        vector<pair<int, int> > now;
        vector<bool > used;
        int I;
    };
    node Key;
    
    int get(int x, int y) {
        return x*M + y;
    }
    
    pair<int, int> getLen(int j) {
        return mp(Key.len[get(Key.now[Key.I].first, Key.now[Key.I].second)].first+dp[j].first, 
            Key.len[get(Key.now[Key.I].first, Key.now[Key.I].second)].second+dp[j].second);
    }
    
    void output(node ans) {
        printf("Solution %d\n", P);
        int kk=0, L=0;
        for (int i=0; i<N*M; i++) {
            kk++;
            int tx = i / M;
            int ty = i % M;
            for (int j=0; j<4; j++, kk++) {
                if (ans.t[kk])
                    L++;
            }
            kk+=4;
        }
        printf("Line %d\n", L);
        int k=0;
        for (int i=0; i<N*M; i++) {
            k++;
            int tx = i / M;
            int ty = i % M;
            for (int j=0; j<4; j++, k++) {
                if (ans.t[k])
                    printf("Segment %d %d %d %d\n", tx, ty, tx+dx[j], ty+dy[j]);
            }
            k+=4;
        }
    }
    
    bool none() {
        for (int i=0; i<eN; i++)
            if (Key.used[i]) return false;
        return true;
    }
    
    int myabs(int xx) {
        if (xx<0) return -xx;
        return xx;
    }
    
    
    int UPPER_BOUND = 28;
    void dfs(int depth) {
        if (depth > UPPER_BOUND) return;
    
        int x = Key.now[Key.I].first;
        int y = Key.now[Key.I].second;
    
        int i = rand() % 8;
        for (int iii=0; iii<8; iii++, i=(i+1)%8) {
            int newX = x + dx[i];
            int newY = y + dy[i];
    
            if (newX < 0 || newX >= N) continue;
            if (newY < 0 || newY >= M) continue;
            if (Null[newX][newY]) continue;
            if (XY[newX][newY] == -1 || getLen(i).first + getLen(i).second + XY[newX][newY] > UP) continue;
            if (Key.cnt > 0 &&
                                (Key.ans.first < getLen(i).first || Key.ans.second < getLen(i).second) ) continue;
            if (Key.cnt > 0 &&
                                (Key.ans.first == getLen(i).first && Key.ans.second == getLen(i).second) ) continue;
    
    
            if (i == 0) {
                if (Key.t[get(x+1, y) * 9 + 6]) continue;
            }
            if (i == 2) {
                if (Key.t[get(x+1, y) * 9 + 8]) continue;
            }
            if (i == 7) {
                if (Key.t[get(x, y-1) * 9 + 6]) continue;
            }
            if (i == 5) {
                if (Key.t[get(x, y+1) * 9 + 8]) continue;
            }
    
    
            if (newX == fX && newY == fY) {
                if (Key.cnt > 0 && Key.ans != getLen(i)) continue;
                Key.cnt++; Key.ans = getLen(i);
                Key.t[(newX*M + newY)*9 + verse[i]] = true;
                Key.t[get(x, y)*9 + i + 1] = true;
                Key.used[Key.I] = false;
                Key.now[Key.I] = mp(newX, newY);
    
    
                if (Key.cnt > 0 && none()) {
                    output(Key);
                    exit(0);
                }
    
                int II = Key.I, ttt = 0;
                for (Key.I=(Key.I+1)%eN; ttt<eN; ttt++, Key.I=(Key.I+1)%eN)
                    if (Key.used[Key.I])
                        break;
                if (ttt < eN) {
                    dfs(depth+1);
                }
    
                Key.t[(newX*M + newY)*9 + verse[i]] = false;
                Key.t[get(x, y)*9 + i + 1] = false;
                Key.I = II;
                Key.used[Key.I] = true;
                Key.now[Key.I] = mp(x, y);
                Key.cnt--;
            } else if (Key.t[get(newX, newY) * 9]) {
                if (Key.len[newX*M + newY] != getLen(i)) continue;
    
                Key.t[(newX*M + newY)*9 + verse[i]] = true;
                Key.t[get(x, y)*9 + i + 1] = true;
                Key.used[Key.I] = false;
                Key.now[Key.I] = mp(newX, newY);
    
                int II = Key.I, ttt = 0;
                for (Key.I=(Key.I+1)%eN; ttt<eN; ttt++, Key.I=(Key.I+1)%eN)
                    if (Key.used[Key.I])
                        break;
                if (ttt < eN) {
                    dfs(depth+1);
                }
    
                Key.t[(newX*M + newY)*9 + verse[i]] = false;
                Key.t[get(x, y)*9 + i + 1] = false;
                Key.I = II;
                Key.used[Key.I] = true;
                Key.now[Key.I] = mp(x, y);
            } else {
                Key.t[(newX*M + newY)*9] = true;
                Key.t[(newX*M + newY)*9 + verse[i]] = true;
                Key.t[get(x, y)*9 + i + 1] = true;
                Key.len[newX*M + newY] = getLen(i);
                Key.now[Key.I] = mp(newX, newY);
    
                int II = Key.I, ttt = 0;
                for (Key.I=(Key.I+1)%eN; ttt<eN; ttt++, Key.I=(Key.I+1)%eN)
                    if (Key.used[Key.I])
                        break;
                if (ttt < eN) {
                    dfs(depth+1);
                }
    
                Key.t[(newX*M + newY)*9] = false;
                Key.t[(newX*M + newY)*9 + verse[i]] = false;
                Key.t[get(x, y)*9 + i + 1] = false;
                Key.I = II;
                Key.now[Key.I] = mp(x, y);
            }
        }
    }
    
    void dfs2(int x, int y) {
        for (int i=0; i<8; i++) {
            int newX = x + dx[i];
            int newY = y + dy[i];
    
            if (newX < 0 || newX >= N) continue;
            if (newY < 0 || newY >= M) continue;
            if (Null[newX][newY]) continue;
    
            if (XY[newX][newY] != -1 && XY[newX][newY] <= XY[x][y] + 1) continue;
            XY[newX][newY] = XY[x][y] + 1;
            dfs2(newX, newY);
        }
    }
    
    void solve() {
        scanf("%*s%d", &P);
    
        scanf("%*s%*d");
        scanf("%d%d", &N, &M);
    
        scanf("%*s%*d");
        scanf("%d%d", &fX, &fY);
    
    
    
        Key.t.resize(N * M * 9);
        Key.len.resize(N * M);
    
        scanf("%*s%d", &eN);
        for (int i=0; i<eN; i++) {
            int tx, ty;
            scanf("%d%d", &tx, &ty);
    
            Key.now.pb(mp(tx, ty));
            Key.t[get(tx, ty) * 9] = true;
            Key.used.pb(true);
        }
    
        int nN;
        scanf("%*s%d", &nN);
        memset(Null, 0, sizeof(Null));
        memset(Null2, 0, sizeof(Null2));
        for (int i=0; i<nN; i++) {
            int tx, ty;
            scanf("%d%d", &tx, &ty);
            Null[tx][ty] = true;
            Null2[tx][ty] = true;
        }
    
    
    
        memset(XY, -1, sizeof(XY));
        XY[fX][fY] = 0;
        dfs2(fX, fY);
    
        node base = Key;
        for (UP=6; ; UP++) {
            dfs(0);
            Key = base;
        }
    }
    
    int main() {
        solve();
    }
    

# Result

You can check out <http://ascii.io/a/3644> for the result.

**CAVEAT:** We use many heuristics in the program, so it will not guarantee success for every run.