---
layout: post
title: NOIP 2004 數字遊戲（蟲食算）
author: MaskRay
tags: [algorithm, oi, noip]
---

下面程序可計算如下形式的式子：

```
one
+  nine
+ fifty
+twenty
-------
eighty

a
+a
--
bc
```

方法是枚舉進位+解方程，涉及的字母以及各個位上的進位作爲變量，然後解方程。這樣可以用　各個位上的進位　和　字母變量中的自由變量　表示　其他字母變量。枚舉　各個位上的進位　和　字母變量中的自由變量　以求出　其他字母變量，以此得到各組解。使用方法：根據要求解的式子設置各個變量，N、M設置得大點沒關係，L要正好等於式子的行數，然後輸入L行字符串

<!-- more -->

舉例：

```
N=10（10個變量）
M=6（一行最大長度6）
L=5（5行）
```

輸入

```
one
nine
fifty
twenty
eighty
```

```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <iostream>
#include <math.h>
#include <string>
#include <vector>
using namespace std;

const int N = 10; //maximum number of variables
const int M = 6; //maximum line length
const int L = 3; //number of lines
string s[L];
int len[L], mapping[256];
int m = 0, n = 0, res;
int pi[N], x[M+N], a[M][M+N], d[M];
vector<int> fre;

int gcd(int x, int y)
{
        int t;
        x = abs(x);
        y = abs(y);
        while (y) t = x%y, x = y, y = t;
        return x;
}

bool GaussJordanElimination()
{
        memset(pi, 0xFF, sizeof(pi));
        for (int i = 0; i < m; ++i)
        {
                int pivot = i;
                if (!a[i][i])
                {
                        for (int j = 0; j < n; ++j)
                                if (a[i][j]) pivot = j;
                        if (pivot == i) continue;
                }
                pi[pivot] = i;
                for (int j = 0; j < m; ++j)
                        if (j != i && a[j][pivot])
                        {
                                int lcm = a[i][pivot] / gcd(a[i][pivot], a[j][pivot]) * a[j][pivot], k1 = lcm/a[i][pivot], k2 = lcm/a[j][pivot];
                                for (int k = 0; k < m+n; ++k)
                                        a[j][k] = a[j][k]*k2-a[i][k]*k1;
                        }
        }
        for (int i = 0; i < n; ++i)
                if (pi[i] == -1)
                        fre.push_back(i);
}

void DFS_left(int k)
{
        if (k == fre.size())
        {
                for (int i = 0; i < n; ++i)
                        if (pi[i] != -1)
                        {
                                int t = 0;
                                for (int j = 0; j < fre.size(); ++j)
                                        t -= a[pi[i]][fre[j]]*x[fre[j]];
                                for (int j = 0; j < m; ++j)
                                        t -= a[pi[i]][n+j]*x[n+j];
                                if (t%a[pi[i]][i]) return;
                                x[i] = t/a[pi[i]][i];
                                if (x[i] < 0 || x[i] >= 10) return;
                        }
                for (int i = 0; i < L; ++i)
                {
                        for (int j = m-s[i].size(); j; --j) printf("   ");
                        for (string::iterator it = s[i].begin(); it != s[i].end(); ++it)
                                printf("%3d", x[mapping[*it]]);
                        puts("");
                }
                puts("");
                ++res;
        }
        else for (int i = 0; i < 10; ++i)
                         x[fre[k]] = i, DFS_left(k+1);
}

void DFS_right(int k)
{
        if (k == m)
                DFS_left(0);
        else for (int i = 0; i < L-1; ++i)
                         x[n+k] = i, DFS_right(k+1);
}

int main()
{
        memset(mapping, 0xFF, sizeof(mapping));
        memset(a, 0, sizeof(a));
        for (int i = 0; i < L; ++i)
        {
                cin >> s[i];
                if (s[i].size() > m) m = s[i].size();
                for (int j = 0; j < s[i].size(); ++j)
                {
                        if (mapping[s[i][j]] == -1)
                                mapping[s[i][j]] = n++;
                        a[s[i].size()-1-j][mapping[s[i][j]]] += i == L-1 ? -1 : 1;
                }
        }

        for (int i = 0; i < m; ++i)
        {
                if (i) a[i][n+i] = 1;
                if (i < m-1) a[i][n+i+1] = -10;
        }
        GaussJordanElimination();
        res = 0;
        DFS_right(1);
        printf("\n%d solution(s).\n", res);
}
```
