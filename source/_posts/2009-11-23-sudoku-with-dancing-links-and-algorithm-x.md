---
layout: post
title: Dancing Links+Algorithm X求解數獨
author: MaskRay
tags: [algorithm, oi, puzzle]
---

```cpp
#include <algorithm>
#include <limits>
#include <cstdio>
#include <cstring>
using namespace std;

const int N = 9, LEN = 3; //size
const int MAX_R = N*N*N, MAX_C = N*N*4, MAX = MAX_C+1+MAX_R*4;
int L[MAX], R[MAX], U[MAX], D[MAX], size[MAX_C+1], column[MAX];
int res[N*N]; //result
pair<int, int> mean[MAX]; //first: pos. second: number

int mapping[256];
char mapping2[N];

void insert(int cur, int left, int right, int top)
{
        L[cur] = left; R[cur] = right;
        U[cur] = U[top]; D[U[cur]] = cur;
        D[cur] = top; U[top] = cur;
        column[cur] = top;
        ++size[top];
}

void remove(int c)
{
        L[R[c]] = L[c];
        R[L[c]] = R[c];
        for (int i = D[c]; i != c; i = D[i])
                for (int j = R[i]; j != i; j = R[j])
                {
                        U[D[j]] = U[j];
                        D[U[j]] = D[j];
                        --size[column[j]];
                }
}

void resume(int c)
{
        for (int i = U[c]; i != c; i = U[i])
                for (int j = L[i]; j != i; j = L[j])
                {
                        U[D[j]] = j;
                        D[U[j]] = j;
                        ++size[column[j]];
                }
        L[R[c]] = c;
        R[L[c]] = c;
}

bool DLX(int k)
{
        if (R[MAX_C] == MAX_C) return true;
        int c, mins = numeric_limits<int>::max();
        for (int i = R[MAX_C]; i != MAX_C; i = R[i])
                if (size[i] < mins)
                        mins = size[i], c = i;
        remove( c );
        for (int i = D[c]; i != c; i = D[i])
        {
                for (int j = R[i]; j != i; j = R[j])
                        remove(column[j]);
                res[mean[i].first] = mean[i].second;
                if (DLX(k+1)) return true;
                for (int j = L[i]; j != i; j = L[j])
                        resume(column[j]);
        }
        resume( c );
        return false;
}

int main()
{
        char str[N*N+1], tmp[N*N+1];
        str[0] = 0;
        mapping['1'] = 0; mapping2[0] = '1';
        mapping['2'] = 1; mapping2[1] = '2';
        mapping['3'] = 2; mapping2[2] = '3';
        mapping['4'] = 3; mapping2[3] = '4';
        mapping['5'] = 4; mapping2[4] = '5';
        mapping['6'] = 5; mapping2[5] = '6';
        mapping['7'] = 6; mapping2[6] = '7';
        mapping['8'] = 7; mapping2[7] = '8';
        mapping['9'] = 8; mapping2[8] = '9';
        while (strlen(str) < N*N)
                scanf("%s", tmp), strcat(str, tmp);

        int cur = MAX_C+1;
        for (int i = 0; i <= MAX_C; ++i)
        {
                L[i] = i-1; R[i] = i+1;
                U[i] = i; D[i] = i;
                size[i] = 0;
        }
        L[0] = MAX_C; R[MAX_C] = 0;
        for (int i = 0; i < N; ++i)
                for (int j = 0; j < N; ++j)
                        if (str[i*N+j] == '.')
                        {
                                for (int k = 0; k < N; ++k)
                                {
                                        insert(cur, cur+3, cur+1, i*N+k); mean[cur++] = make_pair(i*N+j, k);
                                        insert(cur, cur-1, cur+1, N*N+j*N+k); mean[cur++] = make_pair(i*N+j, k);
                                        insert(cur, cur-1, cur+1, N*N*2+(i/LEN*LEN+j/LEN)*N+k); mean[cur++] = make_pair(i*N+j, k);
                                        insert(cur, cur-1, cur-3, N*N*3+i*N+j); mean[cur++] = make_pair(i*N+j, k);
                                }
                        }
                        else
                        {
                                const int k = mapping[str[i*N+j]];
                                insert(cur, cur+3, cur+1, i*N+k); mean[cur++] = make_pair(i*N+j, k);
                                insert(cur, cur-1, cur+1, N*N+j*N+k); mean[cur++] = make_pair(i*N+j, k);
                                insert(cur, cur-1, cur+1, N*N*2+(i/LEN*LEN+j/LEN)*N+k); mean[cur++] = make_pair(i*N+j, k);
                                insert(cur, cur-1, cur-3, N*N*3+i*N+j); mean[cur++] = make_pair(i*N+j, k);
                                res[i*N+j] = k;
                        }

        if (!DLX(0))
                puts("No solution.");
        else
                for (int i = 0; i < N; ++i, puts(""))
                        for (int j = 0; j < N; ++j)
                                putchar(mapping2[res[i*N+j]]);
}
```

## 2013年8月17日更新

NOIP 2009考了靶形數獨，可能前一天我還敲了一遍這段代碼……
