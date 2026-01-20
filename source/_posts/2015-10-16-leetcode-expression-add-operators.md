---
layout: post
title: LeetCode Expression Add Operators：SLR(1)、meet-in-the-middle及其他
author: MaskRay
tags: [algorithm, leetcode]
mathjax: true
---

## 題目描述

[Expression Add Operators](https://leetcode.com/problems/expression-add-operators/)

<!-- more -->

Given a string that contains only digits 0-9 and a target value, return all possibilities to add binary operators (not unary) +, -, or * between the digits so they evaluate to the target value.

```
Examples:
"123", 6 -> ["1+2+3", "1*2*3"]
"232", 8 -> ["2*3+2", "2+3*2"]
"105", 5 -> ["1*0+5","10-5"]
"00", 0 -> ["0+0", "0-0", "0*0"]
"3456237490", 9191 -> []
```

Credits:
Special thanks to @davidtan1890 for adding this problem and creating all test cases.

## 分析

枚舉所有位置的符號(並置、`+`、`-`、`*`)後轉化爲表達式求值。加速的辦法是讓枚舉和求值一起進行。

考慮題目要求的文法：
```
E ::= E "+" F | E "-" F
F ::= F "*" L
L ::= "0" | L1
L1 ::= "1" | ... | "9" | L1 "0" | ... | L1 "9"
```

使用不考慮前導0的簡化文法。我們會採用遞歸算法，形似recursive descent parser，在遞歸過程中不難採取一些措施避免生成前導0。
```
E ::= E "+" F | E "-" F
F ::= F "*" L
L ::= "0" | ... | "9" | L "0" | ... | L "9"
```

Parse tree以`0`~`9`爲葉子，`+`、`-`、並置(下面用`@`表示)爲內部節點，其最右節點到根的距離小於等於3，之後會說明原因。對於某一前綴的parse tree，添加一個符號和一個數字後會得到一棵新的parse tree，這一過程可以看作在二叉樹中添加一中序遍歷序號最大的節點(類似於Cartesian tree線性構造算法)。取決於添加的符號，可能有以下三種變化。最左邊爲原先的parse tree，後面三棵爲添加不同符號後得到的parse tree：
```
   +         +         +           +
  / \       / \       / \         / \
 2   *     2   *     2   *       +   6
    / \       / \       / \     / \
   3   @     3   @     *   6   2   *
      / \       / \   / \         / \
     4   5     @   6 3   @       3   @
              / \       / \         / \
             4   5     4   5       4   5
```

我們給parse tree每個節點設置semantic value，即對應的表達式的結果。因爲節點按照中序遍歷順序添加，任何左子樹的結構不會再發生變化，可以把整棵子樹收縮成一個節點。`@`是文法優先級最高，也可以規約，即把對應子樹收縮。上面的四棵樹收縮後變成下面這樣：
```
   +         +         +           +
  / \       / \       / \         / \
 2   *     2   *     2   *      137  6
    / \       / \       / \
   3   45    3  456   135  6
```

文法中的三個產生式有層級關係，高優先級符號的子樹中沒有低優先級符號，這就是任意節點到根距離小於等於3的原因。從另一個角度看，LR(0)項集是無圈的，最長路徑長度爲3。枚舉並求parse tree的過程很像SLR(1)，又像Earley parser。

`-`可以看作`+`，同時把右邊的數字視爲相反數。這樣，我們得到的tree只剩下`+` `*`兩種操作符。`+`、`*`都有單位元，若最右路徑沒有`*`，可以補上一個左孩子爲1的節點；若沒有`+`，可以補上一個左孩子爲0的節點。這樣保證了最右節點到根的距離恆爲2，且樹中有三個葉子，三個semantic values。

以上解釋了三個semantic values可以刻畫狀態的原因。從左到右枚舉各個位置上的符號，不難得到一個邊枚舉邊求值的算法，枚舉量爲$O(4^n)$。常見優化技巧是meet-in-the-middle，即兩邊同時枚舉到中間，把一邊枚舉的結果記錄下來，另一邊枚舉到相遇時查表。用上這一技巧後，該算法很像packrat parser。倘若題目引入括號，那麼LR(0)項集就含圈了，最右節點到根的距離就沒有限制，3個semantic values就不夠刻畫狀態。

注意相遇時要合併兩邊的部分結果。下面逐一考慮。

`+`相遇比較容易。假設左邊部分的和爲$s$，那麼從右邊部分找和爲$target-s$的式子即可。

`-`相遇類似。

`@`相遇，合併時兩邊都得用到三個semantic values，$a+b*c$與$d*e+f$合併得到$a+b*cd*e+f$，難以計算，不存在有效的查表方法。因此我們不處理`@`相遇，放置並置符號後，交給遞歸調用的函數在之後相遇。若之後出現了`+`或`-`，那麼相遇過程就延遲到那時進行；若沒有出現，則在全式末尾進行。

`*`相遇，若像`@`那麼不處理，考慮枚舉量。設從右到左的有$nn$個操作符待枚舉，預處理表的枚舉量爲$O(4^{nn})$。從左到右的枚舉量爲$O(4^{n-nn})$，每一項得在右邊記錄集合中查表。因爲`*`、`@`兩種符號都沒處理相遇，還得考慮延遲相遇的情況數，爲$O(2^{nn})$。若查表時間爲常數，那麼總的枚舉量爲$O(4^{nn}+4^{n-nn}*2^{nn})$。根據該式算得$nn$取$n/3$時枚舉量最小，考慮到查表的代價，實際取值應在$n/3$與$n/2$之間。

處理`*`相遇需要用到兩個semantic values，把左邊部分在最後一個加號處斷開(`-`可以轉化爲`+`)，得到$a+b$的形式，右邊在第一個加號處斷開，得到$*c+d$的形式，拼接得到$a+b*c+d$。我們希望它等於$target$，即$b*c+d=target-a$。也就是說，對於右邊兩個semantic values $c$和$d$所表示的直線$y=cx+d$，我們希望它經過點$(b,target-a)$。

於是問題轉化爲：給出$O(4^{nn})$個點、$O(4^{n-nn})$根直線，對於每個點求經過它的直線。不妨設$nn$取$n/2$。一種思路是藉助spatial data structure。若能在$O(4^{n/2}*A)$的時間內計算，那麼總的時間複雜度爲$O(4^{n/2}+4^{n/2}*A)$。考慮把點組織成K-d tree，對每根線逐一查詢，$A$不小於$O(\sqrt{4^{nn/2}})$，不優於不處理`*`相遇。更好的方法可能是把點和線批量處理。若$A$爲$O(\log{4^{n/2}})=O(n)$，則總的時間複雜度爲$O(n*2^n)$。

```
typedef long long ll;
class Solution {
  string a;
  int n, nn, target;
  vector<multimap<ll, string>> e_plus, e_minus;
  vector<string> res;
  void backward(int k, string s, ll add, ll mul, ll last, ll ten) {
    if (k < nn) return;
    int x = a[k-1]-'0';
    ll ten2 = 10*ten, sum = add+mul*last;
    backward(k-1, string(1, a[k-1])+s, add, mul, last+ten2*x, ten2);
    if (ten == 1 || last >= ten) { // `last` has no leading zero
      backward(k-1, string(1, a[k-1])+'*'+s, add, mul*last, x, 1);
      backward(k-1, string(1, a[k-1])+'+'+s, sum, 1, x, 1);
      backward(k-1, string(1, a[k-1])+'-'+s, add-mul*last, 1, x, 1);
      e_plus[k].insert(make_pair(sum, s));
      e_minus[k].insert(make_pair(add-mul*last, s));
    }
  }
  void forward(int k, string s, ll add, ll mul, ll last) {
    ll sum = add+mul*last;
    if (k == n) {
      if (sum == target)
        res.push_back(s);
      return;
    }
    int x = a[k]-'0';
    if (last) // no leading zero
      forward(k+1, s+a[k], add, mul, last*10+x);
    forward(k+1, s+'*'+a[k], add, mul*last, x);
    if (k < nn) {
      forward(k+1, s+'+'+a[k], sum, 1, x);
      forward(k+1, s+'-'+a[k], sum, -1, x);
    } else {
      auto rg = e_plus[k].equal_range(target-sum);
      for (auto it = rg.first; it != rg.second; ++it)
        res.push_back(s+'+'+it->second);
      rg = e_minus[k].equal_range(target-sum);
      for (auto it = rg.first; it != rg.second; ++it)
        res.push_back(s+'-'+it->second);
    }
  }
public:
  vector<string> addOperators(string num, int target) {
    a = num;
    n = a.size();
    if (n) {
      nn = n/2; // 0 < nn < n
      this->target = target;
      e_plus.resize(n);
      e_minus.resize(n);
      backward(n-1, string(1, a[n-1]), 0, 1, a[n-1]-'0', 1);
      forward(1, string(1, a[0]), 0, 1, a[0]-'0');
    }
    return res;
  }
};
```
