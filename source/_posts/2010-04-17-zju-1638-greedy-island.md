---
layout: post
title: ZJU 1638. Greedy Island
author: MaskRay
tags: [algorithm, oi, oj]
---

```
Greedy Island
Time Limit: 5 Seconds      Memory Limit: 32768 KB
Gon and Killua are engaged in a game of Greedy Island. The goal of the game is to collect 100 spell cards distributed all over the game. A spell card has three properties: Attack, Defense and Special. The numeric value of each property is between 0 and 100. Each card can be used only ONCE. All the spell cards must be stored in the Book - the initial item of the game. The Book can store at most 50 spell cards, so Gon and Killua can have at most 100 spell cards in all. Now Gon and Killua have n spell cards, and they want to use A cards for Attack, B cards for Defense and C cards for Special uses (A + B + C <= 100). If n > A + B + C, they have to discard some cards.

They would like to know the maximum sum of the Attack value in Attack Group, Defense value in Defense Group and Special value in Special Group. If there are multiple solutions, choose the solution with the maximum sum of ALL the properties of all the cards.

Input

The first line contains an integer T (1 <= T <= 10), the number of test cases.

For each test case, the first line contains a single integer n (n <= 100,100); the next line contains three integers A, B and C (A, B, C >= 0, A + B + C <= n); the following n lines contain the Attack value, Defense value and Special value of the n spell cards.


Output

For each test case, print the maximum sum of Attack value in Attack Group, Defense value in Defense Group and Special value in Special Group, and maximum sum of ALL the properties of all the cards in one line, separated with one space.


Sample Input

2
3
1 1 1
100 0 0
0 100 0
0 0 100
4
1 1 1
12 32 44
33 48 37
37 38 33
46 79 78

Sample Output

300 300
163 429
 ```

有 n (n <= 100100) 張卡片，卡片 i 有 a_i b_i c_i 三個屬性，選則不超過 A 張用於提升屬性一，不超過 B 張提升屬性二，不超過 C 張提升屬性三，每張卡片只能用於提升一種屬性。求三種屬性提升值之和的最大值，若有多解，最大化 sigma(S_i)，S_i = A_i + B_i + C_i。

容易構建最大費用最大流模型，但頂點數很多，需要優化。
以 (A_i, S_i) 爲關鍵字，保留最大的 A+B+C 張卡片
以 (B_i, S_i) 爲關鍵字，保留最大的 A+B+C 張卡片
以 (C_i, S_i) 爲關鍵字，保留最大的 A+B+C 張卡片

```cpp
#include <cstdio>
#include <cstring>
#include <climits>
#include <algorithm>
using namespace std;

const int N = 100100;
struct Card { int a, b, c, s; bool valid; } cards[N];
bool cmp_by_a(const Card &x, const Card &y) {return x.a > y.a || x.a == y.a && x.s > y.s; }
bool cmp_by_b(const Card &x, const Card &y) {return x.b > y.b || x.b == y.b && x.s > y.s; }
bool cmp_by_c(const Card &x, const Card &y) {return x.c > y.c || x.c == y.c && x.s > y.s; }

const int V = 100*3+4;
bool flag[V];
int h[V], mincost;
struct Edge {int v, c, w; Edge *next, *pair; } *e[V], pool[V*10], *pit = pool;
void insert(int u, int v, int c, int w)
{
    *pit = (Edge){v, c, w, e[u]}; e[u] = pit++;
    *pit = (Edge){u, 0, -w, e[v]}; e[v] = pit++;
    e[u]->pair = e[v];
    e[v]->pair = e[u];
}
bool relabel(int sink)
{
    int d = INT_MAX;
    for (int u = 0; u <= sink; u++) if (flag[u])
        for (Edge *it = e[u]; it; it = it->next)
            if (it->c > 0 && !flag[it->v])
                d = min(d, h[it->v]+it->w-h[u]);
    if (d == INT_MAX) return false;
    for (int u = 0; u <= sink; u++)
        if (flag[u]) h[u] += d;
    return true;
}
int augment(int u, int d, int src, int sink)
{
    if (u == sink)
        return mincost += (h[src]-h[sink])*d, d;
    flag[u] = true;
    int old = d;
    for (Edge *it = e[u]; it; it = it->next)
        if (it->c > 0 && !flag[it->v] && h[it->v]+it->w == h[u]) {
            int dd = augment(it->v, min(d, it->c), src, sink);
            it->c -= dd, it->pair->c += dd;
            if (!(d -= dd)) break;
        }
    return old-d;
}

int main()
{
    int cases, n, A, B, C;
    for (scanf("%d", &cases); cases--; ) {
        scanf("%d%d%d%d", &n, &A, &B, &C);
        for (int i = 0; i < n; i++) {
            scanf("%d%d%d", &cards[i].a, &cards[i].b, &cards[i].c);
            cards[i].s = cards[i].a+cards[i].b+cards[i].c;
            cards[i].valid = false;
        }

        nth_element(cards, cards+A+B+C, cards+n, cmp_by_a);
        for (int i = 0; i < A+B+C; i++) cards[i].valid = true;
        nth_element(cards, cards+A+B+C, cards+n, cmp_by_b);
        for (int i = 0; i < A+B+C; i++) cards[i].valid = true;
        nth_element(cards, cards+A+B+C, cards+n, cmp_by_c);
        for (int i = 0; i < A+B+C; i++) cards[i].valid = true;
        int nn = 0;
        for (int i = 0; i < n; i++)
            if (cards[i].valid)
                cards[nn++] = cards[i];
        n = nn;

        const int sink = 4+n, tsrc = sink+1, tsink = tsrc+1;
        pit = pool;
        memset(e, 0, sizeof(e));
        mincost = 0;
        insert(0, 1, A, 0);
        insert(0, 2, B, 0);
        insert(0, 3, C, 0);
        for (int i = 0; i < n; i++) {
            const int foo = cards[i].s;
            insert(4+i, 1, 1, cards[i].a*100000+foo); mincost -= cards[i].a*100000+foo;
            insert(4+i, 2, 1, cards[i].b*100000+foo); mincost -= cards[i].b*100000+foo;
            insert(4+i, 3, 1, cards[i].c*100000+foo); mincost -= cards[i].c*100000+foo;
            insert(tsrc, 4+i, 3, 0);
            insert(4+i, sink, 1, 0);
        }
        insert(1, tsink, n, 0);
        insert(2, tsink, n, 0);
        insert(3, tsink, n, 0);
        insert(sink, 0, INT_MAX, 0);

        memset(h, 0, sizeof(h));
        do while (memset(flag, 0, sizeof(flag)), augment(tsrc, INT_MAX, tsrc, tsink));
        while (relabel(tsink));
        do while (memset(flag, 0, sizeof(flag)), augment(0, INT_MAX, 0, sink));
        while (relabel(sink));
        printf("%d %d\n", (-mincost)/100000, (-mincost)%100000);
    }
}
```
