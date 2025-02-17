---
layout: post
title: Natural loops
author: MaskRay
tags: [algorithm,graph]
---

A [dominator tree](/blog/2020-12-11-dominator-tree) can be used to compute natural loops.

* For every node `H` in a post-order traversal of the dominator tree (or the original CFG), find all predecessors that are dominated by `H`. This identifies all back edges.
* Each back edge `T->H` identifies a natural loop with `H` as the header.
  + Perform a flood fill starting from `T` in the reversed dominator tree (from exiting block to header)
  + All visited nodes reachable from the root belong to the natural loop associated with the back edge. These nodes are guaranteed to be reachable from `H` due to the dominator property.
  + Visited nodes unreachable from the root should be ignored.
  + Loops associated with visited nodes are considered subloops.

<!-- more -->

Here is an C++ implementation:

```cpp
#include <cstdio>
#include <deque>
#include <numeric>
#include <vector>
using namespace std;

vector<vector<int>> e, ee, edom;
vector<int> dfn, dfn2, rdfn, uf, best, sdom, idom;
int tick;

void dfs(int u) {
  dfn[u] = tick;
  rdfn[tick++] = u;
  for (int v : e[u])
    if (dfn[v] < 0) {
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
  idom.assign(n, -1);
  dfn.assign(n, -1);
  rdfn.resize(n); // initial values are unused
  uf.resize(n); // initial values are unused
  sdom.resize(n); // initial values are unused
  tick = 0;
  dfs(r);
  best.resize(n);
  iota(best.begin(), best.end(), 0);
  for (int i = tick; --i; ) {
    int v = rdfn[i];
    sdom[v] = v;
    for (int u : ee[v])
      if (~dfn[u]) {
        eval(u, i);
        if (dfn[best[u]] < dfn[sdom[v]])
          sdom[v] = best[u];
      }
    best[v] = sdom[v];
    idom[v] = uf[v];
  }
  edom.assign(n, vector<int>());
  for (int i = 1; i < tick; i++) {
    int v = rdfn[i];
    while (dfn[idom[v]] > dfn[sdom[v]])
      idom[v] = idom[idom[v]];
    edom[idom[v]].push_back(v);
  }
}

struct Loop {
  int idx, header;
  Loop *parent = nullptr, *child = nullptr, *next = nullptr;
  vector<int> nodes;
};
deque<Loop> loops;

void postorder(int u) {
  dfn[u] = tick;
  for (int v : edom[u])
    if (dfn[v] < 0)
      postorder(v);
  rdfn[tick++] = u;
  dfn2[u] = tick;
}

void identifyLoops(int n, int r) {
  vector<int> worklist;
  vector<Loop *> to_loop(n);
  dfn.assign(n, -1);
  dfn2.assign(n, -1);
  tick = 0;
  postorder(r);
  loops.clear();
  for (int i = 0; i < tick; i++) {
    int header = rdfn[i];
    for (int u : ee[header])
      if (dfn[header] <= dfn[u] && dfn2[u] <= dfn2[header])
        worklist.push_back(u);
    if (worklist.empty())
      continue;
    loops.push_back(Loop{(int)loops.size(), header});
    Loop *lp = &loops.back();
    while (worklist.size()) {
      int v = worklist.back();
      worklist.pop_back();
      if (!to_loop[v]) {
        if (dfn[v] < 0) // Skip unreachable node
          continue;
        // Find a node not in a loop.
        to_loop[v] = lp;
        lp->nodes.push_back(v);
        if (v == header)
          continue;
        for (int u : ee[v])
          worklist.push_back(u);
      } else {
        // Find a subloop.
        Loop *sub = to_loop[v];
        while (sub->parent)
          sub = sub->parent;
        if (sub == lp)
          continue;
        sub->parent = lp;
        sub->next = lp->child;
        lp->child = sub;
        for (int u : ee[sub->header])
          if (to_loop[u] != sub)
            worklist.push_back(u);
      }
    }
  }
}

int main() {
  int n, m;
  scanf("%d%d", &n, &m);
  e.resize(n);
  ee.resize(n);
  for (int i = 0; i < m; i++) {
    int u, v;
    scanf("%d%d", &u, &v);
    e[u].push_back(v);
    ee[v].push_back(u);
  }
  semiNca(n, 0);
  for (int i = 0; i < n; i++)
    printf("%d: %d\n", i, idom[i]);

  identifyLoops(n, 0);
  for (Loop &lp : loops) {
    printf("loop %d:", lp.idx);
    for (int v : lp.nodes)
      printf(" %d", v);
    for (Loop *c = lp.child; c; c = c->next)
      printf(" (loop %d)", c->idx);
    puts("");
  }
}
```

The code iterates over the dominator tree in post-order. Alternatively, a post-order traversal of the original control flow graph could be used.

`worklist` may contain duplicate elements. This is acceptable. You could also deduplicate elements.

Importantly, the header predecessor of a subloop can be another subloop.

In the final `loops` array, parent loops are listed after their child loops.

This example examines multiple subtle details: a self-loop (node 6), an unreachable node (node 8), and a scenario where the header predecessor of one subloop (nodes 2 and 3) leads to another subloop (nodes 4 and 5).

```
9 12
0 1
1 2
1 7
2 3
2 4
3 2
8 3
4 5
4 6
5 4
6 1
6 6
```

Use `awk 'BEGIN{print "digraph G{"} NR>1{print $1"->"$2} END{print "}"}'` to generate a graphviz dot file.
