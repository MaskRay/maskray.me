---
layout: post
title: Dominator tree
author: MaskRay
tags: [compiler]
mathjax: true
---

## Lengauer-Tarjan algorithm

<!-- more -->

Number the vertices by their pre-order (DFS) number and identify each vertex with that number.
A path $P=(u=v_0,v_1,\ldots,v_{k-1},v_k=v)$ in _G_ is a semidominator path if every intermediate vertex has a larger number than the endpoint, i.e. $v_i>v$ for $1\leq i<k$ (the two endpoints $u$ and $v$ are unconstrained).
The semidominator of _v_ is the smallest vertex that can reach _v_ through such a path:

$$sdom(v) = \text{min}\{u | \text{there is a sdom path from } u \text{ to } v\}$$

We compute `sdom[*]` using the reverse pre-order to utilize already-computed `sdom[*]` of larger indices.
For each vertex `v`, enumerate its predecessors `u` and consider an optimal semidominator path into `v`, ending in edge `(u, v)`:

- `u < v`: Contributes candidate `u` (in the following code, `sdom[u] = u` at this moment); `u` cannot be an interior vertex of another semidominator path to `v`.
- `u > v`: Let `w` be the interior vertex with the smallest number, so the infix `w -> ... -> u` has every vertex `>= w > v`.
  At the moment the pre-order DFS discovered `w`, the rest of this infix was unvisited (white), since every such vertex has a larger number; by the white-path theorem `w` is thus a tree-ancestor of `u`.
  The path splits into a semidominator path into `w`, then `w -> ... -> u`, then the edge `u -> v`, so its contribution is `sdom(w)`.
  As `w` ranges over the tree-ancestors of `u` above `v`, this predecessor's best candidate is the minimum `sdom` among them — exactly what `eval` computes.

With a simple implementation of eval-link, the time complexity is $O(E\log V)$.

`eval(v, cur)` walks up the ancestor path of `v` in the DFS spanning tree and returns, among the ancestors with `dfn > cur` (those already linked in this step), the one with the minimum `sdom`.
`uf[]` doubles as the union-find parent array (merged with `parent[]`), and `best[v]` carries the running minimum along the path.

```cpp
#include <algorithm>
#include <cstdio>
using namespace std;

const int N = 100000, M = 500000;
struct Arc { int v, next; } pool[2*M+N], *pit;
int e[N], ee[N], domch[N], tick, dfn[N], rdfn[N], uf[N], sdom[N], best[N], idom[N];

void dfs(int u) {
  dfn[u] = tick;
  rdfn[tick++] = u;
  for (int v, a = e[u]; ~a; a = pool[a].next)
    if (dfn[v = pool[a].v] < 0) {
      uf[v] = u;
      dfs(v);
    }
}

int eval(int v, int cur) {
  if (dfn[v] <= cur)
    return v;
  int u = uf[v], r = eval(u, cur);
  if (dfn[sdom[best[u]]] < dfn[sdom[best[v]]])
    best[v] = best[u];
  return uf[v] = r;
}

void simpleLengauerTarjan(int n, int r) {
  fill_n(dfn, n, -1);
  tick = 0;
  dfs(r);
  for (int i = 0; i < n; i++)
    sdom[i] = best[i] = i;
  for (int i = tick; --i; ) {
    int v = rdfn[i], u;
    for (int a = ee[v]; ~a; a = pool[a].next)
      if (dfn[u = pool[a].v] != -1) {
        eval(u, i);
        if (dfn[sdom[best[u]]] < dfn[sdom[v]])
          sdom[v] = sdom[best[u]];
      }
    *pit = {v, domch[sdom[v]]};
    domch[sdom[v]] = pit++-pool;
    v = rdfn[i-1];
    for (int a = domch[v]; ~a; a = pool[a].next) {
      u = pool[a].v;
      eval(u, i-1);
      idom[u] = sdom[best[u]] == v ? v : best[u];
    }
  }
  for (int i = 1; i < tick; i++) {
    int v = rdfn[i];
    if (idom[v] != sdom[v])
      idom[v] = idom[idom[v]];
  }
}

int main() {
  int n, m;
  scanf("%d%d", &n, &m);
  pit = pool;
  fill_n(e, n, -1);
  fill_n(ee, n, -1);
  fill_n(domch, n, -1);
  for (int i = 0; i < m; i++) {
    int u, v;
    scanf("%d%d", &u, &v);
    *pit = {v, e[u]};
    e[u] = pit++-pool;
    *pit = {u, ee[v]};
    ee[v] = pit++-pool;
  }
  simpleLengauerTarjan(n, 0);

  for (int i = 0; i < n; i++)
    printf("%d: %d\n", i, idom[i]);
}
```

With a sophisticated method balancing union-find trees, the time complexity can be improved to $O(E\alpha(E,V))$.

## Semi-NCA algorithm

Loukas Georgiadis proposed the Semi-NCA algorithm in _Linear-Time Algorithms for Dominators and Related Problems_.
It has a time complexity of $O(V^2)$, but faster than the almost linear Lengauer-Tarjan's algorithm in practice.

For each vertex `v` that is not the source, `idom(v)` is the lowest common ancestor of `sdom(v)` and `parent(v)`.
For each vertex `v` in the pre-order except the source, ascend the ancestor path of `v` and find the deepest vertex whose pre-order number is less than or equal to `sdom(v)`'s number.

```cpp
#include <algorithm>
#include <cstdio>
#include <numeric>
using namespace std;

const int N = 100000, M = 500000;
struct Arc { int v, next; } pool[2*M], *pit;
int e[N], ee[N], tick, dfn[N], rdfn[N], uf[N], sdom[N], best[N], idom[N];

void dfs(int u) {
  dfn[u] = tick;
  rdfn[tick++] = u;
  for (int v, a = e[u]; ~a; a = pool[a].next)
    if (dfn[v = pool[a].v] < 0) {
      uf[v] = u;
      dfs(v);
    }
}

int eval(int v, int cur) {
  if (dfn[v] <= cur)
    return v;
  int u = uf[v], r = eval(u, cur);
  if (dfn[best[u]] < dfn[best[v]])
    best[v] = best[u];
  return uf[v] = r;
}

void semiNca(int n, int r) {
  fill_n(idom, n, -1); // delete if unreachable nodes are not needed
  fill_n(dfn, n, -1);
  tick = 0;
  dfs(r);
  iota(best, best+n, 0);
  for (int i = tick; --i; ) {
    int v = rdfn[i], u;
    sdom[v] = v;
    for (int a = ee[v]; ~a; a = pool[a].next)
      if (~dfn[u = pool[a].v]) {
        eval(u, i);
        if (dfn[best[u]] < dfn[sdom[v]])
          sdom[v] = best[u];
      }
    best[v] = sdom[v];
    idom[v] = uf[v];
  }
  for (int i = 1; i < tick; i++) {
    int v = rdfn[i];
    while (dfn[idom[v]] > dfn[sdom[v]])
      idom[v] = idom[idom[v]];
  }
}

int main() {
  int n, m;
  scanf("%d%d", &n, &m);
  pit = pool;
  fill_n(e, n, -1);
  fill_n(ee, n, -1);
  for (int i = 0; i < m; i++) {
    int u, v;
    scanf("%d%d", &u, &v);
    *pit = {v, e[u]};
    e[u] = pit++-pool;
    *pit = {u, ee[v]};
    ee[v] = pit++-pool;
  }
  semiNca(n, 0);

  for (int i = 0; i < n; i++)
    printf("%d: %d\n", i, idom[i]);
}
```

## Testdata

The following tests helped me diagnose bugs in my semi-NCA implementation.

```graphviz
digraph {
  0 -> 1
  1 -> 2
  2 -> 3
  1 -> 4
  4 -> 5
  3 -> 6
  3 -> 5
  2 -> 0
  3 -> 1
  0 -> 6
  6 -> 4
  5 -> 6
  3 -> 1
  5 -> 2
}
```

```graphviz
digraph {
  0 -> 1
  0 -> 2
  2 -> 3
  3 -> 4
  2 -> 5
  5 -> 6
  4 -> 7
  7 -> 8
  3 -> 9
  3 -> 4
  3 -> 7
  8 -> 2
  5 -> 2
  1 -> 8
  8 -> 6
  6 -> 4
  1 -> 9
}
```

```graphviz
digraph {
  0 -> 1
  1 -> 2
  0 -> 3
  1 -> 4
  1 -> 5
  4 -> 6
  2 -> 7
  5 -> 8
  3 -> 9
  4 -> 8
  6 -> 4
  6 -> 5
  5 -> 3
  5 -> 4
  6 -> 3
  3 -> 4
  2 -> 5
}
```

## Iterative DFS

```cpp
#include <cstdio>
#include <iostream>
#include <type_traits>
#include <vector>
using namespace std;

#define FOR(i, a, b) for (remove_cv<remove_reference<decltype(b)>::type>::type i = (a); i < (b); i++)
#define REP(i, n) FOR(i, 0, n)

struct arc { int v, c; };
vector<vector<arc>> e, ee;
vector<int> seq, pre, post, idom;
int dfn;

void dfs(int u) {
  pre[u] = dfn++;
  for (arc a: e[u])
    if (idom[a.v] == -1) {
      idom[a.v] = u;
      dfs(a.v);
    }
  seq.push_back(u);
  post[u] = dfn;
}

void idfs(int n, int r) {
  bool changed;
  pre.resize(n);
  post.resize(n);
  seq.clear();
  idom.assign(n, -1);
  idom[r] = -2;
  dfn = 0;
  dfs(r);
  do {
    changed = false;
    for (int i = seq.size() - 2; i >= 0; i--) {
      int v = seq[i], x = -1;
      for (arc &a: ee[v]) {
        if (x == -1)
          x = a.v;
        else {
          int y = a.v;
          while (x != y) {
            if (pre[x] > pre[y])
              x = idom[x];
            else
              y = idom[y];
          }
        }
      }
      if (x != idom[v]) {
        idom[v] = x;
        changed = true;
      }
    }
  } while (changed);
}

int main() {
  int n, i, j, c;
  cin >> n;
  e.resize(n);
  ee.resize(n);
  while (cin >> i >> j) {
    e[i].push_back(arc{j});
    ee[j].push_back(arc{i});
  }

  idfs(n, 0);
  REP(i, n)
    printf("%d: %d\n", i, idom[i]);
}
```

## Dynamic dominators

When the CFG changes by one edge at a time, the dominator tree can be updated incrementally instead of rebuilt.
LLVM's `DominatorTree` does this for `insertEdge`/`deleteEdge`/`applyUpdates`, following _An Experimental Study of Dynamic Dominators_.

Two facts keep the work local. Inserting an edge only makes dominance weaker, so some `idom(v)` move up toward the root. Deleting an edge only makes dominance stronger, so some `idom(v)` move down, or `v` becomes unreachable.
Each tree node stores its depth, and the key query is `nca(a,b)`, the nearest common ancestor (nearest common dominator).

### Insertion

Insert `(x,y)`.
If `y` was unreachable, run Semi-NCA on the region that just became reachable and attach the subtree under `x`.

Otherwise `y` is reachable, handled by *depth based search*.
Let `nca = nca(x,y)`. The new edge creates a path `root -> ... -> nca -> x -> y`, so a node reachable only below `nca` may now bypass its old dominator.
`v` is *affected* iff `depth(nca)+1 < depth(v)` and there is a path `P` from `y` to `v` with `depth(w) >= depth(v)` for every `w` on `P`; then `idom(v)` becomes `nca`.

The second condition wants a path from `y` to `v` whose minimum depth is at least `depth(v)` -- a widest-path problem, solved with a bucket-queue Dijkstra that pops the deepest node first.
Starting from `y`, for a successor `s` of the current node at level `L`:

- skip if `depth(s) <= depth(nca)+1` or `s` is already visited;
- if `depth(s) <= L`, the bottleneck to `s` is `depth(s)`, so `s` is *affected*;
- otherwise (`depth(s) > L`) `s` is unaffected for now, but keep expanding through it -- a deeper node can still lead back to an affected node at level `L`.

Set `idom(v) = nca` for each affected `v`. Changing a node's idom cascades new depths through its subtree; unaffected descendants keep their idom but shift level.
<https://reviews.llvm.org/D58349>

### Deletion

Delete `(x,y)` and let `nca = nca(x,y)`. If `y == nca` (i.e. `y` dominates `x`), nothing changes.
Otherwise check whether `y` still has *proper support*: a still-reachable predecessor `p` with `nca(y,p) != y`, i.e. a surviving path keeping `y` dominated from above.

- With proper support, only the subtree topped at `nca(x,y)` can change. Re-run Semi-NCA on the nodes below that level and reattach.
- Without it, `y` and part of its subtree become unreachable. Erase the nodes that lost reachability, then rebuild the remaining affected subtree with Semi-NCA.

So deletion still calls Semi-NCA, but only on a small local subgraph.

### Rebuilding from scratch

The incremental path gives up and reruns Semi-NCA on the whole graph when the affected region reaches the root, when the postdominator roots change, or, for a batch of `k` updates on `n` nodes, when `k` exceeds about `n/40`.

## Natural loops

See [natural loops](/blog/2025-01-20-natural-loops).
