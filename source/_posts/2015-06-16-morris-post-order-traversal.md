---
layout: post
title: 常數空間Invert Binary Tree與仿Morris法後序遍歷
author: MaskRay
tags: [algorithm, tree, traversal, leetcode]
---

今天在地鐵上浪費了好多時間……嗚嗚……做其他事都沒有效率，就利用這些時間寫字了。

前幾天LeetCode的[Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/)火了。名稱有點糟，無法準確描述要求。

主要思路是遍歷二叉樹，把訪問操作修改爲交換左右孩子，每個節點都交換一次即可。

如果採用前序或中序遍歷，則子樹交換髮生在遍歷某棵子樹之前，會引起麻煩。因此我想到了後序遍歷，在遍歷完子樹後再交換左右孩子，重心就是如何實現常數空間複雜度的後序遍歷。某些資料/問題稱之爲Morris post-order traversal。

<!-- more -->

考慮樹遍歷的遞歸算法，一個節點的訪問次數爲度(孩子數)+1。對於前序遍歷，第一次訪問產生輸出；對於中序遍歷，第二次訪問產生輸出。鑑於輸出節點信息的那次訪問比較簡單，把它和下一次訪問合併。

因此前序遍歷最多只有兩次訪問，訪問時執行的操作如下：

- 若左孩子非空則遍歷左子樹
- 訪問當前節點。遍歷右子樹

中序遍歷相仿：

- 訪問當前節點。若左孩子非空則遍歷左子樹
- 遍歷右子樹

代碼如下：

```c
struct Node { Node *l, *r; void visit(); };

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
```

定義一個概念，節點`x`的**右鏈**表示代碼`while (x) x = x->r;`中`x`指向過的所有節點，**最右**節點爲最後一個不等於`NULL(nullptr)`的`x`指針值。

遍歷完某棵子樹後需要回到中序序列的後繼。一個左孩子非空的節點會成爲當前訪問的節點(`p`)兩次。第一次`p`左子樹的最右節點`q`尚未建立線索，令其右孩子指向中序序列後繼`p`(線索)；第二次`q`的線索存在，指向`p`，刪除之。

對於後序遍歷，一個度爲2的節點只有在第3次訪問時才輸出，而Morris in-order traversal算法中一個節點最多只會作爲`p`兩次，無法提供3次訪問。我們需要挖掘更多的信息。實際上除了根的右鏈以外的節點都被指向過3到4次：

- 該節點所在右鏈的父節點爲`p`時，被變量`q`指向
- 第一次被`p`指向。此時左孩子爲空或左子樹最右節點的thread不存在
- 若左孩子非空，會第二次被`p`指向，左子樹右鏈的thread存在
- 該節點所在右鏈的父節點爲`p`時，被變量`q`指向

如果把第2次變量`q`指向當作第3次訪問，則除了根的右鏈以外的節點都能被訪問2(左孩子爲空)或3(左孩子非空)次，如果創建一個虛擬的根，原來的根作爲它的左孩子，則根的右鏈的情況也和其他節點相同了，可以統一處理。

`while (q->r && q->r != p) q = q->r;`中若倒序輸出各個`q`的值，則得到了後序序列。常數空間複雜度實現倒序輸出時需要用到鏈表翻轉的技巧，翻轉右鏈(可以常數空間實現)再沿着鏈輸出，之後再翻轉回來(還原)。

```c
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
```

回到Invert Binary Tree，其實不要求嚴格的遍歷順序，因此我們不必翻轉兩次右鏈達到後序效果：

```c
class Solution {
public:
  TreeNode* invertTree(TreeNode* root) {
    TreeNode aux(0), *p = &aux;
    aux.left = root;
    while (p) {
      TreeNode *q = p->left;
      if (q) {
        while (q->right && q->right != p) q = q->right;
        if (q->right == p) {
          for (TreeNode *r = p->left; ; r = r->left) {
            swap(r->left, r->right);
            if (r == q) break;
          }
          q->left = NULL;
        } else {
          q->right = p;
          p = p->left;
          continue;
        }
      }
      p = p->right;
    }
    return aux.left;
  }
};
```

其他題可以參考[LeetCode solutions](2014-06-29-leetcode-solutions)，持續更新中。
