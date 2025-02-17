---
layout: post
title: 二叉樹遍歷算法總結
author: MaskRay
tags: [algorithm, tree, traversal]
---

三類：

- 棧
  + 系統棧
  + 自己實現的棧
- 線索：Morris preorder/inorder/postorder traversal
- 其他edge-crawling方式：Schorr-Waite graph marking algorithm、Lindstrom-Dwyer algorithm

<!-- more -->

```c++
#include <algorithm>
#include <cstdio>
#include <queue>
#include <stack>
using namespace std;

struct Node {
  int key;
  int s; // only used by Schorr-Waite
  Node *l, *r;
  Node(int key, Node *l, Node *r) : key(key), l(l), r(r), s(0) {}
  void visit() const {
    printf("%d\n", key);
  }
};

enum Type { PREORDER, INORDER, POSTORDER };

/*
preorder模擬棧版：棧中存放的節點`x`表示之後待遍歷`x`的子樹。
*/
void preorder(Node *p)
{
  if (p) {
    p->visit();
    preorder(p->l);
    preorder(p->r);
  }
}

/*
inorder模擬棧版：棧中存放的節點`x`表示`x`待訪問，且之後遍歷`x`的右子樹。
*/
void inorder(Node *p)
{
  if (p) {
    inorder(p->l);
    p->visit();
    inorder(p->r);
  }
}

void postorder(Node *p)
{
  if (p) {
    postorder(p->l);
    postorder(p->r);
    p->visit();
  }
}

void preorder_stack(Node *p)
{
  stack<Node *> s;
  for(;;) {
    while (p) {
      p->visit();
      if (p->r)
        s.push(p->r);
      p = p->l;
    }
    if (s.empty()) break;
    p = s.top();
    s.pop();
  }
}

void inorder_stack(Node *p)
{
  stack<Node *> s;
  for(;;) {
    while (p) {
      s.push(p);
      p = p->l;
    }
    if (s.empty()) break;
    p = s.top();
    s.pop();
    p->visit();
    p = p->r;
  }
}

/*
棧中存放的節點`x`表示兩種階段之一：
1. 當前在訪問`x`左子樹，之後待遍歷`x`的右子樹，然後訪問`x`。
2. 當前在訪問`x`的右子樹，之後待訪問`x`。

祖先節點進入階段2的時刻晚於後裔節點。
外層循環開始處即`while (p) {`行，以下兩個條件之一滿足：
1. `p`非NULL，需要遍歷`p`子樹，棧上存放`p`的所有祖先。
2. `p`爲NULL，棧上是棧頂及棧頂的所有祖先，下一個訪問的是棧頂。

遍歷一棵子樹的方式是沿着左孩子鏈接下溯並把沿途節點都壓棧，直到遇到NULL。
若遇到的最後一個節點`x`有右孩子`y`，則當前指針置爲`y`，`x`及`x`的一些連續的祖先進入階段2；
否則(第二個while循環實現這一複雜邏輯)說明`x`已在階段2，訪問`x`，
同時`x`一些連續的祖先也進入階段2了，也需要訪問並從棧中彈出。
*/
void postorder_stack(Node *p)
{
  stack<Node *> s;
  for(;;) {
    while (p) {
      s.push(p);
      p = p->l;
    }
    while (! s.empty() && s.top()->r == p) {
      p = s.top();
      s.pop();
      p->visit();
    }
    if (s.empty()) break;
    p = s.top()->r;
  }
}

void postorder2_stack(Node *p)
{
  stack<Node *> s;
  for(;;) {
    while (p) {
      if (p->r)
        s.push(p->r);
      s.push(p);
      p = p->l;
    }
    if (s.empty()) break;
    p = s.top();
    s.pop();
    if (! s.empty() && p->r == s.top()) {
      s.pop();
      s.push(p);
      p = p->r;
    } else {
      p->visit();
      p = NULL;
    }
  }
}

/*
按層級順序訪問，使用隊列進行BFS
*/
void level_order(Node *p)
{
  queue<Node *> q;
  if (p) {
    q.push(p);
    while (! q.empty()) {
      p = q.front();
      q.pop();
      p->visit();
      if (p->l)
        q.push(p->l);
      if (p->r)
        q.push(p->r);
    }
  }
}

/*
preorder或inorder可以實現O(1)額外空間的遍歷，
方法是把右孩子爲空的節點的右孩子改造成中序後繼的線索指針。
*/
void morris_traversal(Node *p, Type t)
{
  while (p) {
    Node *q = p->l;
    if (q) {
      while (q->r && q->r != p) q = q->r;
      if (q->r == p)
        q->r = NULL;
      else {
        if (t == PREORDER)
          p->visit();
        q->r = p;
        p = p->l;
        continue;
      }
    } else if (t == PREORDER)
      p->visit();
    if (t == INORDER)
      p->visit();
    p = p->r;
  }
}

// morris_postorder_traversal的輔助函數
void reverse_right_chain(Node *x, Node *y)
{
  Node *p = x, *q = x->r, *r;
  while (p != y) {
    r = q->r;
    q->r = p;
    p = q;
    q = r;
  }
}

/*
增量構造線索進行post-order遍歷比較複雜
*/
void morris_postorder_traversal(Node *p)
{
  Node aux;
  aux.l = p;
  aux.r = NULL;
  p = &aux;
  while (p) {
    Node *q = p->l;
    if (q) {
      while (q->r && q->r != p) q = q->r;
      if (q->r == p) {
        reverse_right_chain(p->l, q);
        for (Node *r = q; ; r = r->r) {
          r->visit();
          if (r == p->l) break;
        }
        reverse_right_chain(q, p->l);
        q->r = NULL;
      } else {
        q->r = p;
        p = p->l;
        continue;
      }
    }
    p = p->r;
  }
}

/* Schorr-Waite graph marking algorithm
 *
 * 該算法需要在節點信息裏額外維護一個域 s，表示遍歷過程中的訪問次數，有如下幾種取值：
 * 0: 尚未訪問
 * 1: 訪問過1次(preorder)，尚未遍歷其左右子樹
 * 2: 訪問過2次(inorder)，已遍歷其左子樹，尚未遍歷其右子樹
 * 3: 訪問過3次(postorder)，已遍歷其左右子樹
 *
 * 訪問到 x 時，其祖先節點及它們的子樹遍歷進度(在遍歷左子樹還是右子樹)是通過棧維護的。
 * 該算法把祖先節點下溯的指針翻轉，再通過設置一個父節點指針 p 以把祖先節點組織爲一個鏈。
 * 通過 s 可以知道各祖先節點當前在遍歷左子樹還是右子樹。
 *
 * 若註釋掉代碼中的 x->s = 0; 則在算法執行完後遍歷到的節點的 s 值爲3，可以判斷節點是否被訪問過。
 *
 * 該算法可用於遍歷一般的有向圖，也需要註釋掉 x->s = 0。帶有 q 個指針域的節點將被訪問 q+1 次。
 */

void schorr_waite(Node *x, Type t)
{
  if (! x) return;
  Node *y, *p = NULL;
  for(;;) {
    if ((int)t == (int)x->s)
      x->visit();
    if (x->s < 2) {
      x->s++;
      y = x->s == 1 ? x->l : x->r;
      if (y && y->s == 0) {
        (x->s == 1 ? x->l : x->r) = p;
        p = x;
        x = y;
      }
    } else {
      x->s = 0;               // reset to 0
      if (! p) return;
      y = x;
      x = p;
      if (x->s == 1)
        p = x->l, x->l = y;
      else
        p = x->r, x->r = y;
    }
  }
}

// http://www.cs.cornell.edu/courses/cs312/2007fa/lectures/lec21-schorr-waite.pdf
// 下面是一種變體，也可用於一般有向圖，每個節點都有兩個指針域
// 原始的Schorr-Waite在下溯時把指針和 p 交換，上溯時再交換回來
// 該方法使用四指針置換和三指針置換來統一幾種邏輯判斷

void schorr_waite_alternative(Node *p, Type t)
{
  Node *q = (Node *)-1;
  while (p != (Node *)-1) {
    if ((int)t == (int)p->s)
      p->visit();
    p->s++;
    if (p->s == 3 || p->l && p->l->s == 0) {
      Node *r = p->l;
      p->l = p->r;
      p->r = q;
      q = p;
      p = r;
    } else {
      Node *r = p->l;
      p->l = p->r;
      p->r = q;
      q = r;
    }
  }
}

// Lindstrom-Dwyer algorithm
// Schorr-Waite的變體，對於二叉樹，上面 schorr_waite_alternative 的代碼可以進一步簡化
// 考察條件 p->s == 3 || p->l && p->l->s == 0
// 若 p->s != 3 則有 1 <= p->s && p->s <= 2
// 此時 p->l 爲原節點的左子樹或右子樹，因爲是樹，不存在其他節點的指向它們
// 因此 p->l 若非 NULL 則必有 p->l->s != 0
// 該條件只有在 p->l == NULL 時不成立。即使不成立，我們依然可以套用四指針置換
// 但需要把 p(此時爲NULL) 設爲 q，把 q 設爲 NULL
// 之後 s 僅用於判斷節點訪問次數，不再參與邏輯控制
// 如果不在乎一個節點調用 visit 三次的話，可以省去 s

void lindstrom_dwyer(Node *p)
{
  Node *q = (Node *)-1;
  while (p != (Node *)-1) {
    p->visit();
    Node *r = p->l;
    p->l = p->r;
    p->r = q;
    q = p;
    p = r;
    if (! p) p = q, q = NULL;
  }
}

int main()
{
  Node *a[7];
  for (int i = 7; i--; )
    a[i] = new Node(i, i*2+1 < 7 ? a[i*2+1] : NULL, i*2+2 < 7 ? a[i*2+2] : NULL);
  preorder(a[0]);
  inorder(a[0]);
  postorder(a[0]);
  level_order(a[0]);
  morris_traversal(a[0], PREORDER);
  morris_traversal(a[0], INORDER);
  preorder_stack(a[0]);
  inorder_stack(a[0]);
  postorder_stack(a[0]);
  postorder2_stack(a[0]);
  schorr_waite(a[0], PREORDER);
  schorr_waite(a[0], INORDER);
  schorr_waite(a[0], POSTORDER);
  schorr_waite_alternative(a[0], PREORDER);
  schorr_waite_alternative(a[0], INORDER);
  schorr_waite_alternative(a[0], POSTORDER);
  lindstrom_dwyer(a[0]);
}
```
