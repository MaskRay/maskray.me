---
layout: post
title: 三柱漢諾塔相關擴展問題
author: MaskRay
mathjax: true
tags: [algorithm, hanoi]
---

設盤子編號爲$1\sim N$，編號大的尺寸大，有三根柱子$1\sim 3$。

輸出初始局面（所有盤子都在1號柱子）到終止局面（所有盤子都在3號柱子）的最優方案。
時間複雜度：$O(2^N)$
經典遞歸，略。

輸出最優方案中某一局面之前經過的步驟數。
時間複雜度：$O(N)$

```cpp
#include <cstdio>
#include <cstdlib>
using namespace std;

const int N = 1000;
int pos[N+1], nmoves = 0;

void move(int maxdisc, int src, int dst)
{
    if (pos[maxdisc] == 6-src-dst)
    {
        fprintf(stderr, "error: this is not a midway of the best solution.\n");
        exit(1);
    }
    if (pos[maxdisc] == src)
    {
        if (maxdisc > 1)
            move(maxdisc-1, src, 6-src-dst);
        pos[maxdisc] = dst;
    }
    else
    {
        nmoves += 1 << maxdisc-1;
        if (maxdisc > 1)
            move(maxdisc-1, 6-src-dst, dst);
    }
}

int main()
{
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; ++i)
        scanf("%d", &pos[i]);
    move(n, 1, 3);
    printf("It is a midway of the best solution and there is/are %d move(s) before.\n", nmoves);
}
```

輸出當前局面（不一定是最優解的中間步驟）到終止局面的最優方案。
時間複雜度：$Ο(2^N)$

```cpp
#include <cstdio>
using namespace std;

const int N = 1000;
int pos[N+1], nmoves = 0;

void move(int disc, int dst)
{
    if (pos[disc] == dst) return;
    for (int i = disc-1; i; --i)
        move(i, 6-pos[disc]-dst);
    printf("%d: %d -> %d\n", disc, pos[disc], dst);
    pos[disc] = dst;
    ++nmoves;
}

int main()
{
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; ++i)
        scanf("%d", &pos[i]);

    //for (int i = n-1; i; --i)
    //    move(i, pos[n]);

    for (int i = n; i; --i)
        move(i, 3);

    printf("total moves: %d\n", nmoves);
}
```

輸出當前局面下，把所有盤子移動到任一根柱子的最優方案。
時間複雜度：Ο(2^N)

```cpp
#include <cstdio>
using namespace std;

const int N = 1000;
int pos[N+1], nmoves = 0;

void move(int disc, int dst)
{
    if (pos[disc] == dst) return;
    for (int i = disc-1; i; --i)
        move(i, 6-pos[disc]-dst);
    printf("%d: %d -> %d\n", disc, pos[disc], dst);
    pos[disc] = dst;
    ++nmoves;
}

int main()
{
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; ++i)
        scanf("%d", &pos[i]);
    for (int i = n-1; i; --i)
        move(i, pos[n]);
    printf("total moves: %d\n", nmoves);
}
```

輸出當前局面下，把所有盤子移動到任一根柱子的最優方案所需步數。
時間複雜度：$Ο(N)$

易知，必定把$1\sim N-1$移動到$N$處
否則的話，需要至少$2^{N-1}$步，不會最優。
所以$N$所在位置就是 目標柱子

注意到如果$N,N-1,\ldots,k+1$初始時都在同一個根柱子上，那就不用再管它們了。
只需考慮$k,\ldots,1$

之所以要移動$i$，要麼是爲了給比它編號大的盤子讓路，要麼是因爲要把它移動到目標柱子。
所以，如果要把$i$移動到某根柱子$X$，那麼$i-1,i-2,\ldots,1$也應該移動到柱子$X$

```
#include <stdio.h>

const int N = 10000;
int cnt = 0, pos[N+1];

void move(int maxdisc, int dst)
{
    if (!maxdisc) return;
    int src = pos[maxdisc], aux = 6-src-dst;
    if (src == dst)
        move(maxdisc-1, dst);
    else
    {
        move(maxdisc-1, aux);
        cnt += 1 << maxdisc-1;
    }
}

int main()
{
    int n;
    scanf("%d", &n);
    for (int i = 1; i <= n; ++i)
        scanf("%d", &pos[i]);
    move(n-1, pos[n]);
    printf("%d\n", cnt);
}
```

總結一下，對於求方案的問題，因爲最壞情況下步驟數可以達到$Ο(2^N)$，所以以上源碼在漸進意義上達到最優複雜度。

對於求步驟數（無須輸出方案），以上源碼時間複雜度都是$Ο(N)$，由於輸入時間已達到$Ο(N)$，所以在漸進意義上也達到了最優複雜度。
