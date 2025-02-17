layout: post
title: Ko-Aluru suffix array construction algorithm
author: MaskRay
tags: [algorithm, data structure, stringology, suffix array]
mathjax: true
---

Suffix array是解決string search問題的一個工具，它是一個字串所有後綴的排序。

線性時間的構建算法主要有Kärkkäinen-Sanders([^ks])、Ko-Aluru([^ka])、KSP三種，其中KSP算法使用了類似suffix tree的構建方式，Kärkkäinen-Sanders算法時間複雜度的遞推式爲$T(n)=T(\frac{2n}{3})+O(n)$，通常認爲性能不如$T(n)=T(\frac{n}{2})+O(n)$的Ko-Aluru。

按照[^yangzhe]的註記，借鑑[^nzc]對LMS substring遞歸排序使用的方法，
對原始的Ko-Aluru算法進行改動以簡化內存使用和提升性能。
我的實現在字串和suffix array外僅使用兩個數組：計數排序使用的桶和一個表示L/S類型的數組，
反映了Ko-Aluru算法能達到和[^nzc]同等的精簡內存佔用。

<!-- more -->

## Ka-Aluru suffix array construction algorithm

規定$S[n]$爲哨兵字符，且小於字符集中任意一個字符。用$S_i$表示後綴$S[i]S[i+1]S[i+2]\ldots S[n-1]$。
對於後綴$S_i$，根據$S_i$和$S_{i+1}$的大小關係得到後綴$S_i$的類型$T[i]$，然後根據$T$把所有後綴分爲兩類：

- $C^- = \{i\in [0,n) | S_i > S_{i+1}\}$
- $C^+ = \{i\in [0,n) | S_i < S_{i+1}\}$

$C^-$也稱爲L類，$C^+$也稱爲S類。

令字符集爲$[0,\sigma)$，再根據後綴首字母分類：

$$C_a = \{i\in [0,n) | S[i] = a\}$$

並結合以上兩種分類定義：

* $C_a^- = \{i\in [0,n) | S[i] = a \text{ and } S_i > S_{i+1}\}$
* $C_a^+ = \{i\in [0,n) | S[i] = a \text{ and } S_i < S_{i+1}\}$

對於$i<j$，$C_i$中的後綴排在$C_j$中後綴的前面，因爲如果兩個後綴$S_i$和$S_j$的第一個字符$S[i]$、$S[j]$不同，那麼$S_i$和$S_j$的大小關係可以直接確定。另外$C_i^-$中的後綴排在$C_i^+$中後綴的前面，因爲$C_i^-$中後綴均小於$S[i]S[i]\ldots$(重複的$S[i]$)，而$C_i^+$中後綴均大於$S[i]S[i]\ldots$。

根據$C_*^*$可以把suffix array粗略劃分爲若干個桶，每個桶內的後綴都小於右邊的桶內的任意後綴。最多有$2\sigma$個桶：$C_0^-C_0^+C_1^-C_1^+C_2^-C_2^+\ldots C_{\sigma-1}^-C_{\sigma-1}^+$。

根據[^ka]的induced sorting，只要確定了$C^-$的大小順序，那麼$C^+$的順序也可以確定。同樣地，只要確定了$C^+$的大小順序，那麼$C^-$的順序也可以確定。根據平均數原理$|C^-|$和$|C^+|$至少有一個$\leq\frac{n}{2}$，不妨設$|C^-|\leq\frac{n}{2}$，只要求出$C^-$中所有後綴的大小順序，那麼使用induced sorting即可得到$C^+$的大小順序。
再根據$C_i^-$中的後綴排在$C_i^+$中後綴的前面，即可得到所有後綴的大小順序。

下面考慮如何遞歸求出$C^-$的大小順序。[^ka]使用了一個複雜的方法，根據$S$-distance計算。
借鑑[^nzc]對LMS substring遞歸排序使用的方法，我在實現中使用了一個簡單的方法。

定義L-substring爲夾在相鄰兩個L(-)類型字符間的子串。
定義S-substring爲夾在相鄰兩個S(+)類型字符間的子串。
定義一個L-substring $[i,j]$小於另一個L-substring $[k,l]$當且僅當$S[i],T[i],S[i+1],T[i+1],\ldots,S[j],T[j]$字典序小於$S[k],T[k],S[k+1],T[k+1],\ldots,S[l],T[l]$，其中$T[i]$兩種取值規定L小於S。

對於兩個長度不同的L-substring $[i,j]$和$[k,l]$，它們不會相等，且根據L-substring的大小關係可以得到$S_i$和$S_k$的大小關係。如果兩L-substring相等，找到$[i,j]$在原串後的L-substring $[j,p]$，兩者共用一個字符。再找到$[k,l]$在原串後的L-substring $[l,q]$，$[j,p]$和$[l,q]$的大小關係即爲$[i,j]$和$[k,l]$的大小關係。如果這兩個子串也相同了，那麼可以繼續找緊接着的下一對L-substring。根據以上比較方式可以發現如果取出$S$中所有L-substring，並把這些L-substring看作字符，就可以得到子問題遞歸調用算法解決。現在考慮如何確定所有L-substring的大小關係，使得長度不同的可以區分大小，而長度相同的允許相等(所以才需要遞歸確定大小)。

先用桶排序求出所有L類型字符的大小，使用一次induced sorting根據L計算S，這樣便得到了所有形如${}^+{}^+{}^+\cdots{}^-$子串的大小。再用一次induced sorting根據S計算L，便得到了所有形如${}^-{}^-{}^-\cdots{}^+{}^+{}^+\cdots{}^-$類型子串的大小，而這些串就是我們需要的L-substring。對L-substring計算新字符集大小，若等於L-substring的數目則說明所有L-substring的順序已經確定，否則取出L字符遞歸計算suffix array。得到這些L-substring的順序即$C^-$的順序後使用induced sorting即可得到整個suffix array。

## 實現技巧

$S$爲原串，$T$爲類型數組，$B$爲桶排序使用的桶。

整個實現分爲幾個階段，仍然設$|C^-|\leq \frac{n}{2}$：

- 計算$T$數組，需要使用$S$
- 使用兩次induced sorting計算L-substring的大小，需要使用$S,T,B$
- 根據L-substring構建新字串遞歸計算子suffix array
- 根據子suffix array計算$C^-$部分的suffix array
- 使用induced sorting根據$C^-$部分的suffix array得到整體suffix array

其中第2、3步需要特別注意，使用了很多技巧來節省空間佔用。比如新串的長度不大於$\frac{n}{2}$，因此可以和臨時數組共享`sa[]`數組的空間，遞歸調用後對`sa[]`進行原地操作進行位置變換。

注意到$|C^-| \leq \frac{n}{2}$，因此新字串和子suffix array的空間佔用和$\leq n$，可以讓它們共用原始的suffix array的空間。這樣根據子suffix array計算$C^-$部分的suffix array的過程變成了把suffix array擴張的過程，而擴張時需要根據新字串和$P$的字符對應關係的映射進行，因爲映射的性質可以共用一個數組實現，擴張中suffix array儲存了三部分的內容：映射、子suffix array、$C^-$部分的suffix array，其中$C^-$部分的suffix array漸漸變大，子suffix array漸漸變小。求得$C^-$部分的suffix array後，原地桶排序得到原始下標的$C^-$。遞歸也需要使用$T$數組，因爲$T$計算是線性的，可以讓遞歸時繼續使用該數組，遞歸函數返回時重新計算被覆蓋的部分。這樣可以省去$T$數組在遞歸中不斷被創建的開銷。

多數文獻都在$S$末尾添加哨兵元素以方便講解，但個人感覺實現中哨兵元素通常會導致更複雜的判斷，弊大於利。不使用哨兵元素只需要在三個地方注意：

- $T[n-1]$爲L，因爲把哨兵字符看作字典序最小。
- 根據S-substring計算L-substring的induced sorting過程提前把$S[n-1]$放入桶中。否則因爲它是最後一個字符，無法被放入桶中，當該字符的桶包含兩個或更多元素時可能會導致它的位置被覆蓋而出錯。
- 計算新字符集時，需要注意比較過程中到達邊界的情況。

Ko-Aluru算法的僞代碼如下：

```text
fn ka S sa n
  計算T
  確定少數類型minor: -或+

  if minor爲L(-)
    計數排序S中L(-)類型的字符，S[n-1]需要提前放
    用induced sorting根據L(-)計算S(+)，即確定了所有形如++..+-的串的大小
    用induced sorting根據S(+)計算L(-)，即確定了所有形如--..-++..+-的串的大小
  else
    類似

  把依序的各個L-substring作爲新的串
  若新字符集大小小於新串長度則遞歸調用，計算新串的sa2[]

  把sa2[]的值(新串的位置)映射爲原串的位置
  if minor爲L(-)
    把sa2[]的元素移動到字符代表的桶的前部
    用induced sorting根據S(+)計算L(-)，得到桶的後部
  else
    把sa2[]的元素移動到字符代表的桶的後部
    用induced sorting根據L(-)計算S(+)，得到桶的前部
```

## 實現

C++：

```cpp
#include <algorithm>
#include <cstdio>
#include <cstring>
using namespace std;

#define REP(i, n) for (int i = 0; i < (n); i++)
#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define ROF(i, a, b) for (int i = (b); --i >= (a); )

namespace KoAluru
{
  bool *t;
  int *b;

  template<typename T>
  void bucket(T a[], int n, int k, bool end)
  {
    fill_n(b, k, 0);
    REP(i, n) b[a[i]]++;
    if (end)
      FOR(i, 1, k) b[i] += b[i-1];
    else {
      int s = 0;
      REP(i, k)
        s += b[i], b[i] = s-b[i];
    }
  }

  template<typename T>
  void plus_to_minus(T a[], int sa[], int n, int k)
  {
    bucket(a, n, k, false);
    sa[b[a[n-1]]++] = n-1;
    REP(i, n-1) {
      int j = sa[i]-1;
      if (j >= 0 && ! t[j])
        sa[b[a[j]]++] = j;
    }
  }

  template<typename T>
  void minus_to_plus(T a[], int sa[], int n, int k)
  {
    bucket(a, n, k, true);
    ROF(i, 0, n) {
      int j = sa[i]-1;
      if (j >= 0 && t[j])
        sa[--b[a[j]]] = j;
    }
  }

  template<typename T>
  void ka(T a[], int sa[], int n, int k)
  {
    t[n-1] = false;
    ROF(i, 0, n-1)
      t[i] = a[i] < a[i+1] || a[i] == a[i+1] && t[i+1];
    bool minor = 2 * count(t, t+n, false) > n;

    bucket(a, n, k, minor);
    fill_n(sa, n, -1);
    if (minor) {
      REP(i, n)
        if (t[i])
          sa[--b[a[i]]] = i;
      plus_to_minus(a, sa, n, k);
      minus_to_plus(a, sa, n, k);
    } else {
      sa[b[a[n-1]]++] = n-1;
      REP(i, n-1)
        if (! t[i])
          sa[b[a[i]]++] = i;
      minus_to_plus(a, sa, n, k);
      plus_to_minus(a, sa, n, k);
    }

    int last = -1, name = 0, nn = count(t, t+n, minor);
    int *sa2, *pi;
    if (minor)
      sa2 = sa, pi = sa+n-nn;
    else
      sa2 = sa+n-nn, pi = sa;
    fill_n(b, n, -1);
    REP(i, n)
      if (sa[i] >= 0 && minor == t[sa[i]]) {
        bool diff = last == -1;
        int p = sa[i];
        if (! diff)
          REP(j, n) {
            if (last+j >= n || p+j >= n || a[last+j] != a[p+j] || t[last+j] != t[p+j]) {
              diff = true;
              break;
            } else if (j > 0 && (minor == t[last+j] || minor == t[p+j]))
              break;
          }
        if (diff) {
          name++;
          last = p;
        }
        b[p] = name-1;
      }
    nn = 0;
    REP(i, n)
      if (b[i] >= 0)
        pi[nn++] = b[i];

    if (name < nn)
      ka(pi, sa2, nn, name);
    else
      REP(i, nn)
        sa2[pi[i]] = i;

    ROF(i, 0, nn)
      t[i] = a[i] < a[i+1] || a[i] == a[i+1] && t[i+1];

    nn = 0;
    bucket(a, n, k, minor);
    REP(i, n)
      if (minor == t[i])
        pi[nn++] = i;
    if (minor) {
      REP(i, nn)
        sa[i] = pi[sa2[i]];
      ROF(i, 0, nn) {
        int j = sa[i];
        sa[i] = -1;
        sa[--b[a[j]]] = j;
      }
    } else {
      ROF(i, 0, nn)
        sa[n-nn+i] = pi[sa2[i]];
      REP(i, nn) {
        int j = sa[n-nn+i];
        sa[n-nn+i] = -1;
        sa[b[a[j]]++] = j;
      }
    }
    if (minor)
      plus_to_minus(a, sa, n, k);
    else
      minus_to_plus(a, sa, n, k);
  }

  template<typename T>
  void main(T a[], int sa[], int b[], int n, int k)
  {
    KoAluru::b = b;
    if (n > 0) {
      t = new bool[n];
      ka(a, sa, n, k);
      delete[] t;
    }
  }

  template<typename T>
  void calc_rank_lcp(T a[], int sa[], int n, int rank[], int lcp[])
  {
    REP(i, n)
      rank[sa[i]] = i;
    int k = 0;
    lcp[0] = 0;
    REP(i, n)
      if (rank[i]) {
        for (int j = sa[rank[i]-1]; i+k < n && j+k < n && a[i+k] == a[j+k]; k++);
        lcp[rank[i]] = k;
        k && k--;
      }
  }
};

const int N = 1992;
unsigned char a[N+1];
int sa[N], rnk[N], lcp[N], q[N];

int main()
{
  gets((char *)a);
  int n = int(strlen((char *)a));
  KoAluru::main(a, sa, q, n, 256);
  KoAluru::calc_rank_lcp(a, sa, n, rnk, lcp);

  puts("sa");
  REP(i, n)
    printf("%d: %d\n", i, sa[i]);
  puts("rank");
  REP(i, n)
    printf("%d: %d\n", i, rnk[i]);
  puts("lcp");
  REP(i, n)
    printf("%d: %d\n", i, lcp[i]);
}
```

OCaml：

```ocaml
module KoAluru = struct
  (** [k]: Elements of [a] are in \[0,k) *)
  let ka a k =
    let n = Array.length a in
    let t = Array.make n false in
    let b = Array.make (max n k) 0 in
    let sa = Array.make n 0 in

    let bucket ai n k minor =
      Array.fill b 0 k 0;
      for i = 0 to n-1 do
        b.(a.(ai+i)) <- b.(a.(ai+i)) + 1
      done;
      if minor then
        for i = 1 to k-1 do
          b.(i) <- b.(i) + b.(i-1)
        done
      else
        let s = ref 0 in
        for i = 0 to k-1 do
          let t = !s in
          s := !s + b.(i);
          b.(i) <- t
        done
    in

    let minus_to_plus ai sai n k =
      bucket ai n k true;
      for i = n-1 downto 0 do
        let j = sa.(sai+i) - 1 in
        if j >= 0 && t.(j) then (
          b.(a.(ai+j)) <- b.(a.(ai+j)) - 1;
          sa.(b.(a.(ai+j))) <- j
        )
      done
    in

    let plus_to_minus ai sai n k =
      bucket ai n k false;
      sa.(b.(a.(ai+n-1))) <- n-1;
      b.(a.(ai+n-1)) <- b.(a.(ai+n-1)) + 1;
      for i = 0 to n-1 do
        let j = sa.(sai+i) - 1 in
        if j >= 0 && not t.(j) then (
          sa.(b.(a.(ai+j))) <- j;
          b.(a.(ai+j)) <- b.(a.(ai+j)) + 1
        )
      done
    in

    let rec ka_iter ai sai n k =
      (* calculate t[] *)
      t.(n-1) <- false;
      let minus_cnt = ref 1 in
      for i = n-2 downto 0 do
        t.(i) <- a.(ai+i) < a.(ai+i+1) || a.(ai+i) = a.(ai+i+1) && t.(i+1);
        if not t.(i) then
          incr minus_cnt
      done;
      let minor = !minus_cnt > n - !minus_cnt in
      let minor_cnt = if minor then n - !minus_cnt else !minus_cnt in

      (* use two induced sorting to order L/S-substrings, identical L/S-substrings are ordered arbitrarily *)
      bucket ai n k minor;
      Array.fill sa sai n (-1);
      if minor then (
        for i = 0 to n-1 do
          if t.(i) then (
            b.(a.(ai+i)) <- b.(a.(ai+i)) - 1;
            sa.(b.(a.(ai+i))) <- i
          )
        done;
        plus_to_minus ai sai n k;
        minus_to_plus ai sai n k
      ) else (
        for i = 0 to n-1 do
          if not t.(i) then (
            sa.(b.(a.(ai+i))) <- i;
            b.(a.(ai+i)) <- b.(a.(ai+i)) + 1
          )
        done;
        minus_to_plus ai sai n k;
        plus_to_minus ai sai n k
      );

      (* label L/S-substrings, identical L/S-substrings share the same label *)
      Array.fill b 0 n (-1);
      let last = ref (-1)
      and name = ref 0 in
      for i = 0 to n-1 do
        if sa.(i) >= 0 && minor = t.(sa.(sai+i)) then (
          let p = sa.(sai+i) in
          let rec check_lsl j =
            if !last+j >= n || p+j >= n || a.(!last+j) <> a.(p+j) || t.(!last+j) <> t.(p+j) then
              true
            else if j > 0 && (minor = t.(!last+j) || minor = t.(p+j)) then
              false
            else
              check_lsl (j+1)
          in
          if !last = -1 || check_lsl 0 then (
            incr name;
            last := p
          );
          b.(p) <- !name - 1
        )
      done;

      (* calculate SA of L/S-substrings *)
      let pi = if minor then sai+n-minor_cnt else sai in
      let sa2i = if minor then sai else sai+n-minor_cnt in
      let rec collect_lsl nn i =
        if i = n then
          nn
        else (
          if b.(i) >= 0 then (
            sa.(pi+nn) <- b.(i);
            collect_lsl (nn+1) (i+1)
          ) else
            collect_lsl nn (i+1)
        )
      in
      let nn = collect_lsl 0 0 in
      if !name < nn then
        ka_iter pi sa2i nn !name
      else
        for i = 0 to nn-1 do
          sa.(sa2i+sa.(pi+i)) <- i
        done;

      (* restore destroyed t[] *)
      for i = nn-1 downto 0 do
        t.(i) <- a.(ai+i) < a.(ai+i+1) || a.(ai+i) = a.(ai+i+1) && t.(i+1)
      done;

      (* transform calculated sa2[] *)
      let rec collect_minor_indices nn i =
        if i = n then
          nn
        else if minor = t.(i) then (
          sa.(pi+nn) <- i;
          collect_minor_indices (nn+1) (i+1)
        ) else
          collect_minor_indices nn (i+1)
      in
      let nn = collect_minor_indices 0 0 in
      for i = 0 to nn-1 do
        sa.(sa2i+i) <- sa.(pi+sa.(sa2i+i))
      done;

      (* move transformed sa2[] to front/rear of buckets *)
      bucket ai n k minor;
      if minor then (
        for i = nn-1 downto 0 do
          let j = sa.(i) in
          sa.(i) <- -1;
          b.(a.(ai+j)) <- b.(a.(ai+j)) - 1;
          sa.(b.(a.(ai+j))) <- j
        done;
        plus_to_minus ai sai n k
      ) else (
        for i = 0 to nn-1 do
          let j = sa.(sa2i+i) in
          sa.(sa2i+i) <- -1;
          sa.(b.(a.(ai+j))) <- j;
          b.(a.(ai+j)) <- b.(a.(ai+j)) + 1
        done;
        minus_to_plus ai sai n k;
      )
    in

    ka_iter 0 0 n k;
    sa

  let calc_rank_lcp a sa =
    let n = Array.length a in
    let rank = Array.make n 0 in
    let lcp = Array.make n 0 in
    for i = 0 to n-1 do
      rank.(sa.(i)) <- i
    done;
    let rec go k i =
      if i < n then
        if rank.(i) > 0 then (
          let j = sa.(rank.(i)-1) in
          let rec go2 k =
            if i+k < n && j+k < n && a.(i+k) = a.(j+k) then
              go2 (k+1)
            else
              k
          in
          let k = go2 k in
          lcp.(rank.(i)) <- k;
          go (k-1 |> max 0) (i+1)
        ) else
          go (k-1 |> max 0) (i+1)
    in
    go 0 0;
    rank, lcp
end

let () =
  let a = [| 1;1;2;1;2;3;2;3;1;3;0 |] in
  let n = Array.length a in
  let sa = KoAluru.ka a 4 in
  let rank,lcp = KoAluru.calc_rank_lcp a sa in
  for i = 0 to n-1 do
    Printf.printf "%d " rank.(i)
  done
```

## 參考文獻

[^ka]: Pang Ko and Srinivas Aluru. “Space efficient linear time construction of suffix arrays”. In: Journal of Discrete Algorithms. Springer, 2003, pp. 200–210.
[^ks]: Juha Kärkkäinen and Peter Sanders. “Simple Linear Work Suffix Array Construction”. English.  In: Automata, Languages and Programming. Ed. by JosC.M. Baeten et al. Vol. 2719. Lecture Notes in Computer Science. Springer Berlin Heidelberg, 2003, pp. 943–955. isbn: 978-3-540-40493-4. doi: 10.1007/3-540-45061-0_73. url: <http://dx.doi.org/10.1007/3-540-45061-0_73>.
[^nzc]: Ge Nong, Sen Zhang, and Wai Hong Chan. “Two Efficient Algorithms for Linear Time Suffix Array Construction”. In: IEEE Transactions on Computers 60.10 (2011), pp. 1471–1484. issn: 0018-9340. doi: <http://doi.ieeecomputersociety.org/10.1109/TC.2010.188>.
[^yangzhe]: YANG Zhe. Suffix Array Construction: KA's algorithm and Induced Sorting, which is faster? 2012. url: <http://yangzhe1990.wordpress.com/2012/11/04/ka-and-induced-sortingwhich-is-faster/>.
