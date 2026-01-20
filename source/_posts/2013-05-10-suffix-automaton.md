---
layout: post
title: "String index structure: suffix automaton"
author: MaskRay
tags: [data structure, stringology, suffix automaton]
mathjax: true
---

## Suffix automaton

Deterministic acyclic finite state automaton，也叫directed acyclic word graph(DAWG)，是一個識別若幹字串的無環DFA。識別若干字串的所有DAWG中，存在唯一一個狀態數最少的DAWG，稱作minimal DAWG，即識別這個字串集的DFA。

DAWG在有些文獻中也指識別一個字串的所有後綴的自動機，這種用法有個更常見的名稱：suffix automaton，此時該自動機可以看做是字串所有後綴的DAWG。

<!-- more -->

## 定義

- Factor，即子串。

- $\mathit{factor}(z)$，$z$的所有子串。比如$\mathit{factor}(ab)=\{\epsilon,a,b,ab\}$。

- $\mathit{suff}(z)$表示$z$的所有後綴的集合。比如$\mathit{suff}(abc)=\{abc,bc,c,\epsilon\}$。

- $R_w(z)$表示$w$中$z$所有出現位置右邊的串的集合。比如對於$w="abcde"$，其$R_w("bc")="de"$。

- $u \equiv_w v$表示$w$中$u$和$v$的出現位置集合相同，即$R_w(u)=R_w(v)$。

- $s_w(z)$爲串$w$中子串$z$的suffix function：$z$的最長後綴$y$，滿足$y \not\equiv_w z$。
$s_w(\epsilon)$無定義。當$s_w$定義在$w$的所有後綴上時就得到了suffix link，因此suffix function是suffix link的推廣，用於處理factor的集合。

- 用上標表示函數應用的次數，即$s_w^0(z)=z$，$s_w^1(z)=s_w(z)$，$s_w^2(z)=s_w(s_w(z))$……

## 一些引理

在考慮$w$時，對於$w$的兩個子串$u$和$v$，定義$u$和$v$在一個集合中當且僅當$R_w(u)\equiv R_w(v)$，把$R_w(u)$看作一個狀態。
添加一個新字符後，易知會產生一個新狀態$R_{wa}=\{\epsilon\}$，因爲它與已有的集合都不相同。
下面的三條引理說明了添加字符後，$R_{w}$集合是如何演變爲$R_{wa}$的，從而知道大多數狀態不會發生變化，狀態不會合併，且最多隻有一個狀態會分裂成兩個。

### 引理1

給$w$添加字符$a$後$R_{wa}$的變化：
$$
R_{wa}(u) = \begin{cases}
  \{\epsilon\} \cup R_w(u), & \text{if } u \text{ is a suffix of } wa\\
  R_w(u), & \text{otherwise}
\end{cases}
$$

對於$w$的一個子串$x$，若$x$的最後一個字符不是$a$，則添加字符後$x$所屬狀態沒有發生變化。

### 引理2

令$z$爲在$w$中出現過的$wa$的最長後綴，那麼對於$w$的任意兩個子串$u,v$：
$$
u \equiv_w v, not (u \equiv_w z) \implies u \equiv_{wa} v
$$

證明：

- 若$u \in \mathit{suff}(wa)$，則$u$是$w$中出現過的$wa$的一個後綴，由$z$定義知$u \in \mathit{suff}(z)$。
- 存在最大的非負整數$i$使得$|u| \leq |s_w^i(z)|$，
  由$s_w$性質有$u \equiv_w s_w^i(z)$，
  又$u \equiv_w v$，故$v \equiv_w s_w^i(z)$。
  因此$v \in \mathit{suff}(z)$即$v \in \mathit{suff}(wa)$。
- 因此$u \in \mathit{suff}(wa) \implies v \in \mathit{suff}(wa)$。對稱地，$v \in \mathit{suff}(wa) \implies u \in \mathit{suff}(wa)$。
  所以$u$和$v$同時在$\mathit{suff}(wa)$中或同時不在。
- 根據引理1，即$R_{wa}$與$R_w$的關係知$R_{wa}(u)=R_{wa}(v)$。

這條引理說明，對於$wa$的一個後綴$x$，若$x \not\equiv_w z$，則添加字符後$x$所屬狀態沒有發生變化。

### 引理3

依舊令$z$表示在$w$中出現過的$wa$的最長後綴。

下面的引理則說明，$w$中與$z$等價的集合，在$w$末尾添加新字符$a$後是怎樣變化的：

- $u \equiv_w z, |u| \leq |z| \implies u \equiv_{wa} z$。

    + 由引理1和$z \in \mathit{suff}(wa)$知$R_{wa}(z) = \{\epsilon\} \cup R_{w}(z)$
    + 由$|u| \leq |z|$知$u \in \mathit{suff}(z)$，故$u \in \mathit{suff}(wa)$。因此$R_{wa}(u) = \{\epsilon\} \cup R_{u}$
    + 前兩式得$R_{wa}(z) = R_{wa}(u)$

- $u \equiv_w z, |u| > |z| \implies u \equiv_{wa} z'$，其中$z'$為滿足$z' \equiv_w z$的$w$中最長子串，且不是$wa$的後綴。

    + $u$和$z'$都不是$wa$的後綴
    + $R_{wa}(u) = R_{w}(u) = R_w(z') = R_{wa}(z')$

如果第二個分類不爲空，則$w$中與$z$等價的集合會分裂成兩個集合：$z$的等價類和$z'$的等價類，從而多出一個狀態。

## 性質

長度$n$爲1的串的suffix automaton，其狀態數爲2。新增字符$a$後，會多出一個新狀態$R_{wa}$。根據引理3，$z$所屬狀態可能會分裂成兩個狀態。因此添加一個字符會使狀態數增加1或2。對於$n>1$，suffix automaton的狀態數$\mathit{st}(y)\le 3+2(n-2)=2n-1$。而狀態數的下界爲$n+1$，當串$y$爲全`a`時即可取到下界。

另外對於$n>2$，邊數的下界爲$n$，上界爲$3n-4$。

記$f_y(p)$爲$p$節點的suffix function，它表示$s_y(u)$的等價類，其中$u$是$p$表示的等價類中的任意一個串。

## 構建

採用一個在線算法構建字串$y$的suffix automaton，從從空串對應的suffix automaton(僅含一個頂點)出發，逐個添加$y$中的字符。維護變量$\mathit{last}$表示最後添加的頂點，它表示當前已添加的字串$w$的所屬狀態。
每個狀態$x$記錄$f(x)$，表示$x$表示的串的suffix function對應的狀態。另外記錄$l(x)$，表示$x$的等價類中最長串的長度。

添加字符$a$時有下面兩種情況：

- $a$未在$w$中出現
  只多了一個狀態$R_{wa}$，該狀態的suffix function爲起始狀態。
  $w$所有後綴的狀態都應添加在字符$a$上的轉移，指向新狀態。方法是從$\mathit{last}$出發，追溯$\mathit{SP}(\mathit{last})$得到的所有狀態都添加指向新狀態的邊。

- $a$在$w$中出現
  從$\mathit{last}$出發，追溯$\mathit{SP}(\mathit{last})$，直到遇到第一個頂點$p$存在字符$a$上的轉移，設轉移到$q$。
  $p$表示的最長串(長度爲$l(p)$)後添加$a$就是$z$(長度爲$l(p)+1$)。
  $q$的等價類則表示$w$中與$z$等價的所有$u$。
  然後再分兩種情況：

  + 若$l(p)+1=l(q)$，則$z$的長度大於等於$w$中與$z$等價的任一$u$。$z$的等價類(表示爲狀態$q$)不會分裂。Suffix automaton只多了一個表示$wa$等價類的狀態$x$，$l(x)=|wa|$，其$R_{wa}(wa)=\{\epsilon\}$，包含與$R_{wa}(q)=\{\epsilon\} \cup R_{w}(q)$，因此設置$f(x)=q$。
  + 若$l(p)+1<l(q)$，則存在$u \equiv_w z, |u|=l(q)+1 > l(p)+1=|z|$，$q$需要拆分爲兩個狀態，分別表示$z'$(狀態$q$)和$z$(新狀態$v$)，$v$繼承$q$的所有邊，$l(v)=l(p)+1$，$f(q)$需要修改爲$v$。另外還要沿着$SP(p)$，把沿路狀態在$a$上的轉移改爲$v$。

稍作修改該算法即可用於多串suffix automaton的構建。處理新字串時，把$\mathit{last}$重置爲起始狀態，每次添加字符時，轉移至的狀態可能已在automaton中，無需創建。添加完字符後，$\mathit{last}$可能有兩種取值，需要注意。

```c++
struct Node
{
  Node *f, *c[26];
  int len;
} pool[2*N], *tail, *pit;

void extend(int c)
{
  bool created = false;
  Node *p = tail, *x;
  if (p->c[c])
    x = p->c[c];
  else {
    created = true;
    x = pit++;
    x->len = p->len+1;
    for (; p && ! p->c[c]; p = p->f)
      p->c[c] = x;
  }
  if (! p)
    x->f = pool;
  else if (p->len+1 == p->c[c]->len) {
    if (created)
      x->f = p->c[c];
  } else {
    Node *q = p->c[c], *r = pit++;
    *r = *q;
    r->len = p->len+1;
    for (; p && p->c[c] == q; p = p->f)
      p->c[c] = r;
    q->f = r;
    if (created)
      x->f = r;
    else
      x = r;
  }
  tail = x;
}

{
  // string 0
  tail = pool;
  extend(0); extend(1); extend(2);

  // string 1
  tail = pool;
  extend(0); extend(2); extend(2);

  // string 2
  tail = pool;
  extend(2); extend(1); extend(1);
}
```

## 性質

- 另$SP(p)$表示從$p$開始，沿著$f$函數回溯得到的所有狀態，accepting states是$SP(last)$。

- Suffix automaton中所有邊$(u, s_w(u))$形成一棵spanning tree。

- $|s_w(u)| < |u|$，所以對於suffix automaton中任意頂點$p$，$l(f(p)) < l(p)$

- 對於邊$(p,q)$，$l(p)+1 \leq l(q)$，所以可以對所有頂點的$l$值做計數排序，達到拓撲排序的效果。

## 輔助數據結構

設$p=\delta(q_0, x)$有定義(在自動機中)，

$maxContext(p)$表示$p$對應的$R_y(x)$集合中的最長串長度，即第一次出現的位置的右邊串長度。計算方式如下：

$$maxContext(p)=\begin{cases}
  0, & \text{if } deg(p) = 0 \\
  \max_{(p,q) \in F}{1 + maxContext(q)}, & otherwise
\end{cases}$$

$n-maxContext(p)-|x|$是$x$第一次出現的位置。

$minContext(p)$表示$p$對應的$R_y(x)$集合中的最短串長度，即最後一次出現的位置的右邊串長度。計算方式如下：

$$minContext(p)=\begin{cases}
  0, & \text{if } p \in [last,f(last),f^2(last),\ldots] \\
  \min_{(p,q) \in F}{1 + minContext(q)}, & otherwise
\end{cases}$$

$n-minContext(p)-|x|$是$x$最後一次出現的位置。

$$cnt(p)=\begin{cases}
  1, & \text{if } deg(p) = 0 \\
  \sum_{(p, q) \in F}{cnt(q)}, & otherwise
\end{cases}$$

$cnt(p)$是$x$在串裏的出現次數。

對所有狀態拓撲排序後，根據其逆序計算即可求出各個狀態的$maxContext$、$minContext$和$cnt$值。

## 應用

- 列出模式串在文本串中所有匹配位置(all occurrences)。
- 求出最長的出現至少$k$次的子串。尋找最深的節點$x$滿足$cnt(x)>=k$，其深度即爲答案。
- Ending factors。給定兩個串$x$和$y$，對於$x$的所有前綴$p$，求出$p$的最長後綴爲$y$的子串。可用於計算兩串的最長公共子串。
- Searching for rotations。給定兩個串$x$和$y$，$x$的任意rotation(或稱爲conjugate)在$y$中出現的位置。技巧是把$x$和自身拼接，得到$xx$，它包含$x$的$|x|$個rotation，轉化爲$xx$和$y$的ending factors問題。
- $n$個串的最長公共子串。求$n-1$次ending factors。

## 相關數據結構

### Factor automaton

Suffix automaton有個類似的數據結構，稱作factor automaton，它識別一個字串的所有子串，可以看作識別所有子串的minimal DAWG。

求出$S$的suffix automaton後，把所有狀態標記為accept state後就得到了識別所有子串的DAWG，但可能不是最簡的，做minimize即可得到factor automaton，而acyclic DFA可以用Revuz's algorithm在$O(狀態數+邊數+|字符集|)$時間內求最簡表示。

反過來，可以通過factor automaton獲得suffix automaton。參見Maxime Crochemore、Christophe Hancart的《Automata for Matching Patterns》中的FA-to-SA。

### Suffix oracle和factor oracle

Suffix oracle是一種簡化的suffix automaton。

某次添加字符$a$時，令$z$表示在$w$中出現過的$wa$的最長後綴，如果存在$u \equiv_w z, |u| > |z|$，那麼suffix automaton中會進行狀態分裂，而factor oracle不分裂，相當於把兩個狀態合併了。

Suffix oracle可以看作是suffix automaton合併了一些狀態後得到的自動機。其接受的串的集合是suffix automaton接受的串的集合的超集，也就是說它不會遺漏suffix automaton接受的串，但可能會有誤判。

Suffox oracle的優點是節點數為$n+1$，比最壞情況下的suffix automaton大致少一半。邊數爲$n$到$2n-1$。

把suffix oracle的所有狀態標記爲可接受即得到factor oracle，且已是最簡形態，無法minimize。

## 其他類似數據結構

### Position heap

比suffix tree簡單，存在一個$O(n\log n)$離線構造算法，但實現複雜度不比suffix automaton低。


## 參考文獻

Allauzen C., Crochemore M., Raffinot M., Factor oracle: a new structure for pattern matching; Proceedings of SOFSEM’99; Theory and Practice of Informatics.

@article{Revuz:1992:MAD:135853.135907,
 author = {Revuz, Dominique},
 title = {Minimisation of Acyclic Deterministic Automata in Linear Time},
 journal = {Theor. Comput. Sci.},
 issue_date = {Jan. 6, 1992},
 volume = {92},
 number = {1},
 month = jan,
 year = {1992},
 issn = {0304-3975},
 pages = {181--189},
 numpages = {9},
 url = {http://dx.doi.org/10.1016/0304-3975(92)90142-3},
 doi = {10.1016/0304-3975(92)90142-3},
 acmid = {135907},
 publisher = {Elsevier Science Publishers Ltd.},
 address = {Essex, UK},
}
