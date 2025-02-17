layout: post
title: Segment tree
author: MaskRay
tags: [algorithm]
---

In research papers, a segment tree refers to a tree data structure allowing retrieving a list of segments which contain the given point.
In competitive programming, the name "segment tree" usually refers to a data structure maintaining an array.
According to <http://web.ntnu.edu.tw/~algo/Sequence2.html>, the data structure originated from Baltic OI 2001: Mars Maps.

<!-- more -->

Let's say we have an array with n elements `a[0],a[1],...,a[n-1]`. We will refer to the length by n below.
There are range modifications and range queries.
Our segment tree stores the elements in its leaf nodes, supporting range modifications and range queries in O(log(n)) time.
We can make an in-order traversal at any time to retrieve the updated array.

A range modification applies a unique function `f` to every element whose index is in the range `[l,r)` (note: `f` can be parameterized on the index).
The most frequently used range modification operations are:

* add an integer (this is commutative, no need to use lazy propagation)
* set to an integer

A range query computes the monoid sum of the elements `a[l],a[l+1],...,a[r-1]`.
Some of the most frequently used binary operations for range queries:

* sum: `a[l]+a[l+1]+...+a[r-1]`
* max (range maximum query): `max(a[l],a[l+1],...,a[r-1])`
* min (range minimum query): `min(a[l],a[l+1],...,a[r-1])`

There are many others. The requirement is that the binary operation is associative and there is an identity element.
Strictly speaking, an identity element is not essential, but it makes implementation simple and is trivial to augment once we have a semigroup.
For sum/max/min, the binary operations are also commutative. For non-commutative operations, we should watch for the computation order when implementaing a range query.

Each leaf node stores an element in the original array.
Two adjacent leaf nodes form an inner node which represents an interval of length 2.
Next, two adjacent nodes with length 2 form an inner node which represents an interval of length 4.
Then, two adjacent nodes with length 4 form an inner node which represents an interval of length 8.
This process is recursive. In the end we have a root node of length n.
(This exposition assumes that n is a power of two. If not, some length 2 nodes may join with length 1 nodes.)

For a range modification or range query, the algorithm performs divide and conquer and splits the original interval into several disjoint (but consecutive) subintervals corresponding to nodes.
If n is a power of two greater than or equal to 4, any query `[l,r)` where `0<=l<r<=n` can be split into 2*(log2(n)-1) or fewer nodes.
As a worse case example, when n=16, `[1,15)` splits into: `[1,2) [2,4) [4,8) [8,12) [12,14) [14,15)`.

## Top-down representations

The root represents the whole array `[0,n)`. Then we divide its interval into halves and the two children of the root will represent `[0,n/2)` and `[n/2,n)`.
For each node we check whether the length is greater than 1, and if so, recursively divide the interval into halves.

The tree structure can be represented in multiple ways. Let's start with the verbose explicit representation.

### Explicit tree structure

The pointers to the left and right children are explicitly stored in a node.
When the size is not a concern, the `[l,r)` interval is stored into the node as well.

This representation is useful when:

* there is a need to pack more than one elements into one node. This may be handy and is difficult to be replaced by an iterative bottom-up approach.
* implementating a persistent segment tree.

For other problems, this representation is not recommended.

```cpp
const int N = 1000000, P = 1000000;
int val[P];
long sum[N];
struct Tree *pit;
struct Tree {
  int l, r, minv;
  Tree *left, *right;
  Tree() {}
  Tree(int l, int r) : l(l), r(r), minv(INT_MAX) {
    if (r-l >= 12) left = new Tree(l, l+r >> 1), right = new Tree(l+r >> 1, r);
  }
  void *operator new(size_t) {return pit++; }
  void insert(int u, int v) {
    minv = min(minv, v);
    if (r-l < 12) val[u] = v;
    else (u < l+r >> 1 ? left : right) -> insert(u, v);
  }
  int query(int ll, int rr) {
    if (r <= ll || rr <= l) return INT_MAX;
    if (minv == INT_MAX || ll <= l && r <= rr) return minv;
    if (r-l < 12) return *min_element(val + max(L, ll), val + min(R, rr));
    return min(left->query(ll, rr), right->query(ll, rr));
  }
} pool[P];
```

### Implicit tree structure

#### Binary heap representation (BFS layout)

Let the root node's index (BFS index) be 1. For any node `i`, its left and right children are `2*i` and `2*i+1` respectively.

This representation wastes the index 0.
I prefer zero based numbering (Edsger W. Dijkstra: _Why numbering should start at zero_).
However, the segment tree is an exception.
The reason is that if you have the index of a leaf, you can subtract `n` from it to get the corresponding array index.
While with one based numbering, you need to subtract `n-1`, which is slightly inconvenient.
In addition, getting the index of an ancestor node needs just a right shift operation while with zero based numbering you need one extra plus and one extra minus.
The inconvenience will add to the complexity of lazy propagation.

Typically n is a power of 2. The segment tree is a full binary tree.
If n is not a power of 2, we can pad the length by adding tail sentinel elements.
In the worst case, when `n=2^k+1`, we need to pad `2^k-1` elements. The space consumption is roughly `4*n`.

<style type="text/css">
table.segment-tree {
  table-layout: fixed;
}
table.segment-tree td {
  border: 1px solid #ddd;
  text-align: center;
}
</style>

<table class="segment-tree">
<tr> <td colspan=16><b>1</b>:[0,16)</td> </tr>
<tr> <td colspan=8><b>2</b>:[0,8)</td> <td colspan=8><b>3</b>:[8,16)</td> </tr>
<tr> <td colspan=4><b>4</b>:[0,4)</td> <td colspan=4><b>5</b>:[4,8)</td> <td colspan=4><b>6</b>:[8,12)</td> <td colspan=4><b>7</b>:[12,16)</td> </tr>
<tr> <td colspan=2><b>8</b>:[0,2)</td> <td colspan=2><b>9</b>:[2,4)</td> <td colspan=2><b>10</b>:[4,6)</td> <td colspan=2><b>11</b>:[6,8)</td> <td colspan=2><b>12</b>:[8,10)</td> <td colspan=2><b>13</b>:[10,12)</td> <td colspan=2><b>14</b>:[12,14)</td> <td colspan=2><b>15</b>:[14,16)</td> </tr>
<tr> <td><b>16</b>:0</td> <td><b>17</b>:1</td> <td><b>18</b>:2</td> <td><b>19</b>:3</td> <td><b>20</b>:4</td> <td><b>21</b>:5</td> <td><b>22</b>:6</td> <td><b>23</b>:7</td> <td><b>24</b>:8</td> <td><b>25</b>:9</td> <td><b>26</b>:10</td> <td><b>27</b>:11</td> <td><b>28</b>:12</td> <td><b>29</b>:13</td> <td><b>30</b>:14</td> <td><b>31</b>:15</td> </tr>
</table>

<!--
# The table is generated by ./dot.py 16
def main(n, a, fill):
    print('<table class="segment-tree">')
    j, k = 1, 1
    while j < len(a):
        print('<tr>', end='')
        for i, x in enumerate(a[j:j+k], start=j):
            td = f' colspan={n//k}' if n != k else ''
            if fill[i]: td += ' style="background:#fdd"'
            print(f' <td{td}>{x}</td>', end='')
        print(' </tr>')
        j, k = j+k, k*2
    print('</table>')

import sys
n = int(sys.argv[1])
nn = n
while nn & nn-1: nn += nn & -nn
txt, fill = ['']*(nn*2), [False]*(nn*2)

def build(i, l, r):
    if l == r-1:
        txt[i] = f'<b>{i}</b>:{l}'
    else:
        m = l+r >> 1; build(2*i, l, m); build(2*i+1, m, r)
        txt[i] = f'<b>{i}</b>:{f"[{l},{r})"}'

build(1, 0, n)
for x in sys.argv[2:]:
    i = n+int(x)
    while i > 1:
        i >>= 1; fill[i] = True
main(nn, txt, fill)
-->

When n is not a power of two, this representation can still be used, but note that some indexes will be unused beside index 0.
If we set the middle point to `floor((l+r)/2)`, the approach uses no less space than tail padding.
For the visual representation below for n=7, indexes 8 and 9 are unused.

<table class="segment-tree">
<tr> <td colspan=8><b>1</b>:[0,7)</td> </tr>
<tr> <td colspan=4><b>2</b>:[0,3)</td> <td colspan=4><b>3</b>:[3,7)</td> </tr>
<tr> <td colspan=2><b>4</b>:0</td> <td colspan=2><b>5</b>:[1,3)</td> <td colspan=2><b>6</b>:[3,5)</td> <td colspan=2><b>7</b>:[5,7)</td> </tr>
<tr> <td></td> <td></td> <td><b>10</b>:1</td> <td><b>11</b>:2</td> <td><b>12</b>:3</td> <td><b>13</b>:4</td> <td><b>14</b>:5</td> <td><b>15</b>:6</td> </tr>
</table>
<!--
# The table is generated by ./dot.py 7
-->

```cpp
void build(int i, int l, int r) {
  if (l == r-1)
    minv[i] = a[i];
  else {
    int m = l+r >> 1;
    tag[i] = 0;
    build(2*i, l, m);
    build(2*i+1, m, r);
    minv[i] = min(minv[2*i], minv[2*i+1]);
  }
}

void add(int i, int l, int r, int ll, int rr, long v) {
  if (l <= ll && rr <= r)
    return apply(i, v);
  untag(i, l, r);
  int m = l+r >> 1;
  if (ll < m) add(2*i, l, m, ll, rr, v);
  if (m < rr) add(2*i+1, m, r, ll, rr, v);
  mconcat(i, l, r);
}

long get_min(int i, int l, int r, int ll, int rr) {
  if (l <= ll && rr <= r)
    return minv[i];
  untag(i, l, r);
  int m = l+r >> 1;
  long ans = LONG_MAX;
  if (ll < m) ans = get_min(2*i, l, m, ll, rr);
  if (m < rr) ans = min(ans, get_min(2*i+1, m, r, ll, rr));
  return ans;
}

int main() {
  ...
  add(1, 0, n, l, r, v);
  get_min(1, 0, n, l, r);
  ...
}
```

If we set the middle point to `floor((l+r+1)/2)`, we can use less space.
The exact space is not easy to compute but the last index increases very fast when n increases from `2^k+1` to `2^(k+1)`.
So it is not worth the trouble writing `l+r+1 >> 1`.

```text
1:[0,7)
2:[0,4) 3:[4,7)
4:[0,2) 5:[2,4) 6:[4,6) 7:[6,7)
8:[0,1) 9:[1,2) 10:[2,3) 11:[3,4) 12:[4,5) 13:[5,6)
```

There is another variant when n is not a power of 2, which will be detailed below when introducing the bottom-up open interval implementation.

#### l+r (in-order layout)

This is an alternative representation for a full binary tree.
We can compute the node index by adding its left and right boundaries (in-order traversal indexing).

<table class="segment-tree">
<tr> <td colspan=8><b>8</b>:[0,8)</td> </tr>
<tr> <td colspan=4><b>4</b>:[0,4)</td> <td colspan=4><b>12</b>:[4,8)</td> </tr>
<tr> <td colspan=2><b>2</b>:[0,2)</td> <td colspan=2><b>6</b>:[2,4)</td> <td colspan=2><b>10</b>:[4,6)</td> <td colspan=2><b>14</b>:[6,8)</td> </tr>
<tr> <td><b>1</b>:0</td> <td><b>3</b>:1</td> <td><b>5</b>:2</td> <td><b>7</b>:3</td> <td><b>9</b>:4</td> <td><b>11</b>:5</td> <td><b>13</b>:6</td> <td><b>15</b>:7</td> </tr>
</table>

<!--
# The table is generated by ./dotl+r.py 8
def main(n, a):
    print('<table class="segment-tree">')
    k = 1; a = a[1:]
    while len(a):
        print('<tr>', end='')
        for x in a[:k]:
            css = ''
            if n != k: css = f' colspan={n//k}'
            print(f' <td{css}>{x}</td>', end='')
        print(' </tr>')
        a = a[k:]; k <<= 1
    print('</table>')

import sys
n = int(sys.argv[1])
txt = ['']*(n*2)

def build(i, l, r):
    if l == r-1:
        txt[i] = f'<b>{l+r}</b>:{l}'
    else:
        m = l+r >> 1; build(2*i, l, m); build(2*i+1, m, r)
        txt[i] = f'<b>{l+r}</b>:{str(l) if l==r-1 else f"[{l},{r})"}'

build(1, 0, n)
main(n, txt)
-->

```cpp
void build(int l, int r) {
  if (l == r-1)
    maxv[l+r] = a[l];
  else {
    int m = l+r >> 1;
    tag[i] = 0;
    build(l, m);
    build(m, r);
    maxv[l+r] = max(maxv[l+m], maxh[m+r]);
  }
}
```

This simple rule works because no two node share the same `l+r`.

However, if n is not a power of two, we may have a non-leaf node whose `l+r` is odd, say, `0+5`.
The leftmost node of its right subtree has the same index: `(l+r-1)/2 + (l+r+1)/2`.
Note that such a collision only happens with a leaf node and an internal node.

A naive approach avoiding collision is to special case the index computation for leaf nodes.
The space consumption is `3*n`. There are some unused indexes, though.

```cpp
int id(int l, int r) {
  return r-l == 1 ? 2*n+l : l+r;
}

void build(int l, int r) {
  if (l == r-1)
    minv[i] = a[i];
  else {
    int m = l+r >> 1, i = id(l, r);
    tag[i] = 0;
    build(l, m);
    build(m, r);
    minv[i] = min(minv[id(l, m)], minv[id(m, r)]);
  }
}
```

A better encoding costs 2*n space.

```cpp
int id(int l, int r) {
  return l+r & ~(l != r-1);
}

// right leaning
void build(int l, int r) {
  if (l == r-1)
    minv[i] = a[i];
  else {
    int m = l+r >> 1, i = id(l, r);
    tag[i] = 0;
    build(l, m);
    build(m, r);
    minv[i] = min(minv[id(l, m)], minv[id(m, r)]);
  }
}
```

Alternatively, if we make the tree left leaning, we can use:

```cpp
int id(int l, int r) {
  return l+r-1 | (l != r-1);
}

// left leaning
void build(int l, int r) {
  if (l == r-1)
    minv[i] = a[i];
  else {
    int m = l+r+1 >> 1, i = id(l, r);
    tag[i] = 0;
    build(l, m);
    build(m, r);
    minv[i] = min(minv[id(l, m)], minv[id(m, r)]);
  }
}
```

Anycase, the `l+r` based numbering schemes are not cache friendly.
For the binary heap representation, the indexes on one level are contiguous.
We can imagine that the top few levels are shared and they need fewer blocks.

## Bottom-up representations

### Bottom-up, open interval (recommended)

To the best of my knowledge, it was first discovered by 张昆玮 in 2010 (《统计的力量——线段树全接触》).
张昆玮's representation uses open intervals. I used to find it inconvenient in two places:

* the implementation needs two sentinel elements 0 and n-1 (n-2 >= original length).
* the implementation uses open intervals. I prefer half-open intervals.

However, the sentinel elements are not really needed and open intervals turn out to be an advantage.
Read on.

The segment tree maintains the an array of length n where n is not necessarily a power of two.
0 and n-1 are not necessarily sentinel elements.
(In 张昆玮's code, 0 and n-1 are sentinel elements. There may be other padding elements at the end.)

For a modification or query `[l,r)`, we have `0<=l && r<=n`.
(In 张昆玮's code, we have `0<l && r<n` because 0 and n-1 are sentinel elements and thus cannot be included in modifications or queries.)

Let's read a full program.
The program reads n integers, then either computes `a[l]+a[l+1]+...+a[r-1]` or adds an integer to `a[l],a[l+1],...,a[r-1]`.

```cpp
// The program can be easily adapted to other modifications and queries.
// For range add and range sum specifically, read Commutative modifications and query monoid below for a faster implementation.
#include <algorithm>
#include <cmath>
#include <cstdio>
#include <type_traits>
using namespace std;

#define FOR(i, a, b) for (remove_cv<remove_reference<decltype(b)>::type>::type i = (a); i < (b); i++)
#define REP(i, n) FOR(i, 0, n)
#define ROF(i, a, b) for (remove_cv<remove_reference<decltype(b)>::type>::type i = (b); --i >= (a); )

const int N = 200000;
long sum[N*2], tag[N*2];
int num[N*2], n, ln;

void apply(int i, long v) {
  // Update the monoid sum and the lazy tag stored at node i.
  sum[i] += num[i]*v;
  tag[i] += v;
}

void untag(int i) {
  // ln = ceil(log2(n))
  // No need to test j!=0 if n is a power of 2 or tag[0] guarantees to be valid and empty.
  for (int j, h = ln; h; h--)
    if (j = i>>h, j && tag[j]) {
      apply(2*j, tag[j]);
      apply(2*j+1, tag[j]);
      tag[j] = 0;
    }
}

void mconcat(int l) {
  // Aggregate 2*i and 2*i+1 into i.
  sum[l>>1] = sum[l] + sum[l^1];
}

void add(int l, int r, long v) {
  l += n-1, r += n;
  // (l,r) is an open interval now.
  // Propagate lazy tags.
  untag(l+1);
  untag(r-1);
  bool lf = false, rf = false;
  for (; l^r^1; l >>= 1, r >>= 1) {
    if (~l&1) apply(l^1, v), lf = true;
    if (r&1) apply(r^1, v), rf = true;
    // If n is a power of 2, no need to test l > 1; otherwise l may be 1, and 2<=r<=4
    if (lf && l > 1) mconcat(l);
    if (rf) mconcat(r);
  }
  for (; l > 1; l >>= 1)
    mconcat(l);
}

long get_sum(int l, int r) {
  l += n-1, r += n;
  // (l,r) is an open interval now.
  // Propagate lazy tags.
  untag(l+1);
  untag(r-1);
  long ls = 0, rs = 0;
  for (; l^r^1; l >>= 1, r >>= 1) {
    if (~l&1) ls += sum[l^1];
    // If the binary operation is not commutative, note the order of sum[r^1] and rs.
    if (r&1) rs = sum[r^1]+rs;
  }
  return ls+rs;
}

int main() {
  int l, r, v;
  char c;
  scanf("%d", &n);
  // ln = ceil(log2(n))
  ln = n > 1 ? 31-__builtin_clz(n-1)+1 : 0;
  REP(i, n)
    scanf("%ld", &sum[n+i]);
  fill_n(num+n, n, 1);
  fill_n(tag, 2*n, 0);
  ROF(i, 1, n) {
    num[i] = num[2*i]+num[2*i+1];
    mconcat(2*i);
  }
  while (scanf("%d%d%c", &l, &r, &c) == 3) {
    // [l, r)
    if (c == '\n')
      printf("%ld\n", get_sum(l, r));
    else {
      scanf("%d", &v);
      add(l, r, v);
    }
  }
}
```

(
For completeness, here is the function for single-element updates.
```cpp
void add(int i, long v) {
  i += n;
  untag(i);
  for (; i > 1; i >>= 1)
    mconcat(i);
}
```
)

After `l += n-1, r += n`, l and r have changed semantics from array indexes to leaf node indexes.
They now represent an open interval of node indexes. The two initial node indexes are not on the same level if:

* if n is a power of two, `l==0 || r==n-1`
* if n is not a power of two, the first 2**ceil(log2(n))-n elements are one level above the rest elements.

(If we use 张昆玮's representation, the two initial nodes are guaranteed to be on the same level.)

In particular, when n is not a power of two, not all leaf nodes (which store array elements) are on the same level.
Say n=13, while `a[3]~a[12]` are stored in leaf nodes 16~25, `a[0]~a[2]` are stored in nodes 13~15 which are one level above.
Indexes 26~31 are unused and not considered nodes. The initial r may be the invalid 26 but the program will not access it because 26 is even.

<table class="segment-tree">
<tr> <td colspan=16><b>1</b>:[3,13),[0,3)</td> </tr>
<tr> <td colspan=8><b>2</b>:[3,11)</td> <td colspan=8><b>3</b>:[11,13),[0,3)</td> </tr>
<tr> <td colspan=4><b>4</b>:[3,7)</td> <td colspan=4><b>5</b>:[7,11)</td> <td colspan=4><b>6</b>:[11,13),0</td> <td colspan=4><b>7</b>:[1,3)</td> </tr>
<tr> <td colspan=2><b>8</b>:[3,5)</td> <td colspan=2><b>9</b>:[5,7)</td> <td colspan=2><b>10</b>:[7,9)</td> <td colspan=2><b>11</b>:[9,11)</td> <td colspan=2><b>12</b>:[11,13)</td> <td colspan=2><b>13</b>:0</td> <td colspan=2><b>14</b>:1</td> <td colspan=2><b>15</b>:2</td> </tr>
<tr> <td><b>16</b>:3</td> <td><b>17</b>:4</td> <td><b>18</b>:5</td> <td><b>19</b>:6</td> <td><b>20</b>:7</td> <td><b>21</b>:8</td> <td><b>22</b>:9</td> <td><b>23</b>:10</td> <td><b>24</b>:11</td> <td><b>25</b>:12</td> <td></td> <td></td> <td></td> <td></td> <td></td> <td></td> </tr>
</table>

<!--
# The table is generated by
def main(n, a):
    assert(len(a) == 2*n)
    print('<table class="segment-tree">')
    k = 1
    a = a[1:]
    while len(a):
        print('<tr>', end='')
        for x in a[:k]:
            css = ''
            if n != k:
                css = f' colspan={n//k}'
            print(f' <td{css}>{x}</td>', end='')
        print(' </tr>')
        a = a[k:]
        k <<= 1
    print('</table>')

n = 13
nn = 16
vs = [[]]*(nn*2)
txt = ['']*(nn*2)
for i in range(n):
    vs[n+i] = [(i,i+1)]
    txt[n+i] = f'<b>{n+i}</b>:{i}'
for i in range(n-1,0,-1):
    ls, rs = vs[2*i], vs[2*i+1]
    if ls[-1][1] == rs[0][0]:
        vs[i] = ls[:-1] + [(ls[-1][0], rs[0][1])] + rs[1:]
    else:
        vs[i] = ls+rs
    txt[i] = f'<b>{i}</b>:{",".join(str(x[0]) if x[0]==x[1]-1 else f"[{x[0]},{x[1]})" for x in vs[i])}'
main(nn, txt)
-->

Let's discuss the position of the two initial nodes and how they affect the final value of l and r when the loop condition `l^r^1` evaluates to false:

* If initial l and r are in subtree 2, we have `l>=4` after the main loop.
* If initial l and r are in subtree 3, we have `l>=6` after the main loop..
* If initial l is in subtree 2 and initial r is in subtree 3, we have `l==2 && r==3` after the main loop.
* (Only if the two initial nodes are on different levels) If initial l is in subtree 3 and initial r is in subtree 2, we have `l==0 && r==1` after the main loop. In a previous iteration, we have `l==1 && 2<=r && r<=4`. (r=4 can only happen when n is a power of two and the initial interval is the full interval.)

From the last point, we can see that if initial nodes are not on the same level, there are some subtleties. Fortunately the program can correctly handle such cases without extra code.

Let's analyze some concrete examples when the initial nodes are not on the same level:

Say n=8 and the query is `[0,8)`.
We have l=7 and r=16.
The first iteration in the main loop does not update any node. l becomes 3 and r becomes 8.
The second iteration does not update any node. l becomes 1 and r becomes 4.
The third iteration does not update any node. l becomes 0 and r becomes 2.
The fourth (last) iteration updates node 1. l is still 0 and r becomes 1.
The loop condition evaluates to false.

Let's look at another example `[0,7)`.
We have l=7 and r=15.
The first iteration updates node 14. l becomes 3 and r becomes 7.
The second iteration updates node 6. l becomes 1 and r becomes 3.
The third (last) iteration updates node 2. l becomes 0 and r becomes 1.
The loop condition evaluates to false.

When adding an integer to a subtree, we need to know the number of elements in the subtree to compute its influence on the monoid sum.
If n is a power of two, we can derive the number of elements in a subtree easily.
In the program, we introduce the array `num[]`. We initialize nodes `[n,2*n)` to have 1 element, and propagate up the numbers.

For many other modifications, we don't need the number of elements in a subtree.
When n is not a power of two, there is no extra bookkeeping.

#### What does the node at index one represent?

When n is a power of two, the node at index one (root) stores the monoid sum of the whole array.
Top-down traversals are straightforward.
For example, if the array has 0 and 1 elements and we try to find the kth 1 (the select operation in an order statistic tree).
We can start from node 1 (it represents the whole array) and walk down from it.
If subtree 2 (which corresponds to elements 0 to n/2-1) has more than k ones, we go to subtree 2, otherwise subtree 3.
Keep this process until we reach a leaf.
```cpp
// n is a power of two.
int i = 1;
while (i < n) {
  i <<= 1;
  if (k >= cnt[i])
    k -= cnt[i];
}
return i-n;  // array index
```

When n is not a power of two, the node at index one stores the monoid sum of a permutation of the whole array.
If the query operation is commutative (e.g. sum/min/max), we can still query node 1;
otherwise we need to visit multiple full binary subtrees with increasing array indexes, and compute their monoid sum.
Subtree 2 no longer aggregates elements 0 to n/2-1 and subtree 3 no longer aggregate elements n/2 to n-1.
The above top-down walk code will be incorrect.

As a rule of thumb, pad the original array if you need top-down walks or the monoid sum of the original array in the node at index 1.

### Bottom-up, half-open interval

In 2011, Dmitry Urbanovich wrote <https://codeforces.com/blog/entry/1256> and introduced an implementation dealing with closed intervals.
There is a great write-up <https://codeforces.com/blog/entry/18051> by Oleksandr Bacherikov using half-open intervals.

```cpp
// Different from the open interval implementation.
void mconcat(int i, int k) {
  // Aggregate 2*i (length k/2) and 2*i+1 (length k/2) into i (length k).
}

void modify(int l, int r, int v) {
  untag(l += n); // untag accepts a node index
  untag((r+=n) - 1);
  bool lf = false, rf = false;
  int k = 1;
  for (k = 1; l < r; l >>= 1, r >>= 1, k <<= 1) {
    if (lf) mconcat(l-1, k);
    if (rf) mconcat(r, k);
    if (l&1) apply(l++, v), lf = true;
    if (r&1) apply(--r, v), rf = true;
  }
  for (l--; r; l >>= 1, r >>= 1, k <<= 1) {
    if (lf) mconcat(l, k);
    if (rf && (!lf || l != r)) mconcat(r, k);
  }
}
```

For a while, I used the following style which is similar to Oleksandr's.
```cpp
void modify(int l, int r, int v) {
  untag(l-1); // untag accepts an array index
  untag(r);
  bool lf = false, rf = false;
  for (l += NN, r += NN; l < r; ) {
    if (l & 1) lf = true, apply(l++, v);
    l >>= 1;
    if (lf) mconcat(l-1);
    if (r & 1) rf = true, apply(--r, v);
    r >>= 1;
    if (rf) mconcat(r);
  }
  for (l--; l >>= 1, r >>= 1; ) {
    if (lf || l == r) mconcat(l);
    if (rf && l != r) mconcat(r);
  }
}
```

I hope you have noticed the pain point now. `mconcat(l-1, k)` in the main loop is a bit unnatural.
The bottom-up propagation (`l--`) in the end is particularly error-prone.
Two `mconcat` calls instead of one are needed due to a case which cannot be handled elegantly by the loop condition `l < r`.
(If you convert the open interval code to the half-closed interval style, you shall note that the problem can be fixed by using `(l-1)^r^1`, but then we lose elegance.)

Say n=4, we need to update [1,3). We have l=5 and r=7.
After applying to node 5 and node 6, we get l=r=3.
`l--` is to ensure node 2 is properly updated.

If we use open intervals, we have l=4 and r=7 initially.
After applying to node 5 and node 6, we get l=2 and r=3.
Even if `r-l == 1`, `(l^r^1) != 0`! This means with open intervals and the loop condition `l^r^1`, the main loop gets executed one more time which ensures node 2 and node 3 are updated.
This is a big difference.
In addition, with open intervals, we can compute the parent node index with `l>>1` instead of `l = (l+1)>>1`.

The main observation of Oleksandr Bacherikov is that n does not need to be a power of two.

## Lazy propagation

In both top-down and bottom-up implementations, we call `untag`. Now let's figure out why `untag` is needed.

Say n=16, the query range is `[1,13)`. The program will visit the green nodes.

<table class="segment-tree">
<tr> <td colspan=16><b>1</b>:[0,16)</td> </tr>
<tr> <td colspan=8><b>2</b>:[0,8)</td> <td colspan=8><b>3</b>:[8,16)</td> </tr>
<tr> <td colspan=4><b>4</b>:[0,4)</td> <td colspan=4 style="background:#dfd"><b>5</b>:[4,8)</td> <td colspan=4 style="background:#dfd"><b>6</b>:[8,12)</td> <td colspan=4><b>7</b>:[12,16)</td> </tr>
<tr> <td colspan=2><b>8</b>:[0,2)</td> <td colspan=2 style="background:#dfd"><b>9</b>:[2,4)</td> <td colspan=2><b>10</b>:[4,6)</td> <td colspan=2><b>11</b>:[6,8)</td> <td colspan=2><b>12</b>:[8,10)</td> <td colspan=2><b>13</b>:[10,12)</td> <td colspan=2><b>14</b>:[12,14)</td> <td colspan=2><b>15</b>:[14,16)</td> </tr>
<tr> <td><b>16</b>:0</td> <td style="background:#dfd"><b>17</b>:1</td> <td><b>18</b>:2</td> <td><b>19</b>:3</td> <td><b>20</b>:4</td> <td><b>21</b>:5</td> <td><b>22</b>:6</td> <td><b>23</b>:7</td> <td><b>24</b>:8</td> <td><b>25</b>:9</td> <td><b>26</b>:10</td> <td><b>27</b>:11</td> <td style="background:#dfd"><b>28</b>:12</td> <td><b>29</b>:13</td> <td><b>30</b>:14</td> <td><b>31</b>:15</td> </tr>
</table>
<!-- generated by ./query.py 16 1 13
def main(n, a, fill):
    print('<table class="segment-tree">')
    j, k = 1, 1
    while j < len(a):
        print('<tr>', end='')
        for i, x in enumerate(a[j:j+k], start=j):
            td = f' colspan={n//k}' if n != k else ''
            if fill[i]: td += ' style="background:#dfd"'
            print(f' <td{td}>{x}</td>', end='')
        print(' </tr>')
        j, k = j+k, k*2
    print('</table>')

import sys
n = int(sys.argv[1])
nn = n
while nn & nn-1: nn += nn & -nn
txt, fill = ['']*(nn*2), [False]*(nn*2)

def build(i, l, r):
    if l == r-1:
        txt[i] = f'<b>{i}</b>:{l}'
    else:
        m = l+r >> 1; build(2*i, l, m); build(2*i+1, m, r)
        txt[i] = f'<b>{i}</b>:{f"[{l},{r})"}'

build(1, 0, n)
l, r = int(sys.argv[2]), int(sys.argv[3])
l, r = l+n-1, r+n
while l^r^1:
    if l%2 == 0: fill[l^1] = True
    if r%2: fill[r^1] = True
    l, r = l>>1, r>>1
main(nn, txt, fill)
-->

However, if there were previously modifications to the ancestors of these nodes.
The to-be-visited nodes may not have up-to-date monoid sums.
There is a generic lazy propagation approach and a simpler approach for commutative modifications.
Let's consider the generic approach first.

We need to call `untag` on the leftmost and rightmost element in the interval to ensure the monoid sums are up-to-date.

untag(1) visits every ancestor of n+1 in a top-down manner, propagating modifications downwards.
<table class="segment-tree">
<tr> <td colspan=16 style="background:#fdd"><b>1</b>:[0,16)</td> </tr>
<tr> <td colspan=8 style="background:#fdd"><b>2</b>:[0,8)</td> <td colspan=8><b>3</b>:[8,16)</td> </tr>
<tr> <td colspan=4 style="background:#fdd"><b>4</b>:[0,4)</td> <td colspan=4><b>5</b>:[4,8)</td> <td colspan=4><b>6</b>:[8,12)</td> <td colspan=4><b>7</b>:[12,16)</td> </tr>
<tr> <td colspan=2 style="background:#fdd"><b>8</b>:[0,2)</td> <td colspan=2><b>9</b>:[2,4)</td> <td colspan=2><b>10</b>:[4,6)</td> <td colspan=2><b>11</b>:[6,8)</td> <td colspan=2><b>12</b>:[8,10)</td> <td colspan=2><b>13</b>:[10,12)</td> <td colspan=2><b>14</b>:[12,14)</td> <td colspan=2><b>15</b>:[14,16)</td> </tr>
<tr> <td><b>16</b>:0</td> <td><b>17</b>:1</td> <td><b>18</b>:2</td> <td><b>19</b>:3</td> <td><b>20</b>:4</td> <td><b>21</b>:5</td> <td><b>22</b>:6</td> <td><b>23</b>:7</td> <td><b>24</b>:8</td> <td><b>25</b>:9</td> <td><b>26</b>:10</td> <td><b>27</b>:11</td> <td><b>28</b>:12</td> <td><b>29</b>:13</td> <td><b>30</b>:14</td> <td><b>31</b>:15</td> </tr>
</table>

untag(12) visits every ancestor of n+12 in a top-down manner, propagating modifications downwards.
<table class="segment-tree">
<tr> <td colspan=16 style="background:#fdd"><b>1</b>:[0,16)</td> </tr>
<tr> <td colspan=8><b>2</b>:[0,8)</td> <td colspan=8 style="background:#fdd"><b>3</b>:[8,16)</td> </tr>
<tr> <td colspan=4><b>4</b>:[0,4)</td> <td colspan=4><b>5</b>:[4,8)</td> <td colspan=4><b>6</b>:[8,12)</td> <td colspan=4 style="background:#fdd"><b>7</b>:[12,16)</td> </tr>
<tr> <td colspan=2><b>8</b>:[0,2)</td> <td colspan=2><b>9</b>:[2,4)</td> <td colspan=2><b>10</b>:[4,6)</td> <td colspan=2><b>11</b>:[6,8)</td> <td colspan=2><b>12</b>:[8,10)</td> <td colspan=2><b>13</b>:[10,12)</td> <td colspan=2 style="background:#fdd"><b>14</b>:[12,14)</td> <td colspan=2><b>15</b>:[14,16)</td> </tr>
<tr> <td><b>16</b>:0</td> <td><b>17</b>:1</td> <td><b>18</b>:2</td> <td><b>19</b>:3</td> <td><b>20</b>:4</td> <td><b>21</b>:5</td> <td><b>22</b>:6</td> <td><b>23</b>:7</td> <td><b>24</b>:8</td> <td><b>25</b>:9</td> <td><b>26</b>:10</td> <td><b>27</b>:11</td> <td><b>28</b>:12</td> <td><b>29</b>:13</td> <td><b>30</b>:14</td> <td><b>31</b>:15</td> </tr>
</table>


Say we have applied `f` on `a[1]`, next `g` on `a[0,2)`, then `h` on `a[0,4)`, finally `m` on `a[0,8)`.
The final value of `a[0]` is therefore `m(h(g(f(a[0]))))`.
Note that the outer functions apply to larger intervals.

```text
1:[0,16) identity
2:[0,8) m
4:[0,4) h
8:[0,2) g
17:[1,2) f
```

Say we want to apply another operation `f'` to `a[0]`.
If the operations commute, i.e. `f'(m(h(g(f(a[0]))))) = m(h(g(f'(f(a[0])))))`, we may ignore `m`, `h` and `g` on ancestor nodes and apply `f'` directly to `a[0]`, essentially composing `f'` and `f`.

```text
1:[0,16) identity
2:[0,8) m
4:[0,4) h
8:[0,2) g
17:[1,2) f' . f
```

Then the next `a[0]` retrieval query will return `m(h(g(f'(f(a[0])))))`.
The classical example of this class is the add operation. See the next section.

However, if modification operations are not commutative, we cannot ignore `m`, `h` and `g`.
We have to propagate operations downwards. This process is usually called lazy propagation.
When the operations stores in ancestor nodes are all identity, we can compose `f` and `f'`, because an identity commutes with any operation.

```text
1:[0,16) identity
2:[0,8) identity
4:[0,4) identity
8:[0,2) identity
17:[1,2) f' . f
```

## Commutative modifications and query monoid

If both modifications and the query monoid are commutative, we can apply modifications in a bottom-up manner.
We can thus omit `untag` calls.
The query function needs an update. Unlike the `untag` implementation, after `l^r^1` evaluates to false, we should keep bubbling up to add in modifications to ancestor nodes.

Here is an example of range add and range sum.
```cpp
#include <algorithm>
#include <cstdio>
#include <type_traits>
using namespace std;

#define FOR(i, a, b) for (remove_cv<remove_reference<decltype(b)>::type>::type i = (a); i < (b); i++)
#define REP(i, n) FOR(i, 0, n)
#define ROF(i, a, b) for (remove_cv<remove_reference<decltype(b)>::type>::type i = (b); --i >= (a); )

const int N = 200000;
long sum[N*2], tag[N*2];
int n;

void add(int l, int r, long v) {
  l += n-1, r += n;
  long lc = 0, rc = 0;
  for (int k = 1; l^r^1; l >>= 1, r >>= 1, k <<= 1) {
    if (~l&1)
      sum[l^1] += v*k, tag[l^1] += v, lc += k;
    if (r&1)
      sum[r^1] += v*k, tag[r^1] += v, rc += k;
    sum[l>>1] += v*lc, sum[r>>1] += v*rc;
  }
  while (l > 1) {
    sum[l>>=1] += v*lc;
    sum[r>>=1] += v*rc;
  }
}

long get_sum(int l, int r) {
  l += n-1, r += n;
  long ans = 0, lc = 0, rc = 0;
  for (int k = 1; l^r^1; l >>= 1, r >>= 1, k <<= 1) {
    if (~l&1)
      ans += sum[l^1], lc += k;
    if (r&1)
      ans += sum[r^1], rc += k;
    ans += tag[l>>1]*lc + tag[r>>1]*rc;
  }
  while (l > 1)
    ans += tag[l>>=1]*lc + tag[r>>=1]*rc;
  return ans;
}

int main() {
  int l, r, v;
  char c;
  scanf("%d", &n);
  REP(i, n)
    scanf("%ld", &sum[n+i]);
  ROF(i, 1, n)
    sum[i] = sum[2*i]+sum[2*i+1];
  fill_n(tag, 2*n, 0);
  while (scanf("%d%d%c", &l, &r, &c) == 3) {
    // [l, r)
    if (c == '\n')
      printf("%ld\n", get_sum(l, r));
    else {
      scanf("%d", &v);
      add(l, r, v);
    }
  }
}
```

## Combining monoid sum and lazy tag

This further optimizes based on commutative modifications. 《统计的力量——线段树全接触》 names this 标记永久化.

For range add and range min/max, we can store one value instead of two in an inner node.
Without loss of generality, let's say the query type is range min.
The value is defined as `min_value(i)-min_value(parent(i))`.

```cpp
void mconcat(int l) {
  auto mn = min(a[l], a[l^1]);
  a[l] -= mn, a[l^1] -= mn, a[l>>1] += mn;
}

void add(int l, int r, long v) {
  bool lf = false, rf = false;
  for (l += n-1, r += n; l^r^1; l >>= 1, r >>= 1) {
    if (~l&1) a[l^1] += v, lf = true;
    if (r&1) a[r^1] += v, rf = true;
    if (lf && l > 1) mconcat(l);
    if (rf) mconcat(r);
  }
  for (; l > 1; l >>= 1)
    mconcat(l);
}

long get_min(int l, int r) {
  long ls = LONG_MAX, rs = LONG_MAX;
  for (l += n-1, r += n; l^r^1; l >>= 1, r >>= 1) {
    if (~l&1) ls = min(ls, a[l^1]);
    if (r&1) rs = min(rs, a[r^1]);
    if (ls != LONG_MAX) ls += a[l>>1];
    if (rs != LONG_MAX) rs += a[r>>1];
  }
  long ans = min(ls, rs);
  while (l > 1)
    ans += a[l >>= 1];
  return ans;
}

int main() {
  // n is not necessarily a power of two.
  read(n);
  REP(i, n)
    read(a[n+i]);
  for (int i = n; --i; )
    mconcat(2*i);
  // ...
}
```

## Relations with other data structures

### Fenwick tree

If the binary operation used by range queries supports subtraction, we can answer a `[l,r)` query with the difference of two prefix sums `sum[0,r) - sum[0,l)`, and therefore switch to a Fenwick tree (binary indexed tree).
Unlike a segment tree (`2n-1` nodes), a Fenwick tree needs exactly n elements.

### Binary search tree

A segment tree maintains a static array.
If the array is dynamic (insertion/deletion), we can switch to a binary search tree which encodes the array in its in-order traversal.
For range queries, a Splay tree and a Treap (with join-based tree algorithms) are common.

A binary search tree needs n nodes, but larger space consumption encoding the left and right children, and potential extra space ensuring self balancing.

## Application

* Maintain an array with value updates
* Change the role of values to indexes, maintain n (persistent) segment trees with frequency as value.
* Euler-tour technique
* ...
