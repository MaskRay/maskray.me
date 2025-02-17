layout: post
title: kth element in a subarray
author: MaskRay
tags: [algorithm]
---

本文总结经典的区间第k小值数据结构题。
给定一个长为n的数组，元素为范围为`[0,σ)`的整数。有m个询问：求区间[l,r)中第k小的元素。

一些方法支持扩展问题：有m个操作，或者修改某个位置上的元素，或者询问区间[l,r)中第k小的元素。

<!-- more -->

## 描述位置区间的线段树/Fenwick tree

归并树(merge sort tree)用`O(n*log(n))`时间构建线段树，每个节点描述一个位置区间。一个节点存储对应区间的有序数组。
对于一个询问，二分搜索答案ans转化为计数问题：区间[l,r)内小于ans的元素个数是否大于等于k。
对于这个计数问题，把区间[l,r)解构为不超过`log(n)`个线段树节点。对于每个节点，二分查找这个节点存储的有序数组里小于ans的元素数。

区间计数支持区间减法，因此外层的线段树也可改成Fenwick tree，减少一半节点。
外层线段树还可以改成块状链表(unrolled linked list)，但只在增删元素的扩展问题上才有用(如洛谷P4278 带插入区间K小值)。

二分搜索答案需要在每个解构的线段树/Fenwick tree区间上进行。查询一个区间的二分搜索无法加速查询下一个区间，因而影响了时间复杂度。

* static: `O(n*log(n)+m*log(n)^2*log(σ))`, space complexity: `O(n*log(n))`, not recommended

如果所有Fenwick tree节点存储形态相同的描述值域的惰性线段树(代替有序数组)，那么查询可以优化。
询问区间[l,r)可以解构为不超过2*log(n)个Fenwick tree节点。我们可以对这些节点存储的多棵值域线段树整体进行查询。
获取这些树根的左孩子值域区间的元素数，判断答案落在左孩子值域区间还是右孩子值域区间。
```cpp
  while (l < h-1) {
    int m = l+h >> 1, lcnt = 0;
    for (int t : span(pos, npos)) lcnt += seg[seg[t].ch[0]].cnt;
    for (int t : span(neg, nneg)) lcnt -= seg[seg[t].ch[0]].cnt;
    int d = k >= lcnt;
    if (d) l = m, k -= lcnt;
    else h = m;
    for (int &t : span(pos, npos)) t = seg[t].ch[d];
    for (int &t : span(neg, nneg)) t = seg[t].ch[d];
  }
```

这样就把二分答案的复杂度和查询线段树混合了。这种算法也支持修改某个位置上的元素。除了需要提前知道获取值域外，修改和查询都可以顺序处理，不需要打乱顺序。
有些中文资料称这种方法为“动态主席树”，但实际上没有用到可持久化线段树的特性。

* static: `O(n*log(n)+m*log(n)*log(σ))`
* dynamic: `O((n+m)*log(n)*log(σ))`, space complexity: `O((n+m)*log(n)*log(σ))`, not recommended

外层Fenwick tree每个节点也能存储binary search tree(这类嵌套树形结构俗称树套树)。空间复杂度降低，但无法应用值域线段树的这种二分优化，单次查询时间复杂度仍为`O(log(n)^2*log(σ))`。

* dynamic: `O(n*log(n)+m*log(n)^2*log(σ))`, space complexity: `O((n+m)*log(n))`, not recommended

```cpp
// 外层为描述位置区间的Fenwick tree，内层为惰性的值域线段树
#include <algorithm>
#include <cstdio>
#include <span>
using namespace std;

#define REP(i, n) for (int i = 0; i < (n); i++)

namespace {
int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar()) s = s*10+c-'0';
  return m ? -s : s;
}

const int N = 100000, M = 100000, LOG2N = 32-__builtin_clz(N), LOG2NM = 32-__builtin_clz(N+M-1);
int a[N], b[N+M], roots[N], pos[LOG2N], neg[LOG2N], allo;
struct Op { bool modify; int l, r, k; } q[M];
struct Segment { int ch[2], cnt; } seg[(N+M*2)*LOG2N*LOG2NM];

void add(int n, int nv, int i, int v) {
  int x = lower_bound(b, b+nv, a[i]) - b;
  for (; i < n; i |= i+1) {
    int l = 0, r = nv, *t = &roots[i];
    for(;;) {
      if (!*t) *t = ++allo;
      seg[*t].cnt += v;
      if (l == r-1) break;
      int m = l+r >> 1, d = x >= m;
      if (d) l = m;
      else r = m;
      t = &seg[*t].ch[d];
    }
  }
}

int kth(int npos, int nneg, int l, int h, int k) {
  while (l < h-1) {
    int m = l+h >> 1, lcnt = 0;
    for (int t : span(pos, npos))
      lcnt += seg[seg[t].ch[0]].cnt;
    for (int t : span(neg, nneg))
      lcnt -= seg[seg[t].ch[0]].cnt;
    int d = k >= lcnt;
    if (d) l = m, k -= lcnt;
    else h = m;
    for (int &t : span(pos, npos))
      t = seg[t].ch[d];
    for (int &t : span(neg, nneg))
      t = seg[t].ch[d];
  }
  return l;
}
}

int main() {
  int n = ri(), m = ri(), nv = n;
  REP(i, n)
    a[i] = b[i] = ri();
  REP(i, m) {
    char c;
    while (c = getchar_unlocked(), c != 'C' && c != 'Q');
    if (c == 'C') {
      q[i] = {true, ri()-1, -1, ri()};
      b[nv++] = q[i].k;
    } else
      q[i] = {false, ri()-1, ri(), ri()};
  }

  sort(b, b+nv);
  nv = unique(b, b+nv) - b;
  REP(i, n)
    add(n, nv, i, 1);
  REP(i, m)
    if (q[i].modify) {
      add(n, nv, q[i].l, -1);
      a[q[i].l] = q[i].k;
      add(n, nv, q[i].l, 1);
    } else {
      int npos = 0, nneg = 0;
      for (int j = q[i].r; j; j &= j-1)
        pos[npos++] = roots[j-1];
      for (int j = q[i].l; j; j &= j-1)
        neg[nneg++] = roots[j-1];
      printf("%d\n", b[kth(npos, nneg, 0, nv, q[i].k-1)]);
    }
}
```

## 描述值域的线段树

构建一棵线段树，每个节点描述一个值域区间。节点存储的内容有多种选择。

### 节点存储位置序列

每个节点描述一个值域区间。一个节点存储对于值域区间里元素的位置序列。对于静态问题，位置序列可以是一个有序数组。
询问时，获取左孩子中落在[l,r)间的元素的个数，判断答案落在左孩子值域区间还是右孩子值域区间。

区间计数支持区间减法，因此外层的线段树也可改成Fenwick tree，减少一半节点。

若要支持修改元素，节点存储binary search tree。也可以存储惰性的线段树，内部节点只在值域区间有元素时才展开，但会增大空间复杂度。

* static: `O(n*log(n)+m*log(n)*log(σ))`, space complexity: `O(n*log(σ))`, not recommended
* dynamic: `O(n*log(n)+m*log(n)*log(σ))`, space complexity: `O((n+m)*log(σ))`, not recommended

### 可持久化线段树(persistent segment tree)

这是描述值域的线段树的另一种优化。中文网络往往叫它“主席树”。
用O(n*log(n))时间构建n+1棵描述值域的线段树。每棵线段树表示一个原数组的一个前缀(共n+1个)。在每棵线段树中，每个节点存储值域区间里的元素数。
相邻两棵线段树描述的区间只相差一个元素，它们可以共用大部分节点，只有ceil(log(n))个节点有差异。

* static: `O((n+m)*log(σ))`, space complexity: `O(n*log(σ))`

```cpp
// persistent segment tree
#include <algorithm>
#include <cstdio>
using namespace std;

#define REP(i, n) for (int i = 0; i < (n); i++)

int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar()) s = s*10+c-'0';
  return m ? -s : s;
}

const int N = 200000, M = 200000, LOG2N = 32-__builtin_clz(N-1);
int a[N], b[N], roots[N+1], allo;
struct Segment { int ch[2], cnt; } seg[N*2+M*LOG2N];

void build(int &t, int l, int r) {
  t = ++allo;
  if (l < r-1) {
    int m = l+r >> 1;
    build(seg[t].ch[0], l, m);
    build(seg[t].ch[1], m, r);
  }
}

void add(int *t, int u, int l, int r, int v) {
  while (l < r-1) {
    *t = ++allo;
    seg[*t].cnt = seg[u].cnt+1;
    int m = l+r >> 1, d = v >= m;
    if (d) l = m;
    else r = m;
    seg[*t].ch[d^1] = seg[u].ch[d^1];
    t = &seg[*t].ch[d];
    u = seg[u].ch[d];
  }
  *t = ++allo;
  seg[*t].cnt = seg[u].cnt+1;
}

int kth(int t, int u, int l, int r, int k) {
  while (l < r-1) {
    int m = l+r >> 1, lcnt = seg[seg[t].ch[0]].cnt-seg[seg[u].ch[0]].cnt, d = k >= lcnt;
    if (d) l = m, k -= lcnt;
    else r = m;
    t = seg[t].ch[d];
    u = seg[u].ch[d];
  }
  return l;
}

int main() {
  int n = ri(), m = ri();
  REP(i, n) {
    a[i] = ri();
    b[i] = a[i];
  }
  sort(b, b+n);
  int nn = unique(b, b+n) - b;
  build(roots[0], 0, nn);
  REP(i, n) {
    int v = lower_bound(b, b+nn, a[i]) - b;
    add(&roots[i+1], roots[i], 0, nn, v);
  }
  while (m--) {
    int l = ri(), r = ri(), k = ri();
    printf("%d\n", b[kth(roots[r], roots[l-1], 0, nn, k-1)]);
  }
}
```

## Wavelet tree及变体

### Wavelet tree

来自R. Grossi, A. Gupta, and J. S. Vitter, _High-order entropy-compressed text indexes_, Proceedings of the 14th Annual SIAM/ACM Symposium on Discrete Algorithms (SODA), January 2003, 841-850.

Wavelet tree是一种succinct data structure，可以在O(log σ)时间求rank、select、和kth操作，其中σ是字符集大小。
树的每个节点描述一个值域区间，存储区间内所有元素。根节点描述原数组。
选取一个2的幂`2^(ceil(log2(r-l))-1)`把区间划分成`[l,2^(ceil(log2(r-l))-1))`和`[l+2^(ceil(log2(r-l))-1),r)`两部分，把元素分配给左右两个孩子。
递归该分配过程，直到值域区间`[l,r)`是单位区间。

Levelwise wavelet tree去除了explicit tree structure，简化了实现，但实现复杂度和表现均不如wavelet matrix。

```
5 4 5 5 2 1 5 6 1 3 5 0
2 1 1 3 0 | 5 4 5 5 5 6 5           # split by bit 2
1 1 0 | 2 3 | 5 4 5 5 5 5 | 6       # split by bit 1
0 | 1 1 | 2 | 3 | 4 | 5 5 5 5 5 | 6 # split by bit 0
```

* static: `O((n+m)*log(sigma))`, space complexity: `O(n)`, not recommended

### 划分树

Wavelet tree的一种变体，使用中位数来分割值域区间，实现较为简单。中文网络称这种方法为“划分树”。
每个节点存储值域区间里按顺序出现的元素数组。
较小的一半元素分配给左孩子，较大的一半元素分配给右孩子。
```
5 4 5 5 2 1 5 6 1 3 5 0
4 2 1 1 3 0 | 5 5 5 5 6 5
1 1 0 | 4 2 3 | 5 5 5 | 5 6 5
0 | 1 1 | 2 | 4 3 | 5 | 5 5 | 5 | 6 5
0 | 1 | 1 | 2 | 4 | 3 | 5 | 5 | 5 | 5 | 6 | 5
```

使用range tree fractional cascading技巧，节点存储另一个数组记录前i个元素中有几个分配到左孩子(wavelet tree的rank0询问)。
对于一个询问[l,r)，可以O(1)知道落在左孩子值域的元素个数，从而答案在左孩子还是右孩子。
另外还能知道[l,r)应该映射为左(或右)孩子的哪一个区间。

空间复杂度为`O(n*log(n))`。应用binary bitmap的rank技术，可以降低到O(n)，但多数资料不用这种技术(算法题中没必要)。

* static: `O((n+m)*log(n))`, space complexity: `O(n)`

### Wavelet matrix

Wavelet tree的每一层在上一层的分割方案下继续分割，形成了树形分割结构。
Wavelet matrix则去除了树形解构，直接把当前层所有0元素放到左边，所有1元素放到右边。
实现复杂度低且实践性能更好。
```cpp
height = sigma == 1 ? 1 : 32-__builtin_clz(sigma-1);
int n = a.size();
for (int i = height; i--; ) {
  ...
  stable_partition(a.begin(), a.end(), [&](int x) { return !(x>>i&1); });
}
```

```
5 4 5 5 2 1 5 6 1 3 5 0
2 1 1 3 0 | 5 4 5 5 5 6 5  # partition by bit 2
1 1 0 5 4 5 5 5 5 | 2 3 6  # partition by bit 1
0 4 2 6 | 1 1 5 5 5 5 5 3  # partition by bit 0
```

* static: `O((n+m)*log(n))`, space complexity: `O(n)`

```cpp
// wavelet matrix
#include <algorithm>
#include <cstdint>
#include <cstdio>
#include <memory>
#include <span>
#include <vector>
using namespace std;

#define REP(i, n) for (int i = 0; i < (n); i++)

namespace {
int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar()) s = s*10+c-'0';
  return m ? -s : s;
}

struct WaveletMatrix {
  int height;
  vector<vector<uint64_t>> b;
  vector<vector<int>> bcnt;
  vector<int> z;
  WaveletMatrix(span<int> a, int sigma) :
      height(sigma == 1 ? 1 : 32-__builtin_clz(sigma-1)),
      b(height, vector<uint64_t>(a.size()/64+1)),
      bcnt(height, vector<int>(a.size()/64+1)),
      z(height) {
    int n = a.size();
    for (int i = height; i--; ) {
      for (int j = 0; j < a.size(); j++)
        b[i][j/64] |= uint64_t(a[j]>>i & 1) << j%64;
      for (int j = 1; j < b[i].size(); j++)
        bcnt[i][j] = bcnt[i][j-1] + __builtin_popcountll(b[i][j-1]);
      z[i] = stable_partition(a.begin(), a.end(), [&](int x) { return !(x>>i&1); }) - a.begin();
    }
  }
  int rank1(int i, int x) const {
    return bcnt[i][x/64] + __builtin_popcountll(b[i][x/64] & ((uint64_t(1) << x%64) - 1)); 
  };
  int kth(int l, int r, int k) const {
    int res = 0;
    for (int i = height; i--; ) {
      int rnkl = rank1(i, l), rnkr = rank1(i, r), cnt0 = r-rnkr-(l-rnkl);
      if (k < cnt0) {
        l -= rnkl;
        r -= rnkr;
      } else {
        k -= cnt0;
        l = z[i]+rnkl;
        r = z[i]+rnkr;
        res |= 1 << i;
      }
    }
    return res;
  }
};
}

int main() {
  int n = ri(), m = ri();
  auto a = std::make_unique_for_overwrite<int[]>(n);
  auto b = std::make_unique_for_overwrite<int[]>(n);
  REP(i, n)
    a[i] = b[i] = ri();
  sort(b.get(), b.get()+n);
  int nn = unique(b.get(), b.get()+n) - b.get();
  REP(i, n)
    a[i] = lower_bound(b.get(), b.get()+nn, a[i]) - b.get();
  WaveletMatrix wm(span(a.get(), n), nn);
  while (m--) {
    int l = ri()-1, r = ri(), k = ri();
    printf("%d\n", b[wm.kth(l, r, k-1)]);
  }
}
```

## 整体二分(parallel binary search)

有多组修改和询问。每个询问会受到时间序之前的修改的影响，询问目标可以二分搜索。
这类离线算法将二分答案应用到多组修改和询问上。

假设我们要处理编号为`[l,h)`的询问，它们的答案在值域区间`[vl,vh)`内(`vl`表示区间内最小的元素)。

* 取`vm=(vl+vh)/2`，把值域在`[vl,vm)`区间内的元素根据位置插入到Fenwick tree里
* 对于编号为`[l,h)`的每一个询问，计算Fenwick tree在该区间的元素数，和k值比较，决定它的答案落在`[vl,vm)`值域区间(k值不变)还是`[vm,vh)`值域区间(k值减少)
* 把值域在`[vl,vm)`区间内的元素根据位置从Fenwick tree里删除
* 原问题分治为两个子问题，处理值域区间`[vl,vm)`的询问和值域区间`[vm,vh)`的询问

在二分答案后，单点修改的影响为commutative monoid，区间询问的目标也是一个commutative monoid。

* static: `O((n+m)*log(n)*log(σ))`, space complexity: `O(n+m)`
* dynamic: `O((n+m)*log(n+m)*log(σ))`, space complexity: `O(n+m)`

```cpp
// parallel binary search, static
#include <algorithm>
#include <cstdio>
#include <utility>
using namespace std;

#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define REP(i, n) for (int i = 0; i < (n); i++)

const int N = 200000, M = 200000;

namespace {
int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar()) s = s*10+c-'0';
  return m ? -s : s;
}

pair<int, int> a[N];
int ans[M], fenwick[N], n;
struct Query { int id, l, r, k; } q[M], qq[M];

void add(int i, int d) {
  for (; i < n; i |= i+1)
    fenwick[i] += d;
}

int get_sum(int i) {
  int sum = 0;
  for (; i; i &= i-1)
    sum += fenwick[i-1];
  return sum;
}

void conquer(int ml, int mh, int l, int h) {
  if (ml == mh-1) {
    FOR(i, l, h)
      ans[q[i].id] = a[ml].first;
    return;
  }
  int mm = ml+mh >> 1, nl = 0, nh = h-l;
  FOR(i, ml, mm)
    add(a[i].second, 1);
  FOR(i, l, h) {
    int t = get_sum(q[i].r)-get_sum(q[i].l);
    if (q[i].k <= t)
      qq[nl++] = q[i];
    else
      qq[--nh] = q[i], qq[nh].k -= t;
  }
  FOR(i, ml, mm)
    add(a[i].second, -1);
  copy_n(qq, nl, q+l);
  copy(qq+nh, qq+h-l, q+l+nl);
  if (nl) conquer(ml, mm, l, l+nl);
  if (l+nl < h) conquer(mm, mh, l+nl, h);
}
}

int main() {
  n = ri();
  int m = ri();
  REP(i, n)
    a[i] = {ri(), i};
  REP(i, m)
    q[i] = {i, ri()-1, ri(), ri()};
  sort(a, a+n);
  conquer(0, n, 0, m);
  REP(i, m)
    printf("%d\n", ans[i]);
}
```

```cpp
// parallel binary search, dynamic
#include <algorithm>
#include <cstdio>
#include <utility>
using namespace std;

#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define REP(i, n) for (int i = 0; i < (n); i++)

const int N = 100000, M = 100000;

namespace {
int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar_unlocked())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar_unlocked()) s = s*10+c-'0';
  return m ? -s : s;
}

int a[N+M], ans[M], fenwick[N], n;
struct Op { int id, l, r, k; } q[N+2*M], qq[N+2*M];

void add(int i, int d) {
  for (; i < n; i |= i+1)
    fenwick[i] += d;
}

int get_sum(int i) {
  int sum = 0;
  for (; i; i &= i-1)
    sum += fenwick[i-1];
  return sum;
}

void conquer(int vl, int vh, int l, int h) {
  if (vl == vh-1) {
    FOR(i, l, h)
      if (q[i].id >= 0)
        ans[q[i].id] = a[vl];
    return;
  }
  int vm = vl+vh >> 1, nl = 0, nh = h-l;
  FOR(i, l, h) {
    auto x = q[i];
    if (x.id < 0) {
      if (x.k < vm)
        qq[nl++] = x, add(x.l, x.r);
      else
        qq[--nh] = x;
    } else {
      int t = get_sum(x.r)-get_sum(x.l);
      if (x.k <= t)
        qq[nl++] = x;
      else
        x.k -= t, qq[--nh] = x;
    }
  }
  REP(i, nl)
    if (qq[i].id < 0)
      add(qq[i].l, -qq[i].r);
  copy_n(qq, nl, q+l);
  reverse_copy(qq+nh, qq+h-l, q+l+nl);
  if (nl) conquer(vl, vm, l, l+nl);
  if (l+nl < h) conquer(vm, vh, l+nl, h);
}
}

int main() {
  n = ri();
  int m = ri(), nv = n, nop = n, nq = 0;
  REP(i, n) {
    a[i] = ans[i] = ri();
    q[i] = {-1, i, 1, a[i]};
  }
  REP(i, m) {
    char c;
    while (c = getchar_unlocked(), c != 'C' && c != 'Q');
    if (c == 'C') {
      int j = ri()-1;
      q[nop++] = {-1, j, -1, a[j]};
      a[j] = a[nv++] = ri();
      q[nop++] = {-1, j, 1, a[j]};
    } else
      q[nop++] = {nq++, ri()-1, ri(), ri()};
  }
  copy_n(ans, n, a);
  sort(a, a+nv);
  nv = unique(a, a+nv) - a;
  REP(i, nop)
    if (q[i].id < 0)
      q[i].k = lower_bound(a, a+nv, q[i].k) - a;
  conquer(0, nv, 0, nop);
  REP(i, nq)
    printf("%d\n", ans[i]);
}
```

## 莫涛算法(Mo's algorithm)

* static: `O(n*log(n)+m*sqrt(n)+m*log(m))`
* dynamic (binary search on the value, 二分答案): `O(n*log(n)+m*sqrt(n)*log(n)*log(n+m))`

静态情形：维护两个频度数组，一个表示元素x的频度，另一个表示元素区间(如[i,i+block_size))的频度。区间长度加减一时，O(1)修改频度。
```cpp
c1[a[i]] += d;
c2[block[a[i]]] += d;
```

询问时O(sqrt(n))扫描频度数组得到答案。
```cpp
int x = 0, k = qs[i].k;
while (c2[x] < k) k -= c2[x++];
for (int j = x*block_size; ; j++)
  if ((k -= c1[j]) <= 0) {
    qs[i].ans = j;
    break;
  }
```

要点在于不要用有序数据结构维护区间内的元素，会不必要增大修改的时间复杂度。

要支持修改元素，可在每个分块里里维护一个有序数组。
修改时重建有序数组。
询问时二分答案ans。在包含的分块里二分搜索小于ans的元素数。在分块外线性遍历至多2*block_size个元素

```cpp
// Mo's algorithm
#include <algorithm>
#include <cmath>
#include <cstdio>
using namespace std;

#define REP(i, n) for (int i = 0; i < (n); i++)

int ri() {
  int m = 0, s = 0; unsigned c;
  while ((c = getchar())-'0' >= 10u) m = c == '-';
  for (; c-'0' < 10u; c = getchar()) s = s*10+c-'0';
  return m ? -s : s;
}

const int N = 200000, M = 200000;
int a[N], b[N], block[N], c1[N], c2[N], block_size;
struct Query {
  int l, r, k, id, ans;
  bool operator<(const Query &o) const {
    int i = block[l], j = block[o.l];
    if (i != j) return i < j;
    return i & 1 ? r < o.r : r > o.r;
  }
} qs[M];

static void add(int i, int d) {
  c1[a[i]] += d;
  c2[block[a[i]]] += d;
}

int main() {
  int n = ri(), m = ri();
  REP(i, n) {
    a[i] = ri();
    b[i] = a[i];
  }
  sort(b, b+n);
  int nn = unique(b, b+n) - b;
  REP(i, n)
    a[i] = lower_bound(b, b+nn, a[i]) - b;
  REP(i, m) {
    qs[i].l = ri()-1;
    qs[i].r = ri();
    qs[i].k = ri();
    qs[i].id = i;
  }
  block_size = sqrt(n);
  REP(i, n)
    block[i] = i/block_size;
  sort(qs, qs+m);

  int l = 0, r = 0;
  REP(i, m) {
    while (qs[i].l < l) add(--l, 1);
    while (r < qs[i].r) add(r++, 1);
    while (l < qs[i].l) add(l++, -1);
    while (qs[i].r < r) add(--r, -1);
    int x = 0, k = qs[i].k;
    while (c2[x] < k) k -= c2[x++];
    for (int j = x*block_size; ; j++)
      if ((k -= c1[j]) <= 0) {
        qs[i].ans = j;
        break;
      }
  }
  REP(i, m)
    a[qs[i].id] = b[qs[i].ans];
  REP(i, m)
    printf("%d\n", a[i]);
}
```

这在算法通用性更强，支持很多特殊操作，比如[Ynoi2018] 未来日记、P4278 带插入区间K小值。

## 总结

对于静态问题，推荐(有先后)：wavelet matrix、整体二分、莫涛算法、可持久化线段树

对于元素可修改问题，推荐(有先后)：整体二分、Fenwick tree套惰性值域线段树
