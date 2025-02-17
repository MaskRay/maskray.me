---
layout: post
title: 稱球問題擴展——根據當前已知信息選擇最優稱量方案
author: MaskRay
tags: [algorithm, oi, puzzle]
---

似乎是某個TopCoder SRM的題目。

N 個球，有一個是壞的，壞球重量與好球不同（略輕/重於好球）。有一個天平，可以告知左右兩邊孰重孰輕，最小化最壞情況下稱量次數，給出稱量方案。下面討論該問題的一個擴展——如何通過已知信息（即之前稱量結果，之前的稱量結果可能不是最優的）選擇下一步稱量方案，最小化最壞情況下的稱量次數。

根據已有信息把所有球分爲4類：
- 肯定是好球
- 不可能比好球輕（不能確定是否爲壞球）
- 不可能比好球重（不能確定是否爲壞球）
- 不知道任何信息

每個球必定屬於以上四類中的某一類。我們只需知道每一類的數目即可，具體編號無關緊要。令$s[nh][nl][nu]$表示有$nh$個1類球，nl 個2類球，nu 個3類球，最壞情況下需要稱量的次數。枚舉一邊放置的各類型球數 lh(1類),ll(2類),lu(3類)，枚舉另一邊放置的各類型球數 rh(1類),rl(2類),ru(3類),re(0類)。如果兩邊都有0類球，那麼我們可以在兩邊各移去一個。我們可以保證0類球只出現在一邊，不妨設爲 rh(1類),rl(2類),ru(3類) 一邊。

<!-- more -->

```
s[nh][nl][nu] = max(
s[lh+lu][rl+ru][0]+1, #lh,ll,lu一邊重於rh,rl,ru,re一邊
s[rh+ru][ll+lu][0]+1, #lh,ll,lu一邊輕於rh,rl,ru,re一邊
s[nh-lh-rh][nl-ll-rl][nu-lu-ru]+1, #一樣重
)
```

下面程序應用到了 C++0x 的 multi-declarator auto，該特性已經被 GCC 4.4 實現。

```cpp
#include <algorithm>
#include <climits>
#include <iostream>
#include <iterator>
#include <string>
#include <vector>
using namespace std;
typedef vector<int> vi;

const int N = 81;
int n, mem[N][N][N];
basic_string<int> res, se, sh, sl, su;

int compute(int ne, int nh, int nl, int nu, bool flag)
{
        if (ne == n || nh+nl == 1 && ne == n-1)
                return 0;
        int &ret = mem[nh][nl][nu];
        if (ret) return ret;
        ret = INT_MAX/2;
        for (int lh = 0; lh <= nh; ++lh)
                for (int ll = 0; ll <= nl; ++ll)
                        for (int lu = 0; lu <= nu && lh+ll+lu <= n/2; ++lu)
                                for (int rh = 0; rh <= nh-lh; ++rh)
                                        for (int rl = 0; rl <= nl-ll; ++rl)
                                                for (int ru = 0; ru <= nu-lu; ++ru)
                                                        if (lh+ll+lu >= rh+rl+ru)
                                                        {
                                                                int re = lh+ll+lu-rh-rl-ru;
                                                                if (re > ne) continue;
                                                                int t = max(max(
                                                                                compute(n-lh-lu-rl-ru, lh+lu, rl+ru, 0, false)+1,
                                                                                compute(n-rh-ru-ll-lu, rh+ru, ll+lu, 0, false)+1),
                                                                        compute(ne+lh+ll+lu+rh+rl+ru, nh-lh-rh, nl-ll-rl, nu-lu-ru, false)+1);
                                                                if (flag && t < ret)
                                                                {
                                                                        basic_string<int> s1 = sh.substr(0, lh)+sl.substr(0, ll)+su.substr(0, lu),
                                                                                   s2 = sh.substr(lh, rh)+sl.substr(ll, rl)+su.substr(lu, ru)+se.substr(0, re);
                                                                        sort(s1.begin(), s1.end());
                                                                        sort(s2.begin(), s2.end());
                                                                        res = s1;
                                                                        res.push_back(-1);
                                                                        res += s2;
                                                                }
                                                                ret = min(ret, t);
                                                        }
        return ret;
}

bool solve(vector<vi> left, vector<vi> right, string result)
{
        vi p(n, 3);
        for (int i = 0; i < left.size(); ++i)
                if (result[i] == 'E')
                {
                        for (auto it = left[i].begin(); it != left[i].end(); ++it)
                                p[*it] = 0;
                        for (auto it = right[i].begin(); it != right[i].end(); ++it)
                                p[*it] = 0;
                }
                else
                {
                        if (result[i] == 'R') left[i].swap(right[i]);
                        for (auto it = left[i].begin(); it != left[i].end(); ++it)
                                p[*it] &= ~2;
                        for (auto it = right[i].begin(); it != right[i].end(); ++it)
                                p[*it] &= ~1;
                        for (int j = 0; j < n; ++j)
                                if (find(left[i].begin(), left[i].end(), j) == left[i].end()
                                        && find(right[i].begin(), right[i].end(), j) == right[i].end())
                                        p[j] = 0;
                }
        for (int i = 0; i < n; ++i)
                switch (p[i])
                {
                        case 0: se.push_back(i); break;
                        case 1: sh.push_back(i); break;
                        case 2: sl.push_back(i); break;
                        case 3: su.push_back(i);
                }
        if (se.size() == n) return false;
        compute(se.size(), sh.size(), sl.size(), su.size(), true);
        return true;
}

int main()
{
        int m;
        string result;
        vector<vi> left, right;

        cin >> n >> m;
        while (m--)
        {
                char op;
                int num, x;
                vi L, R;
                cin >> num;
                for (int i = 0; i < num; ++i)
                        cin >> x, L.push_back(x);
                for (int i = 0; i < num; ++i)
                        cin >> x, R.push_back(x);
                while (isspace(cin.peek())) cin.ignore();
                cin >> op;
                result += op;
                left.push_back(L);
                right.push_back(R);
        }

        if (!solve(left, right, result))
                cerr << "error\n";
        else if (!res.empty())
        {
                auto it = find(res.begin(), res.end(), -1);
                copy(res.begin(), it, ostream_iterator<int>(cout, " "));
                cout << "- ";
                copy(it+1, res.end(), ostream_iterator<int>(cout, " "));
                cout << endl;
        }
        return 0;
}
```

輸入格式：
兩個整數 n,m，分別表示球的數量和已經稱量的次數
之後 m 行，每一行第一個數 t，之後是 t 個數 a[0...t-1]，代表天平左邊各個球的編號，然後又是 t 個數 b[0...t-1]，代表天平右邊各個球的編號，接着是一個字符 c
表示天平左右兩邊分別是 a[] 和 b[]，字符 c 表示稱量結果, {E: 一樣中，L: 左邊重，R: 右邊重}

輸出結果爲下一步的其中一種最優稱量方案

比如輸入：

```
12 2
4  0 1 2 3  4 5 6 7  E
3  0 1 2    8 9 10   L
```

程序會輸出：

```
8 - 9
```

表示有12個球，[0 1 2 3] 和 [4 5 6 7] 的稱量結果是一樣重
[0 1 2] 重於 [8 9 10]
易知壞球在 {8 9 10} 中，下一步比較 8 和 9 即可
