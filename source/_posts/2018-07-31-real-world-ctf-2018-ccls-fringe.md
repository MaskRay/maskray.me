---
layout: post
title: Real World CTF 2018 ccls-fringe命題報告
author: MaskRay
tags: [ccls, ctf, forensics]
---

上週日給Real World CTF 2018出了一道forensics題ccls-fringe，向解出此題的31支隊伍表達祝賀。

<!-- more -->

上一次出題已是2016年，一直沒有人教我pwn、reverse、web所以只能從平常接觸的東西裏勉強抽出要素弄成題目。

kelwya找我來出題，他說forensics也可以，阻止了我說另請高明。另外也想給自己的項目打廣告，萌生了從[ccls](https://github.com/MaskRay/ccls)的cache弄一道forensics的想法，因爲惰性拖了兩個星期沒有動手。之前一次LeetCode Weekly前在和一個學弟聊天，就想把他的id嵌入了flag。LeetCode第一題Leaf-Similar Trees沒有叫成same-fringe，所以我的題目就帶上fringe字樣“科普”一下吧。

下載`ccls-fringe.tar.xz`後解壓得到`.ccls-cache/@home@flag@/fringe.cc.blob`。這是存儲的cache文件。

這裏體現了ccls和cquery的一個不同點。ccls默認使用`"cacheFormat":"binary"`了，這個自定義的簡單序列化格式，cquery仍在使用JSON。

寫段程序讀入文件，用`ccls::Derialize`反序列化成`IndexFile`，用`ToString`可以轉成JSON字串，閱讀可以發現其中一個變量`int b`：

```javascript
{
  "usr": 7704954053858737267,
  "detailed_name": "int b",
  "qual_name_offset": 4,
  "short_name_offset": 4,
  "short_name_size": 1,
  "hover": "",
  "comments": "flag is here",
  "declarations": [],
  "spell": "16:80-16:81|1935187987660993811|3|2",
  "extent": "16:76-16:81|1935187987660993811|3|0",
  "type": 52,
  "uses": [],
  "kind": 13,
  "storage": 0
},
```

`clang -fparse-all-comments`會把非Doxygen註釋也嵌入AST，註釋暗示了我們應該找和它類似的變量。`spell`差不多表示spelling SourceRange，第80列很奇怪。寫一段程序收集位於80列的其他單字符變量，按行號排序：

```c++
#include "indexer.h"

const char blob[] = ".ccls-cache/@home@flag@/fringe.cc.blob";

__attribute__((constructor)) void solve() {
  std::string content = *ReadContent(blob);
  auto file = ccls::Deserialize(SerializeFormat::Binary, blob, content, "", {});
  std::vector<std::pair<int, char>> wod;
  for (auto &[usr, v] : file->usr2var) {
    Maybe<Use> spell = v.def.spell;
    if (spell) {
      Position start = spell->range.start;
      if (start.column == 79)
        wod.emplace_back(start.line, v.def.detailed_name[4]);
    }
  }
  sort(wod.begin(), wod.end());
  for (auto t: wod)
    putchar(t.second);
  puts("");
  exit(0);
}
```

`Makefile`指定編譯選項和鏈接選項，除了LLVM只有rapidjson一個依賴了。

```make
CCLS := $(HOME)/Dev/Util/ccls
LLVM := $(HOME)/Dev/llvm

LDFLAGS := -shared
CXXFLAGS := -std=c++17 -fPIC -g
CXXFLAGS += -I$(CCLS)/src -I$(CCLS)/third_party -I$(CCLS)/third_party/rapidjson/include
CXXFLAGS += -I$(LLVM)/include -I$(LLVM)/Release/include
CXXFLAGS += -I$(LLVM)/tools/clang/include -I$(LLVM)/Release/tools/clang/include

solve.so: solve.cc
	$(LINK.cc) $^ -o $@
```

編譯成`ccls.so`，讓`ccls`吐出flag：
```
% LD_PRELOAD=./ccls.so ~/Dev/Util/ccls/Debug/ccls
blesswodwhoisinhk
```

这只是这人用的id的一个子串，他教育我要多做Codeforces，是很好玩的(可惜我依然這麼菜，嗚嗚😭😭😿😿)。

把same-fringe problem寫成coroutine形式也是爲了紀念一個去New York City的同學，他在Stanford讀研期間，給一門課當助教時曾讓我校對一個課程實驗Cooperative User-Level Threads。

```cpp
#include <iostream>
#include <vector>
#include <ucontext.h>
using namespace std;

struct TreeNode {
  int val;
  TreeNode *left;
  TreeNode *right;
};

struct Co {
  ucontext_t c;
  char stack[8192];
  TreeNode *ret;
  Co(ucontext_t *link, void (*f)(Co *, TreeNode *), TreeNode *root) {     {int b; /* flag is here */}
    getcontext(&c);                                                       {int l;}
    c.uc_stack.ss_sp = stack;                                             {int e;}
    c.uc_stack.ss_size = sizeof stack;                                    {int s;}
    c.uc_link = link;                                                     {int s;}
    makecontext(&c, (void(*)())f, 2, this, root);
  }
  void yield(TreeNode *x) {                                               {int w;}
    ret = x;                                                              {int o;}
    swapcontext(&c, c.uc_link);                                           {int d;}
  }
};

void dfs(Co *c, TreeNode *x) {                                            {int w;}
  if (!x) return;                                                         {int h;}
  if (!x->left && !x->right) c->yield(x);                                 {int o;}
  dfs(c, x->left);                                                        {int i;}
  dfs(c, x->right);                                                       {int s;}
}

class Solution {
public:
  bool leafSimilar(TreeNode *root1, TreeNode *root2) {                    {int i;}
    ucontext_t c;                                                         {int n;}
    Co c2(&c, dfs, root2), c1(&c2.c, dfs, root1);                         {int h;}
    do {                                                                  {int k;}
      c1.ret = c2.ret = nullptr;
      swapcontext(&c, &c1.c);
    } while (c1.ret && c2.ret && c1.ret->val == c2.ret->val);
    return !c1.ret && !c2.ret;
  }
};

void insert(TreeNode **x, TreeNode &y) {
  while (*x)
    x = y.val < (*x)->val ? &(*x)->left : &(*x)->right;
  *x = &y;
}

int main() {
  TreeNode xs[] = {{3},{1},{5},{0},{2},{4},{6}};
  TreeNode ys[] = {{5},{3},{6},{1},{4},{0},{2}};
  TreeNode zs[] = {{3},{1},{5},{0},{2},{6}};
  TreeNode *tx = nullptr, *ty = nullptr, *tz = nullptr;
  for (auto &x: xs) insert(&tx, x);
  for (auto &y: ys) insert(&ty, y);
  for (auto &z: zs) insert(&tz, z);
  Solution s;
  cout << s.leafSimilar(tx, ty) << endl;
  cout << s.leafSimilar(tx, tz) << endl;
}
```

假如你用clang trunk (>=7)，ccls可以檢索macro replacement-list中的引用。某些限定條件下template instantiation得到的引用也能索引。

## 代碼

<https://github.com/MaskRay/RealWorldCTF-2018-ccls-fringe>

ccls的xref功能可以看<https://github.com/MaskRay/ccls/wiki/Emacs>裏的截圖。Vim/NeoVim用戶也強烈建議看看Emacs能達到什麼效果。最近也加了一個vscode-ccls，但我不用VSCode因此沒有精力維護。
