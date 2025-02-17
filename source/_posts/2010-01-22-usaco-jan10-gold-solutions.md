---
layout: post
title: USACO JAN10 Gold
author: MaskRay
tags: [algorithm, oi]
---

## hayturn

動態規劃， $F_m$表示當前要做選擇的奶牛在可以選擇$w_{m\ldots n-1}$時可以獲得的最大值。
$S_m$表示當前要做選擇的奶牛做完$w_{m\ldots n-1}$的最優決策後，下一個奶牛可以取得的最大值。

```cpp
#include <stdio.h>

#ifdef WIN32
#define LLD "%I64d"
#else
#define LLD "%lld"
#endif

int w[700000];

int main()
{
    int n;
    scanf("%d", &n);
    for (int i = 0; i < n; ++i)
        scanf("%d", &w[i]);
    long long a = 0, b = 0;
    for (int i = n; --i >= 0; )
        if (b+w[i] >= a)
        {
            long long t = a;
            a = b+w[i];
            b = t;
        }
    printf(LLD" "LLD"\n", a, b);
}
```

## island

根據USACL Analysis（後附），根據一個格子周圍格子的佈局，先把一些點轉換爲A，然後繞着A走一圈。

```cpp
#include <cstdio>
using namespace std;

const int H = 1000, W = 1000;
const int dr[] = {0,1,0,-1}, dc[] = {1,0,-1,0};
char a[H][W+1];
int tmp[4], h, w;

void DFS(int r, int c)
{
    if (a[r][c] != '.' || !r || r == h-1 || !c || c == w-1) return;
    int cntA = 0;
    for (int d = 0; d < 4; ++d)
        if (a[r+dr[d]][c+dc[d]] == 'A')
            tmp[cntA++] = d;
    switch (cntA)
    {
        case 2:
            if ((tmp[0]+tmp[1])%2 == 0 || a[r-dr[tmp[0]]-dr[tmp[1]]][c-dc[tmp[0]]-dc[tmp[1]]] != '.')
                break;
            //fall through
        case 3:
        case 4:
            a[r][c] = 'A';
            for (int d = 0; d < 4; ++d)
                DFS(r+dr[d], c+dc[d]);
    }
}

inline bool check(int r, int c)
{
    return 0 <= r && r < h && 0 <= c && c < w && a[r][c] == '.';
}

int main()
{
    scanf("%d%d", &h, &w);
    for (int r = 0; r < h; ++r)
        scanf("%s", a[r]);
    for (int r = 1; r < h-1; ++r)
        for (int c = 1; c < w-1; ++c)
            DFS(r, c);

    int r, c, rr, cc, d = 0, len = 0;
    for (r = 0; r < h; ++r)
        for (c = 0; c < w; ++c)
            if (a[r][c] == 'A')
            {
                --r;
                goto L1;
            }
L1:
    rr = r;
    cc = c;
    do
    {
        int rrr = r+dr[d+1&3], ccc = c+dc[d+1&3];
        if (check(rrr, ccc))
        {
            r = rrr;
            c = ccc;
            d = d+1 & 3;
        }
        else if (rrr = r+dr[d], ccc = c+dc[d], check(rrr, ccc))
        {
            r = rrr;
            c = ccc;
        }
        else if (rrr = r+dr[d+3&3], ccc = c+dc[d+3&3], check(rrr, ccc))
        {
            r = rrr;
            c = ccc;
            d = d+3 & 3;
        }
        ++len;
    }while (r != rr || c != cc);
    printf("%d\n", len);
}
```

## telephone

先把題目中給出的樹有根化，對於一個頂點u，如果它有不超過K/2個孩子還未被分配，
可以把它們中最多2*K個在u處連接起來。如果有孤立孩子並且還未配對完K對孩子，
那麼只能和u子樹外的頂點配對，這相當於u是其parent的未分配頂點。

```cpp
#include <cstdio>
#include <vector>
using namespace std;

const int N = 100000;
int k, res = 0;
vector<int> e[N];

int compute(int u, int p)
{
    int t = e[u].size() == 1;
    for (vector<int>::iterator it = e[u].begin(); it != e[u].end(); ++it)
        if (*it != p)
            t += compute(*it, u);
    if (t <= k*2)
        return res += t/2, t%2;
    res += k;
    return 0;
}

int main()
{
    int n;
    scanf("%d%d", &n, &k);
    for (int i = n; --i; )
    {
        int u, v;
        scanf("%d%d", &u, &v);
        --u; --v;
        e[u].push_back(v);
        e[v].push_back(u);
    }
    compute(0, -1);
    printf("%d\n", res);
}
```

```
USACO JAN10 Problem 'island' Analysis

by John Pardon

This problem requires nothing more than a modified flood-fill algorithm. Our strategy is too add squares to the main island until the optimal path is just to follow the boundary of the main island. Note that this would not work without the assumption that the main island is connected.

Consider the following four configurations:

?.?    ?..    ?.x    ?.A
A.A    A..    A..    A..
?A?    ?A?    ?A?    ?A?
(I)   (II)   (III)  (IV)
Let's focus on the center square of each of these configurations.

In (I), clearly there is no reason why FJ's ship would ever want to visit the center square; he would just have to leave again. Thus the problem remains unchanged if we change the center square to 'A'.

In (II), the situation is similar. A path like:

 V
?|.
A+->
?A?
can be changed to:

 V
?++
A.+>
?A?
with no increase in length. Thus we may change the center to square to 'A'.

In (III), we may not change the center square to an 'A', because then it wouldn't be possible to go around the main island without also going around the 'x' in the upper-right corner.

In (IV), we may also not change the center square to an 'A', at least not unless the top center square or the right center square is changed to an 'A' first.



After making the modifications (I) and (II) as many times as possible, the optimal solution is to follow the boundary of the main island, and this can just be simulated in order to find its length. Doing (I) and (II) as many times as possible is a modified flood-fill and can be done with a DFS. A solution follows:
```
