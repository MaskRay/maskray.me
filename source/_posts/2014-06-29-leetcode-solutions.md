layout: post
title: LeetCode solutions
author: MaskRay
tags: [algorithm, leetcode]
mathjax: true
---

You have solved 489/1771 problems.
主要追求兩個目標：代碼簡短，時間空間複雜度低。

## 註記

### 4-sum

使用了`multimap`，時間複雜度$O(n^2\log{n})$，得到所有答案後不需要去重操作。

合法的四元組有三類：

- `a<=b<c<=d`，枚舉`c`和`d`，判斷是否存在和爲`target-c-d`的二元組
- `a<=b=c<=d`，枚舉`b`和`d`，判斷是否存在`target-b-b-d`
- `a=b=c<=d`，枚舉`a`，判斷是否存在`target-a-a-a`

分別統計，小心實現可以保證不會產生相同的候選解，從而無需去重。

### Alien Dictionary

對於每一對單詞，可以確定去除最長公共前綴後的下一對字母的大小關係。兩個單詞的最長公共前綴等於夾在其中的(所有相鄰單詞對的最長公共前綴)的最小值。根據相鄰單詞對的信息即可推得所有任意單詞對的信息。因此只需根據相鄰單詞對求出拓撲關係。

### Basic Calculator

這是operator-precedence grammar，可以用operator-precedence parser計算。這類算法的一個特例是shunting-yard algorithm，適用於本題。
爲了方便，我假設字串開頭和末尾都有隱含的'\0'字符。使用的文法如下：

```
S := %x00 E %x00
E := E "+" E
E := E "-" E
E := ( E )
E := 1*DIGIT
```

參考<http://www.scribd.com/doc/51358638/16/Operator-Precedence-Relations>。

在比較兩個操作符時，區分左手邊和右手邊，左手邊用於先處理的操作符，右手邊爲新來的操作符。使用左手邊操作符的in-stack precedence(isp)和右手邊操作符的incoming precedence(icp)進行比較。立即數可以看作icp很大，因此shift後可以立刻reduce。

### Best Time to Buy and Sell Stock IV

線性解法。見[2015-03-27-leetcode-best-time-to-buy-and-sell-stock-iv]()。

### Closest Binary Search Tree Value II

題意：求BST中與目標值最接近的k個節點。
BST中關鍵字小於等於目標值的所有節點可以用若干(不帶右孩子的子樹)表示。這些(不帶右孩子的子樹)在一條鏈上，可以用棧維護，得到用於計算前驅的的迭代器。求出目標值的k個前驅的時間複雜度爲$O(depth+k)$。
類似地，大於目標值的所有節點也可以組織成一個迭代器。
兩個迭代器做合併操作，獲得前k個最接近的值。

### Contains Duplicate III

以$t+1$爲單位把元素分桶，若一個桶內包含兩個元素，則它們的差一定小於等於$t$；差小於等於$t$的數對還可能出現在相鄰兩個桶之間。對於$vector$的各個元素，計算當前元素應放置的桶的編號$j$，檢查三個桶$j-1,j,j+1$，判斷之前$k-1$個元素裏是否有差值小於等於$t$的元素。

### Course Schedule

拓撲排序。入度減爲0的頂點即成爲候選，可以把入度向量組織成單鏈表，從而無需使用額外的隊列或棧存儲候選頂點，減小空間佔用。

### Dungeon Game

動態規劃，對於每個格子記錄從它出發順利到達終點需要的最小血量。從$(m-1,n-1)$遞推至$(0,0)$。

注意對於每個格子其實有進和出兩個狀態：進入該格子時(未考慮當前格子影響)；和離開該格子後(已考慮當前格子影響)。這兩種方法都行，考慮到程序的結構可能有三部分：初始化邊界條件、狀態轉移、根據狀態表示答案，需要仔細斟酌以追求代碼簡短。這裏兩種方式並無明顯代碼長度差異。

### Expression Add Operators

見[2015-10-16-leetcode-expression-add-operators]()。

### Find Minimum in Rotated Sorted Array

方法是二分，如果要寫短可以採用下面的思路：
子數組中如果有相鄰兩個元素`a[i]>a[i+1]`，則`a[i+1]`是整個數組的最小值。
若子數組`a[i..j]`滿足`a[i]>a[j]`，則存在`i<=k<j`使得`a[k]>a[k+1]`。
對於子數組`a[l..h]`判斷`a[l]>a[m]`或`a[m]>a[h]`是否有一個成立，成立則可以去除一半候選元素，不然`a[h]>a[l]`爲一個逆序對。

### Find Minimum in Rotated Sorted Array II

留意幾個特殊情形：`[1,0,1,1,1]`、`[1,1,1,0,1]`。

對於子數組`a[l..h]`，令`m=floor(l+h)/2`(注意`m`可能等於`l`)。判斷`a[l]>a[m]`或`a[m]>a[h]`是否有一個成立，成立則可以去除一半候選元素。
若兩個條件均不滿足則`a[l]<=a[m]<=a[h]`，若`a[h]>a[l]`則說明`a[l]`最小，否則`a[l]=a[m]=a[h]`，可以把範圍縮小爲`a[l+1..h-1]`。

另外一種方法是只判斷`a[m]`與a[h]`的大小關係。

### Find Peak Element

二分查找，把區間縮小爲仍包含候選值的子區間。對於相鄰兩個元素$a$、$b$，若$a>b$則$a$及左邊部分存在peak，否则$b$及右邊部分存在peak。不關心$a$或$b$是否滿足peak條件，將區間折半。
```c++
int l = 0, h = a.size();
while (l < h-1) {
  int m = l+h >> 1;
  if (a[m-1] > a[m]) h = m;
  else l = m;
}
return l;
```

或者檢查中間元素是否滿足要求，可以提前退出循環。
```c++
int l = 0, h = a.size();
while (l < h-1) {
  int m = l+h >> 1;
  if (a[m-1] > a[m]) h = m;
  else if (m+1 == h || a[m] > a[m+1]) l = h = m;
  else l = m+1;
}
return l;
```

### First Missing Possitive

空間複雜度$O(1)$。

### Fraction to Recurring Decimal

注意`INT_MIN/(-1)`會溢出。

### Graph Valid Tree

$n$個節點的樹的判定方式：

- 有$n-1$條邊且沒有simple circle。可以用union-find algorithm判斷。
- 聯通且無simple circle。可以用graph traversal。

### Guess Number Higher or Lower II

動態規劃優化，$O(n^2)$。見張昆瑋的分析：<http://artofproblemsolving.com/community/c296841h1273742s3_leetcode_guess_number_higher_or_lower_ii>。

### Invert Binary Tree

選擇一種binary tree traversal算法，節點訪問改成交換左右孩子即可。
如果要O(1)空間的話，得借鑑Morris pre/in-order traversal的思路，該算法的核心是通過左子樹最右節點的右孩子是否指向當前節點來判斷當前節點被訪問到的次數，從而決定當前應該進行的操作(探索左子樹或訪問當前節點)。但交換子樹操作會弄亂左子樹最右節點的位置，因此pre-order和in-order都行不通，但post-order可行。Morris post-order traversal(這個叫法也許不恰當)可以參考<https://www.quora.com/What-is-a-good-way-to-implement-stackless-recursion-less-post-order-traversal-for-a-non-threaded-binary-tree-using-Morris-method>的描述，pre/in-order的訪問是針對一個節點進行的，但post-order則需要對當前節點左子樹的右鏈進行，並且需要翻轉兩次以達到按post-order輸出的效果。但這裏只需要保證每個節點都被訪問一次，不需要嚴格的post-order順序，因此無需翻轉操作。

### Jump Game

空間複雜度$O(1)$。

### Jump Game II

使用output-restricted queue優化的動態規劃，注意到動態規劃值的單增性可以用類似BFS的方式，空間複雜度$O(1)$。

### Lexicographical Numbers

在trie中找1~n，存在迭代解法。

### Linked List Cycle II

單鏈表找圈。Brent's cycle detection algorithm或Floyd's cycle detection algorithm。

如果只需要判斷是否有圈，而不用指出圈的起點，可以使用pointer reversal。訪問鏈表的過程中翻轉鏈表，倘若訪問到NULL，則說明無圈；若回到了起點，則說明有圈。如果有圈的話，算法執行完畢後，圈的方向會被反轉，可以再執行一次翻轉過程還原。下面是這個方法的實現：

```c++
// return true if a cycle is detected
bool pointerReversal(ListNode *head) {
  if (! head) return false;
  ListNode *x = head->next, *y, *z = head;
  while (x && x != head) {
    y = x->next;
    x->next = z;
    z = x;
    x = y;
  }
  return x == head;
}
```

### Longest Palindromic Substring

線性時間求出以每個位置爲中心的最長迴文子串的Manacher's algorithm。<http://www.csie.ntnu.edu.tw/~u91029/Palindrome.html>提供的描述感覺最清晰。但實現我更喜歡我從某本stringology書上看到的。

### Majority Element

Boyer-Moore majority vote algorithm <http://www.cs.utexas.edu/~moore/best-ideas/mjrty/>，思路是每次找到兩個不同的元素並丟棄，這樣做不會丟失解。容器裏只剩下一種元素時，它就是majority。

### Majority Element II

Boyer-Moore majority vote algorithm的擴展，找出所有出現次數大於$\lfloor N/K \rfloor$的元素。方法是每次找出$K$個互不相同的元素丟掉，最後剩下的元素是候選解。時間複雜度$O(N*K)$。

### Maximum Gap

平均数原理，求出极差$d$后，根据桶大小$\lceil \frac{d}{n-1}\rceil$分成若干个桶，答案必为不同桶的两个元素之差。

### Maximum Subarray

Kadane's algorithm

### Meeting Rooms II

題意中兩條線段衝突當且僅當它們的公共部分長度大於0。
這是interval graph，所求的是chromatic number，interval graph包含與perfect graph，因此chromatic number等於maximum clique頂點數，即最大線段相交數，可以把所有端點排序計算。
或者按起始端點排序後用貪心算法：維護若干線段集合，按起始端點總從小到大考慮每個線段，任選一個不衝突的集合加入，若無不衝突的集合則自成一個新集合。最後，集合數即爲答案。實現時，每個集合用最大右端點值表示，所有集合組織爲一個小根binary heap，若當前考慮線段的起始點小於根則壓入一個新元素，否則修改根的值。

### Merge k Sorted Lists

Tournament sort，可以用`priority_queue`實現。

### Min Stack

使用兩個棧，原棧`S`存放各個元素。每當新元素小於等於當前最小值時，就把新元素複製一份壓入另一個棧`S'`。彈出時，若元素在`S'`中出現，則也從`S'`中彈出。

另一種方法是記錄新元素與當前最小值的差值，每個元素需要多記錄1個bit。可惜C++會Memory Limit Exceeded，感覺不合理。

### Minimum Window Substring

尺取法

### Missing Number

從左到右考慮每個元素，若不等於它的位置則把它交換到正確的位置並重複此過程。

### N Queen

bitmask存儲列，正反斜線控制的格子。

### Number of 1 Bits

求popcount。`__builtin_popcount()`或者網上查找各種bit twiddling hacks。

### One Edit Distance

編輯距離爲1有三種可能：S比T多一個字符、T比S多一個字符、S與T長度相同且有一個位置不同。除去最長公共前綴和最長公共後綴，三種可能的檢查方式可以統一爲判斷較長的串長度是否爲1。

### Palindrome Partitioning

`f[i][j] = calc(f[ii][jj] : i <= ii <= jj <= j)`形式的動態規劃可以採用如下計算方式。

```
for (int i = n; --i >= 0; )
  for (int j = i; j < n; j++) {
    // calc [i,j]
  }
```

### Palindrome Partitioning II

求原串的最少迴文串劃分。O(n^2)時間，可以優化到O(n)空間。

我沒能找到比O(n^2)快的算法。相關的問題有檢測一個串是否分割成k個迴文串。k小於等於4時均有線性算法，參見Text Algorithms一書Theorem 8.17。k大於4時不確定。

### Perfect Rectangle

使用`unordered_map`的$O(n)$解法：判斷除了四個極點，每個小矩形的角是否被兩個或四個矩形共享。

或者判斷所有小矩形面積和等於四個極點圍成的矩形的面積，並且小矩形沒有重疊(用sweep line判斷)。

### Perfect Squares

<https://leetcode.com/discuss/57477/sqrt-applying-fermats-theorm-brahmagupta-fibonacci-identity>

由Lagrange's four-square theorem知答案爲1~4，由Legendre's three-square theorem檢查答案是否爲4。若小於4，則再$O(\sqrt{n})$尺取法檢查是否是perfect square或能表示爲兩個perfect squares之和，若不是則答案爲3。

另有期望$O((\log n)^2)$的算法，參見M. O. Rabin, J. O. Shallit, Randomized Algorithms in Number Theory, Communications on Pure and Applied Mathematics 39 (1986), no. S1, pp. S239–S256. doi:10.1002/cpa.3160390713

### Reconstruct Itinerary

题目即lexicographically smallest Eulerian path。Eulerian path最简单的算法是Hierholzer's algorithm，有向图中仅有一个顶点出度=入度+1，记为源；一个顶点入度=出度+1，记为汇。任求一条源到汇的路径，之后把各个圈合并到路径上即可。本题要求字典序最小，对Hierholzer's algorithm稍加变形即可。

从源开始，贪心地选择序号最小的下一个顶点，删除这条边，重复此过程，即得到一条源到汇的路径。需要把圈添加到该路径上。若一个圈只包含路径上的一个顶点，那么在该顶点处添加圈即可。若一个圈包含路径上的多个顶点，那么应该把它放在离汇最近的顶点上，这样能保证字典序最小。

### Recover Binary Search Tree

使用Morris in-order traversal找到鄰接失序對。

如果交換的元素相鄰，則有一個鄰接失序對(如`0 1 2 3 4 -> `0 1 3 2 4`)，否則有兩個鄰接失序對(如`0 1 2 3 4 5 -> `0 4 2 3 1 5`)。

### Remove Nth Node From End of List

[http://meta.slashdot.org/story/12/10/11/0030249/linus-torvalds-answers-your-questions]()

使用pointers-to-pointers很多時候能簡化實現。

### Regular Expression Matching

`P`爲模式串，`T`爲文本，
`f[i][j]`表示`P`的前`i`個字符能否匹配`T`的前`j`個字符。
根據`f[i-1][*]`計算`f[i][*]`。
這個方法也可以看作構建了精簡表示的Thompson's automaton，時間複雜度$O(|P|\cdot |T|)$。

### Remove Element

原地修改數組，我喜歡把這種操作稱作deflate。

### Remove Invalid Parentheses

最少移除個數很容易計算，把可以配對的左右括號全部抵消，剩餘括號數即是(形如：`)))...(((`)。難點在於輸出所有不同答案，多數解法使用DFS枚舉要刪除哪些括號，枚舉量大於答案個數。
有一個枚舉量固定爲答案個數的方法，找到殘餘串`)))...(((`中右括號與左括號的分割點。DFS分割點左半部分，保證到達殘餘串第$i$個右括號時，至少有$i$個右括號(原串的)被刪除。分割點右半部分也如法炮製，兩邊合併即爲答案。

### Repeated DNA Sequences

可以用類似Rabin-Karp的rolling hash方法，求出所有長爲10的子串的hash值，判斷是否有重複。只有四個字符，長度爲10，故可以使用polynomial hash code。
另外也可以用suffix tree/suffix array等方法，找longest common prefix大於等於10的兩個後綴。注意不能重複輸出字串。

### Reverse Bits

Hacker's Delight (2nd) 7.1 Reversing Bits and Bytes。

### Rotate Array

經典的三次reverse，或者拆成`gcd(k,n)`個置換羣循環右移一格。

### Same Tree

類似於same fringe problem，可以試驗generator、co-routine、lazy evaluation、continuation等。如果要O(1)的空間複雜度可以用Morris in-order traversal等方法。

### Set Matrix Zeroes

$O(1)$空間複雜度。使用兩個標誌表示第0行或第0列是否需要清零，之後用第0行表示各列是否需要清零，第0列表示各行是否需要清零。

### Search For a Range

设序列用区间表示为`[begin,end)`。使用C++ STL風格的`lower_bound、`upper_bound。有几点好处：

- 循环内只有一次比较
- 循环退出后两个指针相等，且落在`[begin,end]`，都是迭代器的合法取值。某些binary search写法返回值可能为`begin-1`，某些容器不合法
- `[lower_bound(a[],x), upper_bound(a[],x)`给出了元素`x`所在的子区间

### Search Insert Position

C++ `<algorithm>`的`lower_bound`。

### Serialize and Deserialize Binary Tree

把NULL視作葉子，則pre/in/post-order traversal可以用於編碼。若使用Morris traversal則額外空間爲$O(1)$。解碼時使用類似Schorr-Waite graph marking algorithm的edge crawling技巧即可做到額外空間$O(1)$。

### Shortest Palindrome

找出原串的最長迴文前綴，用Manacher's algorithm求解，把剩下部分複製一份、翻轉後、接到左邊，就得到了最短的迴文串。求最長迴文前綴的另一種方法是把原串以及翻轉用特殊字符隔開，用Morris-Pratt algorithm求border，該值即爲最長迴文前綴長度。

### Single Number III

所有數的xor等於兩個出現一次的數的xor，不妨設爲`k`，則`k & -k`爲二進制表示上某個爲1的數位。根據這個數位把所有元素劃分成兩份，每份只有一個出現一次的數。

### Smallest Good Base

設$n=1+k+k^2+\ldots+k^m$，由$n>k^m$得$k<n^{1/m}$，由$(1+k)^m>n$得$k+1>n^{1/m}$，因此$n^{1/m}-1<k<n^{1/m}$。枚舉$m$，$k$唯一可能取值爲$floor(n^{1/m})$，判斷是否成立。

### Sort Colors

Dutch national flag problem。如果不要求000111222，允許111000222111，那麼有交換次數更少的Bentley-McIlroy算法<http://www.iis.sinica.edu.tw/~scm/ncs/2010/10/dutch-national-flag-problem-3/>

### Sqrt(x)

Hacker's Delight (2nd) 11.1.1。
牛頓迭代法。`46340 = floor(sqrt(INT_MAX))`

### Scramble String

我用了$O(n^3)$的空間和$O(n^4)$的時間，應該有更好的算法。但查閱了一些文獻還沒有找到。

### Sort List

我用了較麻煩的natural merge sort。Quicksort實現會簡單很多。

### Strobogrammatic Number III

定義一個函數，用於計算小於給定數的strobogrammatic number個數，則用減法即可得到題目所求。下面考慮如何計算這個函數。
小於給定數high的strobogrammatic number分爲兩類：

- 位數不夠的，可看作有若干前導0。容易求得。
- 位數相同。可以枚舉與high的公共前綴的長度，再計算下一個數位小於high中對應數位的strobogrammatic number個數。由於位數爲$n$的strobogrammatic number只有$ceil(n/2)$個數位是獨立的，枚舉的公共前綴長度得小於$\lceil n/2\rfrac$。

### Sudoku Solver

轉化爲exact cover problem，使用dancing links + Algorithm X求解。

### Trapping Rain Water

兩個指針夾逼，空間$O(1)$。

### Unique Binary Search Trees II

類似動態規劃的思想，子樹可以復用。

### Wiggle Sort

從左到右考慮每一對相鄰元素，如果不滿足大小關係則交換，交換不會影響之前一對元素的大小關係。

### Word Ladder

基本方法是BFS或bidirectional BFS，相鄰關係的確定有兩種思路：枚舉所有其他頂點判斷是否相鄰、枚舉頂點的變異判斷是否爲頂點。對於第一種思路，還可以利用一個技巧：把所有字串都分爲左右兩半，若兩個字串hamming distance爲1，則左半部分相同或右半部分相同。

### Verify Preorder Sequence in Binary Search Tree

對於前序序列中的遞減連續段，它們形成了BST的某個左孩子鏈。

### Maximal Rectangle

#### 秋葉拓哉（iwi）、巖田陽一（wata）和北川宜稔（kita_masa）所著，巫澤俊（watashi）、莊俊元（navi）和李津羽（itsuhane）翻譯的《挑戰程序設計競賽》

逐行掃描棋盤，對於每一行$i$記錄三個值：

- $h_j$表示第$j$列上方連續爲1的行數：`h[j] = a[i][j] == 0 ? 0 : h[j]+1;`
- 若$a_{i,j}=1$，令$l_j$爲$min(k : h_{k+1\ldots j}\ge h_j)$；若$a_{i,j}=0$則設爲$j$
- 若$a_{i,j}=1$，令$r_j$爲$max(k : h_{j\ldots k-1}\ge h_j)$；若$a_{i,j}=0$則設爲$j$

$l$的計算方式是：
```
if a[i][j] == 0:
  l[j] = j
elif h[j] == 1:
  l[j] = j-(a[i][j]及左邊連續的1的個數)+1
else
  # 下式中右邊的l[j]是第i-1行計算得到的l[j]
  l[j] = min(l[j], j-(a[i][j]及左邊連續的1的個數)+1)
```

$r$的計算類似。對於每一列$j$，$(r_j-l_j+1)\times h_j$是一個候選解。

```
#define ROF(i, a, b) for (int i = (b); --i >= (a); )
#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define REP(i, n) for (int i = 0; i < (n); i++)

class Solution {
public:
  int maximalRectangle(vector<vector<char> > &a) {
    if (a.empty()) return 0;
    int m = a.size(), n = a[0].size(), ans = 0;
    vector<int> h(n), l(n), r(n, n-1);
    REP(i, m) {
      int ll = -1;
      REP(j, n) {
        h[j] = a[i][j] == '1' ? h[j]+1 : 0;
        if (a[i][j] == '0') ll = j;
        l[j] = h[j] ? max(h[j] == 1 ? 0 : l[j], ll+1) : j;
      }
      int rr = n;
      ROF(j, 0, n) {
        if (a[i][j] == '0') rr = j;
        r[j] = h[j] ? min(h[j] == 1 ? n-1 : r[j], rr-1) : j;
        ans = max(ans, (r[j]-l[j]+1)*h[j]);
      }
    }
    return ans;
  }
};
```

#### 潘宇超 2008

```
#define ROF(i, a, b) for (int i = (b); --i >= (a); )
#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define REP(i, n) for (int i = 0; i < (n); i++)

class Solution {
public:
  int maximalRectangle(vector<vector<char> > &a) {
    if (a.empty()) return 0;
    int m = a.size(), n = a[0].size(), ans = 0;
    vector<int> h(n), l(n), r(n, n-1);
    REP(i, m) {
      REP(j, n) {
        h[j] = a[i][j] == '1' ? h[j]+1 : 0;
        l[j] = j;
        while (l[j] && h[l[j]-1] >= h[j])
          l[j] = l[l[j]-1];
      }
      ROF(j, 0, n) {
        r[j] = j;
        while (r[j]+1 < n && h[j] <= h[r[j]+1])
          r[j] = r[r[j]+1];
        ans = max(ans, (r[j]-l[j]+1)*h[j]);
      }
    }
    return ans;
  }
};
```

#### ACRush某TopCoder SRM

逐行掃描棋盤，對於每一行記錄$h_j$：第$j$列上方連續爲1的行數。

對於每一行$i$，從大到小排序$h$，並計算候選解：
```
x = [0,0,...,0]
foreach j, h[j]: # 從大到小遍歷h[j]
  x[j] = 1
  ans = max(ans, (max(k : x[j..k]都爲1) - min(k : x[k..j]都爲1)) * h[j])
```

上面僞代碼可以在$O(n)$時間內求出。方法是把$x$中設置爲1的元素看成若干集合，相鄰元素在同一個集合中。每次把某個$x$中元素設爲1可以看成新建了一個集合，若左邊或右邊的集合存在則合併。

不相交集合可以用union-find算法，但針對這種特殊情形存在$O(n)$的算法：每個集合(即某個1的連續段)左端和右端互指，其他元素的指針任意。新建集合時若左邊或右邊的集合存在，則更新它們兩端的指針。

```
#define ROF(i, a, b) for (int i = (b); --i >= (a); )
#define FOR(i, a, b) for (int i = (a); i < (b); i++)
#define REP(i, n) for (int i = 0; i < (n); i++)
#define REP1(i, n) for (int i = 1; i <= (n); i++)

class Solution {
public:
  int maximalRectangle(vector<vector<char> > &a) {
    if (a.empty()) return 0;
    int m = a.size(), n = a[0].size(), ans = 0;
    vector<int> h(n), p(n), b(m+1), s(n);
    REP(i, m) {
      REP(j, n)
        h[j] = a[i][j] == '1' ? h[j]+1 : 0;
      fill(b.begin(), b.end(), 0);
      REP(j, n)
        b[h[j]]++;
      REP1(j, m)
        b[j] += b[j-1];
      REP(j, n)
        s[--b[h[j]]] = j;
      fill(p.begin(), p.end(), -1);
      ROF(j, 0, n) {
        int x = s[j], l = x, r = x;
        p[x] = x;
        if (x && p[x-1] != -1) {
          l = p[x-1];
          p[l] = x;
          p[x] = l;
        }
        if (x+1 < n && p[x+1] != -1) {
          l = p[x];
          r = p[x+1];
          p[l] = r;
          p[r] = l;
        }
        ans = max(ans, (r-l+1)*h[x]);
      }
    }
    return ans;
  }
};
```

#### 棧維護histogram

逐行掃描棋盤，對於每一行記錄$h_j$：第$j$列上方連續爲1的行數。

對於每一行$i$，維護一個以$h$值爲元素的棧，滿足從棧底到棧頂$h$值遞增。從左到右枚舉各列$j$，在插入$h_j$前維持棧性質，彈出元素後棧頂代表$j$往左能“看到”的最遠位置，插入$h_j$。

#### 其他方法　

枚舉兩列，求出夾在兩列間的最大1子矩形。$O(n^3)$

令$s_{i,j}$表示以原矩形最上角和$(i,j)$兩個端點的矩形內1的數目，枚舉所有子矩形$((i,j),(ii,jj))$，用$s_{ii,jj}-s_{ii,j-1}-s_{i-1,jj}+s_{i-1,j-1}$求出這個子矩形內1的數目，判斷是否合法。$O(n^4)$

把0看成障礙點，求不包含障礙點的最大子矩形。
把障礙點按橫座標排序。枚舉障礙點作爲候選子矩形的左端，向右掃描各個障礙點，維護縱座標和枚舉點最近的上下個一個障礙點。枚舉點、掃描線和上下障礙點確定了候選子矩形的邊界。
若0的數目爲，$O(m^2)$，最壞$O(n^4)$。

## 代碼索引

| # | Title | Solution |
|---| ----- | -------- |
|1897|[Maximize Palindrome Length From Subsequences](https://leetcode.com/problems/maximize-palindrome-length-from-subsequences)|[maximize-palindrome-length-from-subsequences.cc](/leetcode/maximize-palindrome-length-from-subsequences.cc)|
|1896|[Maximum Score from Performing Multiplication Operations](https://leetcode.com/problems/maximum-score-from-performing-multiplication-operations)|[maximum-score-from-performing-multiplication-operations.cc](/leetcode/maximum-score-from-performing-multiplication-operations.cc)|
|1895|[Minimum Number of Operations to Move All Balls to Each Box](https://leetcode.com/problems/minimum-number-of-operations-to-move-all-balls-to-each-box)|[minimum-number-of-operations-to-move-all-balls-to-each-box.cc](/leetcode/minimum-number-of-operations-to-move-all-balls-to-each-box.cc)|
|1894|[Merge Strings Alternately](https://leetcode.com/problems/merge-strings-alternately)|[merge-strings-alternately.cc](/leetcode/merge-strings-alternately.cc)|
|1876|[Map of Highest Peak](https://leetcode.com/problems/map-of-highest-peak)|[map-of-highest-peak.cc](/leetcode/map-of-highest-peak.cc)|
|1875|[Tree of Coprimes](https://leetcode.com/problems/tree-of-coprimes)|[tree-of-coprimes.cc](/leetcode/tree-of-coprimes.cc)|
|1874|[Form Array by Concatenating Subarrays of Another Array](https://leetcode.com/problems/form-array-by-concatenating-subarrays-of-another-array)|[form-array-by-concatenating-subarrays-of-another-array.cc](/leetcode/form-array-by-concatenating-subarrays-of-another-array.cc)|
|916|[Decoded String at Index](https://leetcode.com/problems/decoded-string-at-index)|[decoded-string-at-index.cc](/leetcode/decoded-string-at-index.cc)|
|830|[Largest Triangle Area](https://leetcode.com/problems/largest-triangle-area)|[largest-triangle-area.cc](/leetcode/largest-triangle-area.cc)|
|673|[Number of Longest Increasing Subsequence](https://leetcode.com/problems/number-of-longest-increasing-subsequence)|[number-of-longest-increasing-subsequence.cc](/leetcode/number-of-longest-increasing-subsequence.cc)|
|635|[Design Log Storage System](https://leetcode.com/problems/design-log-storage-system)|[design-log-storage-system.cc](/leetcode/design-log-storage-system.cc)|
|634|[Find the Derangement of An Array](https://leetcode.com/problems/find-the-derangement-of-an-array)|[find-the-derangement-of-an-array.cc](/leetcode/find-the-derangement-of-an-array.cc)|
|633|[Sum of Square Numbers](https://leetcode.com/problems/sum-of-square-numbers)|[sum-of-square-numbers.cc](/leetcode/sum-of-square-numbers.cc)|
|631|[Design Excel Sum Formula](https://leetcode.com/problems/design-excel-sum-formula)|[design-excel-sum-formula.cc](/leetcode/design-excel-sum-formula.cc)|
|630|[Course Schedule III](https://leetcode.com/problems/course-schedule-iii)|[course-schedule-iii.cc](/leetcode/course-schedule-iii.cc)|
|629|[K Inverse Pairs Array](https://leetcode.com/problems/k-inverse-pairs-array)|[k-inverse-pairs-array.cc](/leetcode/k-inverse-pairs-array.cc)|
|628|[Maximum Product of Three Numbers](https://leetcode.com/problems/maximum-product-of-three-numbers)|[maximum-product-of-three-numbers.cc](/leetcode/maximum-product-of-three-numbers.cc)|
|617|[Merge Two Binary Trees](https://leetcode.com/problems/merge-two-binary-trees)|[merge-two-binary-trees.cc](/leetcode/merge-two-binary-trees.cc)|
|616|[Add Bold Tag in String](https://leetcode.com/problems/add-bold-tag-in-string)|[add-bold-tag-in-string.cc](/leetcode/add-bold-tag-in-string.cc)|
|611|[Valid Triangle Number](https://leetcode.com/problems/valid-triangle-number)|[valid-triangle-number.cc](/leetcode/valid-triangle-number.cc)|
|604|[Design Compressed String Iterator](https://leetcode.com/problems/design-compressed-string-iterator)|[design-compressed-string-iterator.cc](/leetcode/design-compressed-string-iterator.cc)|
|600|[Non-negative Integers without Consecutive Ones](https://leetcode.com/problems/non-negative-integers-without-consecutive-ones)|[non-negative-integers-without-consecutive-ones.cc](/leetcode/non-negative-integers-without-consecutive-ones.cc)|
|599|[Minimum Index Sum of Two Lists](https://leetcode.com/problems/minimum-index-sum-of-two-lists)|[minimum-index-sum-of-two-lists.cc](/leetcode/minimum-index-sum-of-two-lists.cc)|
|598|[Range Addition II](https://leetcode.com/problems/range-addition-ii)|[range-addition-ii.cc](/leetcode/range-addition-ii.cc)|
|594|[Longest Harmonious Subsequence](https://leetcode.com/problems/longest-harmonious-subsequence)|[longest-harmonious-subsequence.cc](/leetcode/longest-harmonious-subsequence.cc)|
|593|[Valid Square](https://leetcode.com/problems/valid-square)|[valid-square.cc](/leetcode/valid-square.cc)|
|592|[Fraction Addition and Subtraction](https://leetcode.com/problems/fraction-addition-and-subtraction)|[fraction-addition-and-subtraction.cc](/leetcode/fraction-addition-and-subtraction.cc)|
|588|[Design In-Memory File System](https://leetcode.com/problems/design-in-memory-file-system)|[design-in-memory-file-system.cc](/leetcode/design-in-memory-file-system.cc)|
|587|[Erect the Fence](https://leetcode.com/problems/erect-the-fence)|[erect-the-fence.cc](/leetcode/erect-the-fence.cc)|
|583|[Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings)|[delete-operation-for-two-strings.cc](/leetcode/delete-operation-for-two-strings.cc)|
|582|[Kill Process](https://leetcode.com/problems/kill-process)|[kill-process.cc](/leetcode/kill-process.cc)|
|581|[Shortest Unsorted Continuous Subarray](https://leetcode.com/problems/shortest-unsorted-continuous-subarray)|[shortest-unsorted-continuous-subarray.cc](/leetcode/shortest-unsorted-continuous-subarray.cc)|
|575|[Distribute Candies](https://leetcode.com/problems/distribute-candies)|[distribute-candies.cc](/leetcode/distribute-candies.cc)|
|565|[Array Nesting](https://leetcode.com/problems/array-nesting)|[array-nesting.cc](/leetcode/array-nesting.cc)|
|556|[Next Greater Element III](https://leetcode.com/problems/next-greater-element-iii)|[next-greater-element-iii.cc](/leetcode/next-greater-element-iii.cc)|
|553|[Optimal Division](https://leetcode.com/problems/optimal-division)|[optimal-division.cc](/leetcode/optimal-division.cc)|
|552|[Student Attendance Record II](https://leetcode.com/problems/student-attendance-record-ii)|[student-attendance-record-ii.cc](/leetcode/student-attendance-record-ii.cc)|
|533|[Lonely Pixel II](https://leetcode.com/problems/lonely-pixel-ii)|[lonely-pixel-ii.cc](/leetcode/lonely-pixel-ii.cc)|
|531|[Lonely Pixel I](https://leetcode.com/problems/lonely-pixel-i)|[lonely-pixel-i.cc](/leetcode/lonely-pixel-i.cc)|
|514|[Freedom Trail](https://leetcode.com/problems/freedom-trail)|[freedom-trail.cc](/leetcode/freedom-trail.cc)|
|508|[Most Frequent Subtree Sum](https://leetcode.com/problems/most-frequent-subtree-sum)|[most-frequent-subtree-sum.cc](/leetcode/most-frequent-subtree-sum.cc)|
|507|[Perfect Number](https://leetcode.com/problems/perfect-number)|[perfect-number.cc](/leetcode/perfect-number.cc)|
|504|[Base 7](https://leetcode.com/problems/base-7)|[base-7.cc](/leetcode/base-7.cc)|
|502|[IPO](https://leetcode.com/problems/ipo)|[ipo.cc](/leetcode/ipo.cc)|
|501|[Find Mode in Binary Search Tree](https://leetcode.com/problems/find-mode-in-binary-search-tree)|[find-mode-in-binary-search-tree.cc](/leetcode/find-mode-in-binary-search-tree.cc)|
|500|[Keyboard Row](https://leetcode.com/problems/keyboard-row)|[keyboard-row.cc](/leetcode/keyboard-row.cc)|
|495|[Teemo Attacking](https://leetcode.com/problems/teemo-attacking)|[teemo-attacking.cc](/leetcode/teemo-attacking.cc)|
|494|[Target Sum](https://leetcode.com/problems/target-sum)|[target-sum.cc](/leetcode/target-sum.cc)|
|491|[Increasing Subsequences](https://leetcode.com/problems/increasing-subsequences)|[increasing-subsequences.cc](/leetcode/increasing-subsequences.cc)|
|490|[The Maze](https://leetcode.com/problems/the-maze)|[the-maze.cc](/leetcode/the-maze.cc)|
|485|[Max Consecutive Ones](https://leetcode.com/problems/max-consecutive-ones)|[max-consecutive-ones.cc](/leetcode/max-consecutive-ones.cc)|
|483|[Smallest Good Base](https://leetcode.com/problems/smallest-good-base)|[smallest-good-base.cc](/leetcode/smallest-good-base.cc)|
|482|[License Key Formatting](https://leetcode.com/problems/license-key-formatting)|[license-key-formatting.cc](/leetcode/license-key-formatting.cc)|
|481|[Magical String](https://leetcode.com/problems/magical-string)|[magical-string.cc](/leetcode/magical-string.cc)|
|480|[Sliding Window Median](https://leetcode.com/problems/sliding-window-median)|[sliding-window-median.cc](/leetcode/sliding-window-median.cc)|
|477|[Total Hamming Distance](https://leetcode.com/problems/total-hamming-distance)|[total-hamming-distance.cc](/leetcode/total-hamming-distance.cc)|
|476|[Number Complement](https://leetcode.com/problems/number-complement)|[number-complement.cc](/leetcode/number-complement.cc)|
|475|[Heaters](https://leetcode.com/problems/heaters)|[heaters.cc](/leetcode/heaters.cc)|
|474|[Ones and Zeroes](https://leetcode.com/problems/ones-and-zeroes)|[ones-and-zeroes.cc](/leetcode/ones-and-zeroes.cc)|
|473|[Matchsticks to Square](https://leetcode.com/problems/matchsticks-to-square)|[matchsticks-to-square.cc](/leetcode/matchsticks-to-square.cc)|
|472|[Concatenated Words](https://leetcode.com/problems/concatenated-words)|[concatenated-words.cc](/leetcode/concatenated-words.cc)|
|469|[Convex Polygon](https://leetcode.com/problems/convex-polygon)|[convex-polygon.cc](/leetcode/convex-polygon.cc)|
|468|[Validate IP Address](https://leetcode.com/problems/validate-ip-address)|[validate-ip-address.cc](/leetcode/validate-ip-address.cc)|
|467|[Unique Substrings in Wraparound String](https://leetcode.com/problems/unique-substrings-in-wraparound-string)|[unique-substrings-in-wraparound-string.cc](/leetcode/unique-substrings-in-wraparound-string.cc)|
|466|[Count The Repetitions](https://leetcode.com/problems/count-the-repetitions)|[count-the-repetitions.cc](/leetcode/count-the-repetitions.cc)|
|465|[Optimal Account Balancing](https://leetcode.com/problems/optimal-account-balancing)|[optimal-account-balancing.cc](/leetcode/optimal-account-balancing.cc)|
|464|[Can I Win](https://leetcode.com/problems/can-i-win)|[can-i-win.cc](/leetcode/can-i-win.cc)|
|463|[Island Perimeter](https://leetcode.com/problems/island-perimeter)|[island-perimeter.cc](/leetcode/island-perimeter.cc)|
|462|[Minimum Moves to Equal Array Elements II](https://leetcode.com/problems/minimum-moves-to-equal-array-elements-ii)|[minimum-moves-to-equal-array-elements-ii.cc](/leetcode/minimum-moves-to-equal-array-elements-ii.cc)|
|461|[Hamming Distance](https://leetcode.com/problems/hamming-distance)|[hamming-distance.cc](/leetcode/hamming-distance.cc)|
|459|[Repeated Substring Pattern](https://leetcode.com/problems/repeated-substring-pattern)|[repeated-substring-pattern.cc](/leetcode/repeated-substring-pattern.cc)|
|456|[132 Pattern](https://leetcode.com/problems/132-pattern)|[132-pattern.cc](/leetcode/132-pattern.cc)|
|455|[Assign Cookies](https://leetcode.com/problems/assign-cookies)|[assign-cookies.cc](/leetcode/assign-cookies.cc)|
|454|[4Sum II](https://leetcode.com/problems/4sum-ii)|[4sum-ii.cc](/leetcode/4sum-ii.cc)|
|453|[Minimum Moves to Equal Array Elements](https://leetcode.com/problems/minimum-moves-to-equal-array-elements)|[minimum-moves-to-equal-array-elements.cc](/leetcode/minimum-moves-to-equal-array-elements.cc)|
|452|[Minimum Number of Arrows to Burst Balloons](https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons)|[minimum-number-of-arrows-to-burst-balloons.cc](/leetcode/minimum-number-of-arrows-to-burst-balloons.cc)|
|447|[Number of Boomerangs](https://leetcode.com/problems/number-of-boomerangs)|[number-of-boomerangs.cc](/leetcode/number-of-boomerangs.cc)|
|446|[Arithmetic Slices II - Subsequence](https://leetcode.com/problems/arithmetic-slices-ii-subsequence)|[arithmetic-slices-ii-subsequence.cc](/leetcode/arithmetic-slices-ii-subsequence.cc)|
|444|[Sequence Reconstruction](https://leetcode.com/problems/sequence-reconstruction)|[sequence-reconstruction.cc](/leetcode/sequence-reconstruction.cc)|
|441|[Arranging Coins](https://leetcode.com/problems/arranging-coins)|[arranging-coins.cc](/leetcode/arranging-coins.cc)|
|440|[K-th Smallest in Lexicographical Order](https://leetcode.com/problems/k-th-smallest-in-lexicographical-order)|[k-th-smallest-in-lexicographical-order.cc](/leetcode/k-th-smallest-in-lexicographical-order.cc)|
|439|[Ternary Expression Parser](https://leetcode.com/problems/ternary-expression-parser)|[ternary-expression-parser.cc](/leetcode/ternary-expression-parser.cc)|
|438|[Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string)|[find-all-anagrams-in-a-string.cc](/leetcode/find-all-anagrams-in-a-string.cc)|
|437|[Path Sum III](https://leetcode.com/problems/path-sum-iii)|[path-sum-iii.cc](/leetcode/path-sum-iii.cc)|
|436|[Find Right Interval](https://leetcode.com/problems/find-right-interval)|[find-right-interval.cc](/leetcode/find-right-interval.cc)|
|435|[Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals)|[non-overlapping-intervals.cc](/leetcode/non-overlapping-intervals.cc)|
|434|[Number of Segments in a String](https://leetcode.com/problems/number-of-segments-in-a-string)|[number-of-segments-in-a-string.cc](/leetcode/number-of-segments-in-a-string.cc)|
|432|[All O`one Data Structure](https://leetcode.com/problems/all-oone-data-structure)|[all-oone-data-structure.cc](/leetcode/all-oone-data-structure.cc)|
|424|[Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement)|[longest-repeating-character-replacement.cc](/leetcode/longest-repeating-character-replacement.cc)|
|423|[Reconstruct Original Digits from English](https://leetcode.com/problems/reconstruct-original-digits-from-english)|[reconstruct-original-digits-from-english.cc](/leetcode/reconstruct-original-digits-from-english.cc)|
|422|[Valid Word Square](https://leetcode.com/problems/valid-word-square)|[valid-word-square.cc](/leetcode/valid-word-square.cc)|
|421|[Maximum XOR of Two Numbers in an Array](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array)|[maximum-xor-of-two-numbers-in-an-array.cc](/leetcode/maximum-xor-of-two-numbers-in-an-array.cc)|
|420|[Strong Password Checker](https://leetcode.com/problems/strong-password-checker)|[strong-password-checker.cc](/leetcode/strong-password-checker.cc)|
|419|[Battleships in a Board](https://leetcode.com/problems/battleships-in-a-board)|[battleships-in-a-board.cc](/leetcode/battleships-in-a-board.cc)|
|417|[Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow)|[pacific-atlantic-water-flow.cc](/leetcode/pacific-atlantic-water-flow.cc)|
|416|[Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum)|[partition-equal-subset-sum.cc](/leetcode/partition-equal-subset-sum.cc)|
|415|[Add Strings](https://leetcode.com/problems/add-strings)|[add-strings.cc](/leetcode/add-strings.cc)|
|414|[Third Maximum Number](https://leetcode.com/problems/third-maximum-number)|[third-maximum-number.cc](/leetcode/third-maximum-number.cc)|
|413|[Arithmetic Slices](https://leetcode.com/problems/arithmetic-slices)|[arithmetic-slices.cc](/leetcode/arithmetic-slices.cc)|
|412|[Fizz Buzz](https://leetcode.com/problems/fizz-buzz)|[fizz-buzz.cc](/leetcode/fizz-buzz.cc)|
|410|[Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum)|[split-array-largest-sum.cc](/leetcode/split-array-largest-sum.cc)|
|409|[Longest Palindrome](https://leetcode.com/problems/longest-palindrome)|[longest-palindrome.cc](/leetcode/longest-palindrome.cc)|
|408|[Valid Word Abbreviation](https://leetcode.com/problems/valid-word-abbreviation)|[valid-word-abbreviation.cc](/leetcode/valid-word-abbreviation.cc)|
|407|[Trapping Rain Water II](https://leetcode.com/problems/trapping-rain-water-ii)|[trapping-rain-water-ii.cc](/leetcode/trapping-rain-water-ii.cc)|
|406|[Queue Reconstruction by Height](https://leetcode.com/problems/queue-reconstruction-by-height)|[queue-reconstruction-by-height.cc](/leetcode/queue-reconstruction-by-height.cc)|
|405|[Convert a Number to Hexadecimal](https://leetcode.com/problems/convert-a-number-to-hexadecimal)|[convert-a-number-to-hexadecimal.cc](/leetcode/convert-a-number-to-hexadecimal.cc)|
|404|[Sum of Left Leaves](https://leetcode.com/problems/sum-of-left-leaves)|[sum-of-left-leaves.cc](/leetcode/sum-of-left-leaves.cc)|
|403|[Frog Jump](https://leetcode.com/problems/frog-jump)|[frog-jump.cc](/leetcode/frog-jump.cc)|
|402|[Remove K Digits](https://leetcode.com/problems/remove-k-digits)|[remove-k-digits.cc](/leetcode/remove-k-digits.cc)|
|401|[Binary Watch](https://leetcode.com/problems/binary-watch)|[binary-watch.cc](/leetcode/binary-watch.cc)|
|400|[Nth Digit](https://leetcode.com/problems/nth-digit)|[nth-digit.cc](/leetcode/nth-digit.cc)|
|399|[Evaluate Division](https://leetcode.com/problems/evaluate-division)|[evaluate-division.cc](/leetcode/evaluate-division.cc)|
|398|[Random Pick Index](https://leetcode.com/problems/random-pick-index)|[random-pick-index.cc](/leetcode/random-pick-index.cc)|
|397|[Integer Replacement](https://leetcode.com/problems/integer-replacement)|[integer-replacement.cc](/leetcode/integer-replacement.cc)|
|396|[Rotate Function](https://leetcode.com/problems/rotate-function)|[rotate-function.cc](/leetcode/rotate-function.cc)|
|395|[Longest Substring with At Least K Repeating Characters](https://leetcode.com/problems/longest-substring-with-at-least-k-repeating-characters)|[longest-substring-with-at-least-k-repeating-characters.cc](/leetcode/longest-substring-with-at-least-k-repeating-characters.cc)|
|394|[Decode String](https://leetcode.com/problems/decode-string)|[decode-string.cc](/leetcode/decode-string.cc)|
|393|[UTF-8 Validation](https://leetcode.com/problems/utf-8-validation)|[utf-8-validation.cc](/leetcode/utf-8-validation.cc)|
|392|[Is Subsequence](https://leetcode.com/problems/is-subsequence)|[is-subsequence.cc](/leetcode/is-subsequence.cc)|
|391|[Perfect Rectangle](https://leetcode.com/problems/perfect-rectangle)|[perfect-rectangle.cc](/leetcode/perfect-rectangle.cc)|
|390|[Elimination Game](https://leetcode.com/problems/elimination-game)|[elimination-game.cc](/leetcode/elimination-game.cc)|
|389|[Find the Difference](https://leetcode.com/problems/find-the-difference)|[find-the-difference.cc](/leetcode/find-the-difference.cc)|
|388|[Longest Absolute File Path](https://leetcode.com/problems/longest-absolute-file-path)|[longest-absolute-file-path.cc](/leetcode/longest-absolute-file-path.cc)|
|387|[First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string)|[first-unique-character-in-a-string.cc](/leetcode/first-unique-character-in-a-string.cc)|
|386|[Lexicographical Numbers](https://leetcode.com/problems/lexicographical-numbers)|[lexicographical-numbers.cc](/leetcode/lexicographical-numbers.cc)|
|385|[Mini Parser](https://leetcode.com/problems/mini-parser)|[mini-parser.cc](/leetcode/mini-parser.cc)|
|384|[Shuffle an Array](https://leetcode.com/problems/shuffle-an-array)|[shuffle-an-array.cc](/leetcode/shuffle-an-array.cc)|
|383|[Ransom Note](https://leetcode.com/problems/ransom-note)|[ransom-note.cc](/leetcode/ransom-note.cc)|
|382|[Linked List Random Node](https://leetcode.com/problems/linked-list-random-node)|[linked-list-random-node.cc](/leetcode/linked-list-random-node.cc)|
|381|[Insert Delete GetRandom O(1) - Duplicates allowed](https://leetcode.com/problems/insert-delete-getrandom-o1-duplicates-allowed)|[insert-delete-getrandom-o1-duplicates-allowed.cc](/leetcode/insert-delete-getrandom-o1-duplicates-allowed.cc)|
|380|[Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1)|[insert-delete-getrandom-o1.cc](/leetcode/insert-delete-getrandom-o1.cc)|
|379|[Design Phone Directory](https://leetcode.com/problems/design-phone-directory)|[design-phone-directory.cc](/leetcode/design-phone-directory.cc)|
|378|[Kth Smallest Element in a Sorted Matrix](https://leetcode.com/problems/kth-smallest-element-in-a-sorted-matrix)|[kth-smallest-element-in-a-sorted-matrix.cc](/leetcode/kth-smallest-element-in-a-sorted-matrix.cc)|
|377|[Combination Sum IV](https://leetcode.com/problems/combination-sum-iv)|[combination-sum-iv.cc](/leetcode/combination-sum-iv.cc)|
|376|[Wiggle Subsequence](https://leetcode.com/problems/wiggle-subsequence)|[wiggle-subsequence.cc](/leetcode/wiggle-subsequence.cc)|
|375|[Guess Number Higher or Lower II](https://leetcode.com/problems/guess-number-higher-or-lower-ii)|[guess-number-higher-or-lower-ii.cc](/leetcode/guess-number-higher-or-lower-ii.cc)|
|374|[Guess Number Higher or Lower](https://leetcode.com/problems/guess-number-higher-or-lower)|[guess-number-higher-or-lower.cc](/leetcode/guess-number-higher-or-lower.cc)|
|373|[Find K Pairs with Smallest Sums](https://leetcode.com/problems/find-k-pairs-with-smallest-sums)|[find-k-pairs-with-smallest-sums.cc](/leetcode/find-k-pairs-with-smallest-sums.cc)|
|372|[Super Pow](https://leetcode.com/problems/super-pow)|[super-pow.cc](/leetcode/super-pow.cc)|
|371|[Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers)|[sum-of-two-integers.cc](/leetcode/sum-of-two-integers.cc)|
|370|[Range Addition](https://leetcode.com/problems/range-addition)|[range-addition.cc](/leetcode/range-addition.cc)|
|369|[Plus One Linked List](https://leetcode.com/problems/plus-one-linked-list)|[plus-one-linked-list.cc](/leetcode/plus-one-linked-list.cc)|
|368|[Largest Divisible Subset](https://leetcode.com/problems/largest-divisible-subset)|[largest-divisible-subset.cc](/leetcode/largest-divisible-subset.cc)|
|367|[Valid Perfect Square](https://leetcode.com/problems/valid-perfect-square)|[valid-perfect-square.cc](/leetcode/valid-perfect-square.cc)|
|366|[Find Leaves of Binary Tree](https://leetcode.com/problems/find-leaves-of-binary-tree)|[find-leaves-of-binary-tree.cc](/leetcode/find-leaves-of-binary-tree.cc)|
|365|[Water and Jug Problem](https://leetcode.com/problems/water-and-jug-problem)|[water-and-jug-problem.cc](/leetcode/water-and-jug-problem.cc)|
|364|[Nested List Weight Sum II](https://leetcode.com/problems/nested-list-weight-sum-ii)|[nested-list-weight-sum-ii.cc](/leetcode/nested-list-weight-sum-ii.cc)|
|362|[Design Hit Counter](https://leetcode.com/problems/design-hit-counter)|[design-hit-counter.cc](/leetcode/design-hit-counter.cc)|
|361|[Bomb Enemy](https://leetcode.com/problems/bomb-enemy)|[bomb-enemy.cc](/leetcode/bomb-enemy.cc)|
|360|[Sort Transformed Array](https://leetcode.com/problems/sort-transformed-array)|[sort-transformed-array.cc](/leetcode/sort-transformed-array.cc)|
|359|[Logger Rate Limiter](https://leetcode.com/problems/logger-rate-limiter)|[logger-rate-limiter.cc](/leetcode/logger-rate-limiter.cc)|
|358|[Rearrange String k Distance Apart](https://leetcode.com/problems/rearrange-string-k-distance-apart)|[rearrange-string-k-distance-apart.cc](/leetcode/rearrange-string-k-distance-apart.cc)|
|357|[Count Numbers with Unique Digits](https://leetcode.com/problems/count-numbers-with-unique-digits)|[count-numbers-with-unique-digits.cc](/leetcode/count-numbers-with-unique-digits.cc)|
|356|[Line Reflection](https://leetcode.com/problems/line-reflection)|[line-reflection.cc](/leetcode/line-reflection.cc)|
|355|[Design Twitter](https://leetcode.com/problems/design-twitter)|[design-twitter.cc](/leetcode/design-twitter.cc)|
|354|[Russian Doll Envelopes](https://leetcode.com/problems/russian-doll-envelopes)|[russian-doll-envelopes.cc](/leetcode/russian-doll-envelopes.cc)|
|353|[Design Snake Game](https://leetcode.com/problems/design-snake-game)|[design-snake-game.cc](/leetcode/design-snake-game.cc)|
|352|[Data Stream as Disjoint Intervals](https://leetcode.com/problems/data-stream-as-disjoint-intervals)|[data-stream-as-disjoint-intervals.cc](/leetcode/data-stream-as-disjoint-intervals.cc)|
|351|[Android Unlock Patterns](https://leetcode.com/problems/android-unlock-patterns)|[android-unlock-patterns.cc](/leetcode/android-unlock-patterns.cc)|
|350|[Intersection of Two Arrays II](https://leetcode.com/problems/intersection-of-two-arrays-ii)|[intersection-of-two-arrays-ii.cc](/leetcode/intersection-of-two-arrays-ii.cc)|
|349|[Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays)|[intersection-of-two-arrays.cc](/leetcode/intersection-of-two-arrays.cc)|
|348|[Design Tic-Tac-Toe](https://leetcode.com/problems/design-tic-tac-toe)|[design-tic-tac-toe.cc](/leetcode/design-tic-tac-toe.cc)|
|347|[Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements)|[top-k-frequent-elements.cc](/leetcode/top-k-frequent-elements.cc)|
|346|[Moving Average from Data Stream](https://leetcode.com/problems/moving-average-from-data-stream)|[moving-average-from-data-stream.cc](/leetcode/moving-average-from-data-stream.cc)|
|345|[Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string)|[reverse-vowels-of-a-string.cc](/leetcode/reverse-vowels-of-a-string.cc)|
|344|[Reverse String](https://leetcode.com/problems/reverse-string)|[reverse-string.cc](/leetcode/reverse-string.cc)|
|343|[Integer Break](https://leetcode.com/problems/integer-break)|[integer-break.cc](/leetcode/integer-break.cc)|
|342|[Power of Four](https://leetcode.com/problems/power-of-four)|[power-of-four.cc](/leetcode/power-of-four.cc)|
|341|[Flatten Nested List Iterator](https://leetcode.com/problems/flatten-nested-list-iterator)|[flatten-nested-list-iterator.cc](/leetcode/flatten-nested-list-iterator.cc)|
|340|[Longest Substring with At Most K Distinct Characters](https://leetcode.com/problems/longest-substring-with-at-most-k-distinct-characters)|[longest-substring-with-at-most-k-distinct-characters.cc](/leetcode/longest-substring-with-at-most-k-distinct-characters.cc)|
|339|[Nested List Weight Sum](https://leetcode.com/problems/nested-list-weight-sum)|[nested-list-weight-sum.cc](/leetcode/nested-list-weight-sum.cc)|
|338|[Counting Bits](https://leetcode.com/problems/counting-bits)|[counting-bits.cc](/leetcode/counting-bits.cc)|
|337|[House Robber III](https://leetcode.com/problems/house-robber-iii)|[house-robber-iii.cc](/leetcode/house-robber-iii.cc)|
|336|[Palindrome Pairs](https://leetcode.com/problems/palindrome-pairs)|[palindrome-pairs.cc](/leetcode/palindrome-pairs.cc)|
|335|[Self Crossing](https://leetcode.com/problems/self-crossing)|[self-crossing.cc](/leetcode/self-crossing.cc)|
|334|[Increasing Triplet Subsequence](https://leetcode.com/problems/increasing-triplet-subsequence)|[increasing-triplet-subsequence.cc](/leetcode/increasing-triplet-subsequence.cc)|
|333|[Largest BST Subtree](https://leetcode.com/problems/largest-bst-subtree)|[largest-bst-subtree.cc](/leetcode/largest-bst-subtree.cc)|
|332|[Reconstruct Itinerary](https://leetcode.com/problems/reconstruct-itinerary)|[reconstruct-itinerary.cc](/leetcode/reconstruct-itinerary.cc)|
|331|[Verify Preorder Serialization of a Binary Tree](https://leetcode.com/problems/verify-preorder-serialization-of-a-binary-tree)|[verify-preorder-serialization-of-a-binary-tree.cc](/leetcode/verify-preorder-serialization-of-a-binary-tree.cc)|
|330|[Patching Array](https://leetcode.com/problems/patching-array)|[patching-array.cc](/leetcode/patching-array.cc)|
|329|[Longest Increasing Path in a Matrix](https://leetcode.com/problems/longest-increasing-path-in-a-matrix)|[longest-increasing-path-in-a-matrix.cc](/leetcode/longest-increasing-path-in-a-matrix.cc)|
|328|[Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list)|[odd-even-linked-list.cc](/leetcode/odd-even-linked-list.cc)|
|327|[Count of Range Sum](https://leetcode.com/problems/count-of-range-sum)|[count-of-range-sum.cc](/leetcode/count-of-range-sum.cc)|
|326|[Power of Three](https://leetcode.com/problems/power-of-three)|[power-of-three.cc](/leetcode/power-of-three.cc)|
|325|[Maximum Size Subarray Sum Equals k](https://leetcode.com/problems/maximum-size-subarray-sum-equals-k)|[maximum-size-subarray-sum-equals-k.cc](/leetcode/maximum-size-subarray-sum-equals-k.cc)|
|324|[Wiggle Sort II](https://leetcode.com/problems/wiggle-sort-ii)|[wiggle-sort-ii.cc](/leetcode/wiggle-sort-ii.cc)|
|323|[Number of Connected Components in an Undirected Graph](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph)|[number-of-connected-components-in-an-undirected-graph.cc](/leetcode/number-of-connected-components-in-an-undirected-graph.cc)|
|322|[Coin Change](https://leetcode.com/problems/coin-change)|[coin-change.cc](/leetcode/coin-change.cc)|
|321|[Create Maximum Number](https://leetcode.com/problems/create-maximum-number)|[create-maximum-number.cc](/leetcode/create-maximum-number.cc)|
|320|[Generalized Abbreviation](https://leetcode.com/problems/generalized-abbreviation)|[generalized-abbreviation.cc](/leetcode/generalized-abbreviation.cc)|
|319|[Bulb Switcher](https://leetcode.com/problems/bulb-switcher)|[bulb-switcher.cc](/leetcode/bulb-switcher.cc)|
|318|[Maximum Product of Word Lengths](https://leetcode.com/problems/maximum-product-of-word-lengths)|[maximum-product-of-word-lengths.cc](/leetcode/maximum-product-of-word-lengths.cc)|
|317|[Shortest Distance from All Buildings](https://leetcode.com/problems/shortest-distance-from-all-buildings)|[shortest-distance-from-all-buildings.cc](/leetcode/shortest-distance-from-all-buildings.cc)|
|316|[Remove Duplicate Letters](https://leetcode.com/problems/remove-duplicate-letters)|[remove-duplicate-letters.cc](/leetcode/remove-duplicate-letters.cc)|
|315|[Count of Smaller Numbers After Self](https://leetcode.com/problems/count-of-smaller-numbers-after-self)|[count-of-smaller-numbers-after-self.cc](/leetcode/count-of-smaller-numbers-after-self.cc)|
|314|[Binary Tree Vertical Order Traversal](https://leetcode.com/problems/binary-tree-vertical-order-traversal)|[binary-tree-vertical-order-traversal.cc](/leetcode/binary-tree-vertical-order-traversal.cc)|
|313|[Super Ugly Number](https://leetcode.com/problems/super-ugly-number)|[super-ugly-number.cc](/leetcode/super-ugly-number.cc)|
|312|[Burst Balloons](https://leetcode.com/problems/burst-balloons)|[burst-balloons.cc](/leetcode/burst-balloons.cc)|
|311|[Sparse Matrix Multiplication](https://leetcode.com/problems/sparse-matrix-multiplication)|[sparse-matrix-multiplication.cc](/leetcode/sparse-matrix-multiplication.cc)|
|310|[Minimum Height Trees](https://leetcode.com/problems/minimum-height-trees)|[minimum-height-trees.cc](/leetcode/minimum-height-trees.cc)|
|309|[Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown)|[best-time-to-buy-and-sell-stock-with-cooldown.cc](/leetcode/best-time-to-buy-and-sell-stock-with-cooldown.cc)|
|308|[Range Sum Query 2D - Mutable](https://leetcode.com/problems/range-sum-query-2d-mutable)|[range-sum-query-2d-mutable.cc](/leetcode/range-sum-query-2d-mutable.cc)|
|307|[Range Sum Query - Mutable](https://leetcode.com/problems/range-sum-query-mutable)|[range-sum-query-mutable.cc](/leetcode/range-sum-query-mutable.cc)|
|306|[Additive Number](https://leetcode.com/problems/additive-number)|[additive-number.cc](/leetcode/additive-number.cc)|
|305|[Number of Islands II](https://leetcode.com/problems/number-of-islands-ii)|[number-of-islands-ii.cc](/leetcode/number-of-islands-ii.cc)|
|304|[Range Sum Query 2D - Immutable](https://leetcode.com/problems/range-sum-query-2d-immutable)|[range-sum-query-2d-immutable.cc](/leetcode/range-sum-query-2d-immutable.cc)|
|303|[Range Sum Query - Immutable](https://leetcode.com/problems/range-sum-query-immutable)|[range-sum-query-immutable.cc](/leetcode/range-sum-query-immutable.cc)|
|302|[Smallest Rectangle Enclosing Black Pixels](https://leetcode.com/problems/smallest-rectangle-enclosing-black-pixels)|[smallest-rectangle-enclosing-black-pixels.cc](/leetcode/smallest-rectangle-enclosing-black-pixels.cc)|
|301|[Remove Invalid Parentheses](https://leetcode.com/problems/remove-invalid-parentheses)|[remove-invalid-parentheses.cc](/leetcode/remove-invalid-parentheses.cc)|
|300|[Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence)|[longest-increasing-subsequence.cc](/leetcode/longest-increasing-subsequence.cc)|
|299|[Bulls and Cows](https://leetcode.com/problems/bulls-and-cows)|[bulls-and-cows.cc](/leetcode/bulls-and-cows.cc)|
|298|[Binary Tree Longest Consecutive Sequence](https://leetcode.com/problems/binary-tree-longest-consecutive-sequence)|[binary-tree-longest-consecutive-sequence.cc](/leetcode/binary-tree-longest-consecutive-sequence.cc)|
|297|[Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree)|[serialize-and-deserialize-binary-tree.cc](/leetcode/serialize-and-deserialize-binary-tree.cc)|
|296|[Best Meeting Point](https://leetcode.com/problems/best-meeting-point)|[best-meeting-point.cc](/leetcode/best-meeting-point.cc)|
|295|[Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream)|[find-median-from-data-stream.cc](/leetcode/find-median-from-data-stream.cc)|
|294|[Flip Game II](https://leetcode.com/problems/flip-game-ii)|[flip-game-ii.cc](/leetcode/flip-game-ii.cc)|
|293|[Flip Game](https://leetcode.com/problems/flip-game)|[flip-game.cc](/leetcode/flip-game.cc)|
|292|[Nim Game](https://leetcode.com/problems/nim-game)|[nim-game.cc](/leetcode/nim-game.cc)|
|291|[Word Pattern II](https://leetcode.com/problems/word-pattern-ii)|[word-pattern-ii.cc](/leetcode/word-pattern-ii.cc)|
|290|[Word Pattern](https://leetcode.com/problems/word-pattern)|[word-pattern.cc](/leetcode/word-pattern.cc)|
|289|[Game of Life](https://leetcode.com/problems/game-of-life)|[game-of-life.cc](/leetcode/game-of-life.cc)|
|288|[Unique Word Abbreviation](https://leetcode.com/problems/unique-word-abbreviation)|[unique-word-abbreviation.cc](/leetcode/unique-word-abbreviation.cc)|
|287|[Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number)|[find-the-duplicate-number.cc](/leetcode/find-the-duplicate-number.cc)|
|286|[Walls and Gates](https://leetcode.com/problems/walls-and-gates)|[walls-and-gates.cc](/leetcode/walls-and-gates.cc)|
|285|[Inorder Successor in BST](https://leetcode.com/problems/inorder-successor-in-bst)|[inorder-successor-in-bst.cc](/leetcode/inorder-successor-in-bst.cc)|
|284|[Peeking Iterator](https://leetcode.com/problems/peeking-iterator)|[peeking-iterator.cc](/leetcode/peeking-iterator.cc)|
|283|[Move Zeroes](https://leetcode.com/problems/move-zeroes)|[move-zeroes.cc](/leetcode/move-zeroes.cc)|
|282|[Expression Add Operators](https://leetcode.com/problems/expression-add-operators)|[expression-add-operators.cc](/leetcode/expression-add-operators.cc)|
|281|[Zigzag Iterator](https://leetcode.com/problems/zigzag-iterator)|[zigzag-iterator.cc](/leetcode/zigzag-iterator.cc)|
|280|[Wiggle Sort](https://leetcode.com/problems/wiggle-sort)|[wiggle-sort.cc](/leetcode/wiggle-sort.cc)|
|279|[Perfect Squares](https://leetcode.com/problems/perfect-squares)|[perfect-squares.cc](/leetcode/perfect-squares.cc)|
|278|[First Bad Version](https://leetcode.com/problems/first-bad-version)|[first-bad-version.cc](/leetcode/first-bad-version.cc)|
|277|[Find the Celebrity](https://leetcode.com/problems/find-the-celebrity)|[find-the-celebrity.cc](/leetcode/find-the-celebrity.cc)|
|276|[Paint Fence](https://leetcode.com/problems/paint-fence)|[paint-fence.cc](/leetcode/paint-fence.cc)|
|275|[H-Index II](https://leetcode.com/problems/h-index-ii)|[h-index-ii.cc](/leetcode/h-index-ii.cc)|
|274|[H-Index](https://leetcode.com/problems/h-index)|[h-index.cc](/leetcode/h-index.cc)|
|273|[Integer to English Words](https://leetcode.com/problems/integer-to-english-words)|[integer-to-english-words.cc](/leetcode/integer-to-english-words.cc)|
|272|[Closest Binary Search Tree Value II](https://leetcode.com/problems/closest-binary-search-tree-value-ii)|[closest-binary-search-tree-value-ii.cc](/leetcode/closest-binary-search-tree-value-ii.cc)|
|271|[Encode and Decode Strings](https://leetcode.com/problems/encode-and-decode-strings)|[encode-and-decode-strings.cc](/leetcode/encode-and-decode-strings.cc)|
|270|[Closest Binary Search Tree Value](https://leetcode.com/problems/closest-binary-search-tree-value)|[closest-binary-search-tree-value.cc](/leetcode/closest-binary-search-tree-value.cc)|
|269|[Alien Dictionary](https://leetcode.com/problems/alien-dictionary)|[alien-dictionary.cc](/leetcode/alien-dictionary.cc)|
|268|[Missing Number](https://leetcode.com/problems/missing-number)|[missing-number.cc](/leetcode/missing-number.cc)|
|267|[Palindrome Permutation II](https://leetcode.com/problems/palindrome-permutation-ii)|[palindrome-permutation-ii.cc](/leetcode/palindrome-permutation-ii.cc)|
|266|[Palindrome Permutation](https://leetcode.com/problems/palindrome-permutation)|[palindrome-permutation.cc](/leetcode/palindrome-permutation.cc)|
|265|[Paint House II](https://leetcode.com/problems/paint-house-ii)|[paint-house-ii.cc](/leetcode/paint-house-ii.cc)|
|264|[Ugly Number II](https://leetcode.com/problems/ugly-number-ii)|[ugly-number-ii.cc](/leetcode/ugly-number-ii.cc)|
|263|[Ugly Number](https://leetcode.com/problems/ugly-number)|[ugly-number.cc](/leetcode/ugly-number.cc)|
|261|[Graph Valid Tree](https://leetcode.com/problems/graph-valid-tree)|[graph-valid-tree.cc](/leetcode/graph-valid-tree.cc)|
|260|[Single Number III](https://leetcode.com/problems/single-number-iii)|[single-number-iii.cc](/leetcode/single-number-iii.cc)|
|259|[3Sum Smaller](https://leetcode.com/problems/3sum-smaller)|[3sum-smaller.cc](/leetcode/3sum-smaller.cc)|
|258|[Add Digits](https://leetcode.com/problems/add-digits)|[add-digits.cc](/leetcode/add-digits.cc)|
|257|[Binary Tree Paths](https://leetcode.com/problems/binary-tree-paths)|[binary-tree-paths.cc](/leetcode/binary-tree-paths.cc)|
|256|[Paint House](https://leetcode.com/problems/paint-house)|[paint-house.cc](/leetcode/paint-house.cc)|
|255|[Verify Preorder Sequence in Binary Search Tree](https://leetcode.com/problems/verify-preorder-sequence-in-binary-search-tree)|[verify-preorder-sequence-in-binary-search-tree.cc](/leetcode/verify-preorder-sequence-in-binary-search-tree.cc)|
|254|[Factor Combinations](https://leetcode.com/problems/factor-combinations)|[factor-combinations.cc](/leetcode/factor-combinations.cc)|
|253|[Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii)|[meeting-rooms-ii.cc](/leetcode/meeting-rooms-ii.cc)|
|252|[Meeting Rooms](https://leetcode.com/problems/meeting-rooms)|[meeting-rooms.cc](/leetcode/meeting-rooms.cc)|
|251|[Flatten 2D Vector](https://leetcode.com/problems/flatten-2d-vector)|[flatten-2d-vector.cc](/leetcode/flatten-2d-vector.cc)|
|250|[Count Univalue Subtrees](https://leetcode.com/problems/count-univalue-subtrees)|[count-univalue-subtrees.cc](/leetcode/count-univalue-subtrees.cc)|
|249|[Group Shifted Strings](https://leetcode.com/problems/group-shifted-strings)|[group-shifted-strings.cc](/leetcode/group-shifted-strings.cc)|
|248|[Strobogrammatic Number III](https://leetcode.com/problems/strobogrammatic-number-iii)|[strobogrammatic-number-iii.cc](/leetcode/strobogrammatic-number-iii.cc)|
|247|[Strobogrammatic Number II](https://leetcode.com/problems/strobogrammatic-number-ii)|[strobogrammatic-number-ii.cc](/leetcode/strobogrammatic-number-ii.cc)|
|246|[Strobogrammatic Number](https://leetcode.com/problems/strobogrammatic-number)|[strobogrammatic-number.cc](/leetcode/strobogrammatic-number.cc)|
|245|[Shortest Word Distance III](https://leetcode.com/problems/shortest-word-distance-iii)|[shortest-word-distance-iii.cc](/leetcode/shortest-word-distance-iii.cc)|
|244|[Shortest Word Distance II](https://leetcode.com/problems/shortest-word-distance-ii)|[shortest-word-distance-ii.cc](/leetcode/shortest-word-distance-ii.cc)|
|243|[Shortest Word Distance](https://leetcode.com/problems/shortest-word-distance)|[shortest-word-distance.cc](/leetcode/shortest-word-distance.cc)|
|242|[Valid Anagram](https://leetcode.com/problems/valid-anagram)|[valid-anagram.cc](/leetcode/valid-anagram.cc)|
|241|[Different Ways to Add Parentheses](https://leetcode.com/problems/different-ways-to-add-parentheses)|[different-ways-to-add-parentheses.cc](/leetcode/different-ways-to-add-parentheses.cc)|
|240|[Search a 2D Matrix II](https://leetcode.com/problems/search-a-2d-matrix-ii)|[search-a-2d-matrix-ii.cc](/leetcode/search-a-2d-matrix-ii.cc)|
|239|[Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum)|[sliding-window-maximum.cc](/leetcode/sliding-window-maximum.cc)|
|238|[Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self)|[product-of-array-except-self.cc](/leetcode/product-of-array-except-self.cc)|
|237|[Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list)|[delete-node-in-a-linked-list.cc](/leetcode/delete-node-in-a-linked-list.cc)|
|236|[Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree)|[lowest-common-ancestor-of-a-binary-tree.cc](/leetcode/lowest-common-ancestor-of-a-binary-tree.cc)|
|235|[Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree)|[lowest-common-ancestor-of-a-binary-search-tree.cc](/leetcode/lowest-common-ancestor-of-a-binary-search-tree.cc)|
|234|[Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list)|[palindrome-linked-list.cc](/leetcode/palindrome-linked-list.cc)|
|233|[Number of Digit One](https://leetcode.com/problems/number-of-digit-one)|[number-of-digit-one.cc](/leetcode/number-of-digit-one.cc)|
|232|[Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks)|[implement-queue-using-stacks.cc](/leetcode/implement-queue-using-stacks.cc)|
|231|[Power of Two](https://leetcode.com/problems/power-of-two)|[power-of-two.cc](/leetcode/power-of-two.cc)|
|230|[Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst)|[kth-smallest-element-in-a-bst.cc](/leetcode/kth-smallest-element-in-a-bst.cc)|
|229|[Majority Element II](https://leetcode.com/problems/majority-element-ii)|[majority-element-ii.cc](/leetcode/majority-element-ii.cc)|
|228|[Summary Ranges](https://leetcode.com/problems/summary-ranges)|[summary-ranges.cc](/leetcode/summary-ranges.cc)|
|227|[Basic Calculator II](https://leetcode.com/problems/basic-calculator-ii)|[basic-calculator-ii.cc](/leetcode/basic-calculator-ii.cc)|
|226|[Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree)|[invert-binary-tree.cc](/leetcode/invert-binary-tree.cc)|
|225|[Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues)|[implement-stack-using-queues.cc](/leetcode/implement-stack-using-queues.cc)|
|224|[Basic Calculator](https://leetcode.com/problems/basic-calculator)|[basic-calculator.cc](/leetcode/basic-calculator.cc)|
|223|[Rectangle Area](https://leetcode.com/problems/rectangle-area)|[rectangle-area.cc](/leetcode/rectangle-area.cc)|
|222|[Count Complete Tree Nodes](https://leetcode.com/problems/count-complete-tree-nodes)|[count-complete-tree-nodes.cc](/leetcode/count-complete-tree-nodes.cc)|
|221|[Maximal Square](https://leetcode.com/problems/maximal-square)|[maximal-square.cc](/leetcode/maximal-square.cc)|
|220|[Contains Duplicate III](https://leetcode.com/problems/contains-duplicate-iii)|[contains-duplicate-iii.cc](/leetcode/contains-duplicate-iii.cc)|
|219|[Contains Duplicate II](https://leetcode.com/problems/contains-duplicate-ii)|[contains-duplicate-ii.cc](/leetcode/contains-duplicate-ii.cc)|
|218|[The Skyline Problem](https://leetcode.com/problems/the-skyline-problem)|[the-skyline-problem.cc](/leetcode/the-skyline-problem.cc)|
|217|[Contains Duplicate](https://leetcode.com/problems/contains-duplicate)|[contains-duplicate.cc](/leetcode/contains-duplicate.cc)|
|216|[Combination Sum III](https://leetcode.com/problems/combination-sum-iii)|[combination-sum-iii.cc](/leetcode/combination-sum-iii.cc)|
|215|[Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array)|[kth-largest-element-in-an-array.cc](/leetcode/kth-largest-element-in-an-array.cc)|
|214|[Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome)|[shortest-palindrome.cc](/leetcode/shortest-palindrome.cc)|
|213|[House Robber II](https://leetcode.com/problems/house-robber-ii)|[house-robber-ii.cc](/leetcode/house-robber-ii.cc)|
|212|[Word Search II](https://leetcode.com/problems/word-search-ii)|[word-search-ii.cc](/leetcode/word-search-ii.cc)|
|210|[Course Schedule II](https://leetcode.com/problems/course-schedule-ii)|[course-schedule-ii.cc](/leetcode/course-schedule-ii.cc)|
|209|[Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum)|[minimum-size-subarray-sum.cc](/leetcode/minimum-size-subarray-sum.cc)|
|208|[Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree)|[implement-trie-prefix-tree.cc](/leetcode/implement-trie-prefix-tree.cc)|
|207|[Course Schedule](https://leetcode.com/problems/course-schedule)|[course-schedule.cc](/leetcode/course-schedule.cc)|
|206|[Reverse Linked List](https://leetcode.com/problems/reverse-linked-list)|[reverse-linked-list.cc](/leetcode/reverse-linked-list.cc)|
|205|[Isomorphic Strings](https://leetcode.com/problems/isomorphic-strings)|[isomorphic-strings.cc](/leetcode/isomorphic-strings.cc)|
|204|[Count Primes](https://leetcode.com/problems/count-primes)|[count-primes.cc](/leetcode/count-primes.cc)|
|203|[Remove Linked List Elements](https://leetcode.com/problems/remove-linked-list-elements)|[remove-linked-list-elements.cc](/leetcode/remove-linked-list-elements.cc)|
|202|[Happy Number](https://leetcode.com/problems/happy-number)|[happy-number.cc](/leetcode/happy-number.cc)|
|201|[Bitwise AND of Numbers Range](https://leetcode.com/problems/bitwise-and-of-numbers-range)|[bitwise-and-of-numbers-range.cc](/leetcode/bitwise-and-of-numbers-range.cc)|
|200|[Number of Islands](https://leetcode.com/problems/number-of-islands)|[number-of-islands.cc](/leetcode/number-of-islands.cc)|
|199|[Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view)|[binary-tree-right-side-view.cc](/leetcode/binary-tree-right-side-view.cc)|
|198|[House Robber](https://leetcode.com/problems/house-robber)|[house-robber.cc](/leetcode/house-robber.cc)|
|191|[Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits)|[number-of-1-bits.cc](/leetcode/number-of-1-bits.cc)|
|190|[Reverse Bits](https://leetcode.com/problems/reverse-bits)|[reverse-bits.cc](/leetcode/reverse-bits.cc)|
|189|[Rotate Array](https://leetcode.com/problems/rotate-array)|[rotate-array.cc](/leetcode/rotate-array.cc)|
|188|[Best Time to Buy and Sell Stock IV](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv)|[best-time-to-buy-and-sell-stock-iv.cc](/leetcode/best-time-to-buy-and-sell-stock-iv.cc)|
|187|[Repeated DNA Sequences](https://leetcode.com/problems/repeated-dna-sequences)|[repeated-dna-sequences.cc](/leetcode/repeated-dna-sequences.cc)|
|186|[Reverse Words in a String II](https://leetcode.com/problems/reverse-words-in-a-string-ii)|[reverse-words-in-a-string-ii.cc](/leetcode/reverse-words-in-a-string-ii.cc)|
|179|[Largest Number](https://leetcode.com/problems/largest-number)|[largest-number.cc](/leetcode/largest-number.cc)|
|174|[Dungeon Game](https://leetcode.com/problems/dungeon-game)|[dungeon-game.cc](/leetcode/dungeon-game.cc)|
|173|[Binary Search Tree Iterator](https://leetcode.com/problems/binary-search-tree-iterator)|[binary-search-tree-iterator.cc](/leetcode/binary-search-tree-iterator.cc)|
|172|[Factorial Trailing Zeroes](https://leetcode.com/problems/factorial-trailing-zeroes)|[factorial-trailing-zeroes.cc](/leetcode/factorial-trailing-zeroes.cc)|
|171|[Excel Sheet Column Number](https://leetcode.com/problems/excel-sheet-column-number)|[excel-sheet-column-number.cc](/leetcode/excel-sheet-column-number.cc)|
|170|[Two Sum III - Data structure design](https://leetcode.com/problems/two-sum-iii-data-structure-design)|[two-sum-iii-data-structure-design.cc](/leetcode/two-sum-iii-data-structure-design.cc)|
|169|[Majority Element](https://leetcode.com/problems/majority-element)|[majority-element.cc](/leetcode/majority-element.cc)|
|168|[Excel Sheet Column Title](https://leetcode.com/problems/excel-sheet-column-title)|[excel-sheet-column-title.cc](/leetcode/excel-sheet-column-title.cc)|
|167|[Two Sum II - Input array is sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted)|[two-sum-ii-input-array-is-sorted.cc](/leetcode/two-sum-ii-input-array-is-sorted.cc)|
|166|[Fraction to Recurring Decimal](https://leetcode.com/problems/fraction-to-recurring-decimal)|[fraction-to-recurring-decimal.cc](/leetcode/fraction-to-recurring-decimal.cc)|
|165|[Compare Version Numbers](https://leetcode.com/problems/compare-version-numbers)|[compare-version-numbers.cc](/leetcode/compare-version-numbers.cc)|
|164|[Maximum Gap](https://leetcode.com/problems/maximum-gap)|[maximum-gap.cc](/leetcode/maximum-gap.cc)|
|163|[Missing Ranges](https://leetcode.com/problems/missing-ranges)|[missing-ranges.cc](/leetcode/missing-ranges.cc)|
|162|[Find Peak Element](https://leetcode.com/problems/find-peak-element)|[find-peak-element.cc](/leetcode/find-peak-element.cc)|
|161|[One Edit Distance](https://leetcode.com/problems/one-edit-distance)|[one-edit-distance.cc](/leetcode/one-edit-distance.cc)|
|160|[Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists)|[intersection-of-two-linked-lists.cc](/leetcode/intersection-of-two-linked-lists.cc)|
|159|[Longest Substring with At Most Two Distinct Characters](https://leetcode.com/problems/longest-substring-with-at-most-two-distinct-characters)|[longest-substring-with-at-most-two-distinct-characters.cc](/leetcode/longest-substring-with-at-most-two-distinct-characters.cc)|
|158|[Read N Characters Given Read4 II - Call multiple times](https://leetcode.com/problems/read-n-characters-given-read4-ii-call-multiple-times)|[read-n-characters-given-read4-ii-call-multiple-times.cc](/leetcode/read-n-characters-given-read4-ii-call-multiple-times.cc)|
|157|[Read N Characters Given Read4](https://leetcode.com/problems/read-n-characters-given-read4)|[read-n-characters-given-read4.cc](/leetcode/read-n-characters-given-read4.cc)|
|156|[Binary Tree Upside Down](https://leetcode.com/problems/binary-tree-upside-down)|[binary-tree-upside-down.cc](/leetcode/binary-tree-upside-down.cc)|
|155|[Min Stack](https://leetcode.com/problems/min-stack)|[min-stack.cc](/leetcode/min-stack.cc)|
|154|[Find Minimum in Rotated Sorted Array II](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array-ii)|[find-minimum-in-rotated-sorted-array-ii.cc](/leetcode/find-minimum-in-rotated-sorted-array-ii.cc)|
|153|[Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array)|[find-minimum-in-rotated-sorted-array.cc](/leetcode/find-minimum-in-rotated-sorted-array.cc)|
|152|[Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray)|[maximum-product-subarray.cc](/leetcode/maximum-product-subarray.cc)|
|151|[Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string)|[reverse-words-in-a-string.cc](/leetcode/reverse-words-in-a-string.cc)|
|150|[Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation)|[evaluate-reverse-polish-notation.cc](/leetcode/evaluate-reverse-polish-notation.cc)|
|149|[Max Points on a Line](https://leetcode.com/problems/max-points-on-a-line)|[max-points-on-a-line.cc](/leetcode/max-points-on-a-line.cc)|
|148|[Sort List](https://leetcode.com/problems/sort-list)|[sort-list.cc](/leetcode/sort-list.cc)|
|147|[Insertion Sort List](https://leetcode.com/problems/insertion-sort-list)|[insertion-sort-list.cc](/leetcode/insertion-sort-list.cc)|
|146|[LRU Cache](https://leetcode.com/problems/lru-cache)|[lru-cache.cc](/leetcode/lru-cache.cc)|
|145|[Binary Tree Postorder Traversal](https://leetcode.com/problems/binary-tree-postorder-traversal)|[binary-tree-postorder-traversal.cc](/leetcode/binary-tree-postorder-traversal.cc)|
|144|[Binary Tree Preorder Traversal](https://leetcode.com/problems/binary-tree-preorder-traversal)|[binary-tree-preorder-traversal.cc](/leetcode/binary-tree-preorder-traversal.cc)|
|143|[Reorder List](https://leetcode.com/problems/reorder-list)|[reorder-list.cc](/leetcode/reorder-list.cc)|
|142|[Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii)|[linked-list-cycle-ii.cc](/leetcode/linked-list-cycle-ii.cc)|
|141|[Linked List Cycle](https://leetcode.com/problems/linked-list-cycle)|[linked-list-cycle.cc](/leetcode/linked-list-cycle.cc)|
|140|[Word Break II](https://leetcode.com/problems/word-break-ii)|[word-break-ii.cc](/leetcode/word-break-ii.cc)|
|139|[Word Break](https://leetcode.com/problems/word-break)|[word-break.cc](/leetcode/word-break.cc)|
|138|[Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer)|[copy-list-with-random-pointer.cc](/leetcode/copy-list-with-random-pointer.cc)|
|137|[Single Number II](https://leetcode.com/problems/single-number-ii)|[single-number-ii.cc](/leetcode/single-number-ii.cc)|
|136|[Single Number](https://leetcode.com/problems/single-number)|[single-number.cc](/leetcode/single-number.cc)|
|135|[Candy](https://leetcode.com/problems/candy)|[candy.cc](/leetcode/candy.cc)|
|134|[Gas Station](https://leetcode.com/problems/gas-station)|[gas-station.cc](/leetcode/gas-station.cc)|
|133|[Clone Graph](https://leetcode.com/problems/clone-graph)|[clone-graph.cc](/leetcode/clone-graph.cc)|
|132|[Palindrome Partitioning II](https://leetcode.com/problems/palindrome-partitioning-ii)|[palindrome-partitioning-ii.cc](/leetcode/palindrome-partitioning-ii.cc)|
|131|[Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning)|[palindrome-partitioning.cc](/leetcode/palindrome-partitioning.cc)|
|130|[Surrounded Regions](https://leetcode.com/problems/surrounded-regions)|[surrounded-regions.cc](/leetcode/surrounded-regions.cc)|
|129|[Sum Root to Leaf Numbers](https://leetcode.com/problems/sum-root-to-leaf-numbers)|[sum-root-to-leaf-numbers.cc](/leetcode/sum-root-to-leaf-numbers.cc)|
|128|[Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence)|[longest-consecutive-sequence.cc](/leetcode/longest-consecutive-sequence.cc)|
|127|[Word Ladder](https://leetcode.com/problems/word-ladder)|[word-ladder.cc](/leetcode/word-ladder.cc)|
|126|[Word Ladder II](https://leetcode.com/problems/word-ladder-ii)|[word-ladder-ii.cc](/leetcode/word-ladder-ii.cc)|
|125|[Valid Palindrome](https://leetcode.com/problems/valid-palindrome)|[valid-palindrome.cc](/leetcode/valid-palindrome.cc)|
|124|[Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum)|[binary-tree-maximum-path-sum.cc](/leetcode/binary-tree-maximum-path-sum.cc)|
|123|[Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii)|[best-time-to-buy-and-sell-stock-iii.cc](/leetcode/best-time-to-buy-and-sell-stock-iii.cc)|
|122|[Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii)|[best-time-to-buy-and-sell-stock-ii.cc](/leetcode/best-time-to-buy-and-sell-stock-ii.cc)|
|121|[Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock)|[best-time-to-buy-and-sell-stock.cc](/leetcode/best-time-to-buy-and-sell-stock.cc)|
|120|[Triangle](https://leetcode.com/problems/triangle)|[triangle.cc](/leetcode/triangle.cc)|
|119|[Pascal's Triangle II](https://leetcode.com/problems/pascals-triangle-ii)|[pascals-triangle-ii.cc](/leetcode/pascals-triangle-ii.cc)|
|118|[Pascal's Triangle](https://leetcode.com/problems/pascals-triangle)|[pascals-triangle.cc](/leetcode/pascals-triangle.cc)|
|117|[Populating Next Right Pointers in Each Node II](https://leetcode.com/problems/populating-next-right-pointers-in-each-node-ii)|[populating-next-right-pointers-in-each-node-ii.cc](/leetcode/populating-next-right-pointers-in-each-node-ii.cc)|
|116|[Populating Next Right Pointers in Each Node](https://leetcode.com/problems/populating-next-right-pointers-in-each-node)|[populating-next-right-pointers-in-each-node.cc](/leetcode/populating-next-right-pointers-in-each-node.cc)|
|115|[Distinct Subsequences](https://leetcode.com/problems/distinct-subsequences)|[distinct-subsequences.cc](/leetcode/distinct-subsequences.cc)|
|114|[Flatten Binary Tree to Linked List](https://leetcode.com/problems/flatten-binary-tree-to-linked-list)|[flatten-binary-tree-to-linked-list.cc](/leetcode/flatten-binary-tree-to-linked-list.cc)|
|113|[Path Sum II](https://leetcode.com/problems/path-sum-ii)|[path-sum-ii.cc](/leetcode/path-sum-ii.cc)|
|112|[Path Sum](https://leetcode.com/problems/path-sum)|[path-sum.cc](/leetcode/path-sum.cc)|
|111|[Minimum Depth of Binary Tree](https://leetcode.com/problems/minimum-depth-of-binary-tree)|[minimum-depth-of-binary-tree.cc](/leetcode/minimum-depth-of-binary-tree.cc)|
|110|[Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree)|[balanced-binary-tree.cc](/leetcode/balanced-binary-tree.cc)|
|109|[Convert Sorted List to Binary Search Tree](https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree)|[convert-sorted-list-to-binary-search-tree.cc](/leetcode/convert-sorted-list-to-binary-search-tree.cc)|
|108|[Convert Sorted Array to Binary Search Tree](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree)|[convert-sorted-array-to-binary-search-tree.cc](/leetcode/convert-sorted-array-to-binary-search-tree.cc)|
|107|[Binary Tree Level Order Traversal II](https://leetcode.com/problems/binary-tree-level-order-traversal-ii)|[binary-tree-level-order-traversal-ii.cc](/leetcode/binary-tree-level-order-traversal-ii.cc)|
|106|[Construct Binary Tree from Inorder and Postorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-inorder-and-postorder-traversal)|[construct-binary-tree-from-inorder-and-postorder-traversal.cc](/leetcode/construct-binary-tree-from-inorder-and-postorder-traversal.cc)|
|105|[Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal)|[construct-binary-tree-from-preorder-and-inorder-traversal.cc](/leetcode/construct-binary-tree-from-preorder-and-inorder-traversal.cc)|
|104|[Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree)|[maximum-depth-of-binary-tree.cc](/leetcode/maximum-depth-of-binary-tree.cc)|
|103|[Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal)|[binary-tree-zigzag-level-order-traversal.cc](/leetcode/binary-tree-zigzag-level-order-traversal.cc)|
|102|[Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal)|[binary-tree-level-order-traversal.cc](/leetcode/binary-tree-level-order-traversal.cc)|
|101|[Symmetric Tree](https://leetcode.com/problems/symmetric-tree)|[symmetric-tree.cc](/leetcode/symmetric-tree.cc)|
|100|[Same Tree](https://leetcode.com/problems/same-tree)|[same-tree.cc](/leetcode/same-tree.cc)|
|99|[Recover Binary Search Tree](https://leetcode.com/problems/recover-binary-search-tree)|[recover-binary-search-tree.cc](/leetcode/recover-binary-search-tree.cc)|
|98|[Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree)|[validate-binary-search-tree.cc](/leetcode/validate-binary-search-tree.cc)|
|97|[Interleaving String](https://leetcode.com/problems/interleaving-string)|[interleaving-string.cc](/leetcode/interleaving-string.cc)|
|96|[Unique Binary Search Trees](https://leetcode.com/problems/unique-binary-search-trees)|[unique-binary-search-trees.cc](/leetcode/unique-binary-search-trees.cc)|
|95|[Unique Binary Search Trees II](https://leetcode.com/problems/unique-binary-search-trees-ii)|[unique-binary-search-trees-ii.cc](/leetcode/unique-binary-search-trees-ii.cc)|
|94|[Binary Tree Inorder Traversal](https://leetcode.com/problems/binary-tree-inorder-traversal)|[binary-tree-inorder-traversal.cc](/leetcode/binary-tree-inorder-traversal.cc)|
|93|[Restore IP Addresses](https://leetcode.com/problems/restore-ip-addresses)|[restore-ip-addresses.cc](/leetcode/restore-ip-addresses.cc)|
|92|[Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii)|[reverse-linked-list-ii.cc](/leetcode/reverse-linked-list-ii.cc)|
|91|[Decode Ways](https://leetcode.com/problems/decode-ways)|[decode-ways.cc](/leetcode/decode-ways.cc)|
|90|[Subsets II](https://leetcode.com/problems/subsets-ii)|[subsets-ii.cc](/leetcode/subsets-ii.cc)|
|89|[Gray Code](https://leetcode.com/problems/gray-code)|[gray-code.cc](/leetcode/gray-code.cc)|
|88|[Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array)|[merge-sorted-array.cc](/leetcode/merge-sorted-array.cc)|
|87|[Scramble String](https://leetcode.com/problems/scramble-string)|[scramble-string.cc](/leetcode/scramble-string.cc)|
|86|[Partition List](https://leetcode.com/problems/partition-list)|[partition-list.cc](/leetcode/partition-list.cc)|
|85|[Maximal Rectangle](https://leetcode.com/problems/maximal-rectangle)|[maximal-rectangle.cc](/leetcode/maximal-rectangle.cc)|
|84|[Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram)|[largest-rectangle-in-histogram.cc](/leetcode/largest-rectangle-in-histogram.cc)|
|83|[Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list)|[remove-duplicates-from-sorted-list.cc](/leetcode/remove-duplicates-from-sorted-list.cc)|
|82|[Remove Duplicates from Sorted List II](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii)|[remove-duplicates-from-sorted-list-ii.cc](/leetcode/remove-duplicates-from-sorted-list-ii.cc)|
|81|[Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii)|[search-in-rotated-sorted-array-ii.cc](/leetcode/search-in-rotated-sorted-array-ii.cc)|
|80|[Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii)|[remove-duplicates-from-sorted-array-ii.cc](/leetcode/remove-duplicates-from-sorted-array-ii.cc)|
|79|[Word Search](https://leetcode.com/problems/word-search)|[word-search.cc](/leetcode/word-search.cc)|
|78|[Subsets](https://leetcode.com/problems/subsets)|[subsets.cc](/leetcode/subsets.cc)|
|77|[Combinations](https://leetcode.com/problems/combinations)|[combinations.cc](/leetcode/combinations.cc)|
|76|[Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring)|[minimum-window-substring.cc](/leetcode/minimum-window-substring.cc)|
|75|[Sort Colors](https://leetcode.com/problems/sort-colors)|[sort-colors.cc](/leetcode/sort-colors.cc)|
|74|[Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix)|[search-a-2d-matrix.cc](/leetcode/search-a-2d-matrix.cc)|
|73|[Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes)|[set-matrix-zeroes.cc](/leetcode/set-matrix-zeroes.cc)|
|72|[Edit Distance](https://leetcode.com/problems/edit-distance)|[edit-distance.cc](/leetcode/edit-distance.cc)|
|71|[Simplify Path](https://leetcode.com/problems/simplify-path)|[simplify-path.cc](/leetcode/simplify-path.cc)|
|70|[Climbing Stairs](https://leetcode.com/problems/climbing-stairs)|[climbing-stairs.cc](/leetcode/climbing-stairs.cc)|
|69|[Sqrt(x)](https://leetcode.com/problems/sqrtx)|[sqrtx.cc](/leetcode/sqrtx.cc)|
|68|[Text Justification](https://leetcode.com/problems/text-justification)|[text-justification.cc](/leetcode/text-justification.cc)|
|67|[Add Binary](https://leetcode.com/problems/add-binary)|[add-binary.cc](/leetcode/add-binary.cc)|
|66|[Plus One](https://leetcode.com/problems/plus-one)|[plus-one.cc](/leetcode/plus-one.cc)|
|65|[Valid Number](https://leetcode.com/problems/valid-number)|[valid-number.cc](/leetcode/valid-number.cc)|
|64|[Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum)|[minimum-path-sum.cc](/leetcode/minimum-path-sum.cc)|
|63|[Unique Paths II](https://leetcode.com/problems/unique-paths-ii)|[unique-paths-ii.cc](/leetcode/unique-paths-ii.cc)|
|62|[Unique Paths](https://leetcode.com/problems/unique-paths)|[unique-paths.cc](/leetcode/unique-paths.cc)|
|61|[Rotate List](https://leetcode.com/problems/rotate-list)|[rotate-list.cc](/leetcode/rotate-list.cc)|
|60|[Permutation Sequence](https://leetcode.com/problems/permutation-sequence)|[permutation-sequence.cc](/leetcode/permutation-sequence.cc)|
|59|[Spiral Matrix II](https://leetcode.com/problems/spiral-matrix-ii)|[spiral-matrix-ii.cc](/leetcode/spiral-matrix-ii.cc)|
|58|[Length of Last Word](https://leetcode.com/problems/length-of-last-word)|[length-of-last-word.cc](/leetcode/length-of-last-word.cc)|
|57|[Insert Interval](https://leetcode.com/problems/insert-interval)|[insert-interval.cc](/leetcode/insert-interval.cc)|
|56|[Merge Intervals](https://leetcode.com/problems/merge-intervals)|[merge-intervals.cc](/leetcode/merge-intervals.cc)|
|55|[Jump Game](https://leetcode.com/problems/jump-game)|[jump-game.cc](/leetcode/jump-game.cc)|
|54|[Spiral Matrix](https://leetcode.com/problems/spiral-matrix)|[spiral-matrix.cc](/leetcode/spiral-matrix.cc)|
|53|[Maximum Subarray](https://leetcode.com/problems/maximum-subarray)|[maximum-subarray.cc](/leetcode/maximum-subarray.cc)|
|52|[N-Queens II](https://leetcode.com/problems/n-queens-ii)|[n-queens-ii.cc](/leetcode/n-queens-ii.cc)|
|51|[N-Queens](https://leetcode.com/problems/n-queens)|[n-queens.cc](/leetcode/n-queens.cc)|
|50|[Pow(x, n)](https://leetcode.com/problems/powx-n)|[powx-n.cc](/leetcode/powx-n.cc)|
|48|[Rotate Image](https://leetcode.com/problems/rotate-image)|[rotate-image.cc](/leetcode/rotate-image.cc)|
|47|[Permutations II](https://leetcode.com/problems/permutations-ii)|[permutations-ii.cc](/leetcode/permutations-ii.cc)|
|46|[Permutations](https://leetcode.com/problems/permutations)|[permutations.cc](/leetcode/permutations.cc)|
|45|[Jump Game II](https://leetcode.com/problems/jump-game-ii)|[jump-game-ii.cc](/leetcode/jump-game-ii.cc)|
|44|[Wildcard Matching](https://leetcode.com/problems/wildcard-matching)|[wildcard-matching.cc](/leetcode/wildcard-matching.cc)|
|43|[Multiply Strings](https://leetcode.com/problems/multiply-strings)|[multiply-strings.cc](/leetcode/multiply-strings.cc)|
|42|[Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water)|[trapping-rain-water.cc](/leetcode/trapping-rain-water.cc)|
|41|[First Missing Positive](https://leetcode.com/problems/first-missing-positive)|[first-missing-positive.cc](/leetcode/first-missing-positive.cc)|
|40|[Combination Sum II](https://leetcode.com/problems/combination-sum-ii)|[combination-sum-ii.cc](/leetcode/combination-sum-ii.cc)|
|39|[Combination Sum](https://leetcode.com/problems/combination-sum)|[combination-sum.cc](/leetcode/combination-sum.cc)|
|38|[Count and Say](https://leetcode.com/problems/count-and-say)|[count-and-say.cc](/leetcode/count-and-say.cc)|
|37|[Sudoku Solver](https://leetcode.com/problems/sudoku-solver)|[sudoku-solver.cc](/leetcode/sudoku-solver.cc)|
|36|[Valid Sudoku](https://leetcode.com/problems/valid-sudoku)|[valid-sudoku.cc](/leetcode/valid-sudoku.cc)|
|35|[Search Insert Position](https://leetcode.com/problems/search-insert-position)|[search-insert-position.cc](/leetcode/search-insert-position.cc)|
|33|[Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array)|[search-in-rotated-sorted-array.cc](/leetcode/search-in-rotated-sorted-array.cc)|
|32|[Longest Valid Parentheses](https://leetcode.com/problems/longest-valid-parentheses)|[longest-valid-parentheses.cc](/leetcode/longest-valid-parentheses.cc)|
|31|[Next Permutation](https://leetcode.com/problems/next-permutation)|[next-permutation.cc](/leetcode/next-permutation.cc)|
|30|[Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words)|[substring-with-concatenation-of-all-words.cc](/leetcode/substring-with-concatenation-of-all-words.cc)|
|29|[Divide Two Integers](https://leetcode.com/problems/divide-two-integers)|[divide-two-integers.cc](/leetcode/divide-two-integers.cc)|
|28|[Implement strStr()](https://leetcode.com/problems/implement-strstr)|[implement-strstr.cc](/leetcode/implement-strstr.cc)|
|27|[Remove Element](https://leetcode.com/problems/remove-element)|[remove-element.cc](/leetcode/remove-element.cc)|
|26|[Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array)|[remove-duplicates-from-sorted-array.cc](/leetcode/remove-duplicates-from-sorted-array.cc)|
|25|[Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group)|[reverse-nodes-in-k-group.cc](/leetcode/reverse-nodes-in-k-group.cc)|
|24|[Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs)|[swap-nodes-in-pairs.cc](/leetcode/swap-nodes-in-pairs.cc)|
|23|[Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists)|[merge-k-sorted-lists.cc](/leetcode/merge-k-sorted-lists.cc)|
|22|[Generate Parentheses](https://leetcode.com/problems/generate-parentheses)|[generate-parentheses.cc](/leetcode/generate-parentheses.cc)|
|21|[Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists)|[merge-two-sorted-lists.cc](/leetcode/merge-two-sorted-lists.cc)|
|20|[Valid Parentheses](https://leetcode.com/problems/valid-parentheses)|[valid-parentheses.cc](/leetcode/valid-parentheses.cc)|
|19|[Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list)|[remove-nth-node-from-end-of-list.cc](/leetcode/remove-nth-node-from-end-of-list.cc)|
|18|[4Sum](https://leetcode.com/problems/4sum)|[4sum.cc](/leetcode/4sum.cc)|
|17|[Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number)|[letter-combinations-of-a-phone-number.cc](/leetcode/letter-combinations-of-a-phone-number.cc)|
|16|[3Sum Closest](https://leetcode.com/problems/3sum-closest)|[3sum-closest.cc](/leetcode/3sum-closest.cc)|
|15|[3Sum](https://leetcode.com/problems/3sum)|[3sum.cc](/leetcode/3sum.cc)|
|14|[Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix)|[longest-common-prefix.cc](/leetcode/longest-common-prefix.cc)|
|13|[Roman to Integer](https://leetcode.com/problems/roman-to-integer)|[roman-to-integer.cc](/leetcode/roman-to-integer.cc)|
|12|[Integer to Roman](https://leetcode.com/problems/integer-to-roman)|[integer-to-roman.cc](/leetcode/integer-to-roman.cc)|
|11|[Container With Most Water](https://leetcode.com/problems/container-with-most-water)|[container-with-most-water.cc](/leetcode/container-with-most-water.cc)|
|10|[Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching)|[regular-expression-matching.cc](/leetcode/regular-expression-matching.cc)|
|9|[Palindrome Number](https://leetcode.com/problems/palindrome-number)|[palindrome-number.cc](/leetcode/palindrome-number.cc)|
|8|[String to Integer (atoi)](https://leetcode.com/problems/string-to-integer-atoi)|[string-to-integer-atoi.cc](/leetcode/string-to-integer-atoi.cc)|
|7|[Reverse Integer](https://leetcode.com/problems/reverse-integer)|[reverse-integer.cc](/leetcode/reverse-integer.cc)|
|6|[ZigZag Conversion](https://leetcode.com/problems/zigzag-conversion)|[zigzag-conversion.cc](/leetcode/zigzag-conversion.cc)|
|5|[Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring)|[longest-palindromic-substring.cc](/leetcode/longest-palindromic-substring.cc)|
|4|[Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays)|[median-of-two-sorted-arrays.cc](/leetcode/median-of-two-sorted-arrays.cc)|
|3|[Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters)|[longest-substring-without-repeating-characters.cc](/leetcode/longest-substring-without-repeating-characters.cc)|
|2|[Add Two Numbers](https://leetcode.com/problems/add-two-numbers)|[add-two-numbers.cc](/leetcode/add-two-numbers.cc)|
|1|[Two Sum](https://leetcode.com/problems/two-sum)|[two-sum.cc](/leetcode/two-sum.cc)|


```javascript
copy($('td:has(.ac) ~ td:nth-child(3) a').map(function(_,e){
  var id = $(e).parent().prev().text();
  var h=e.href.replace(/htt.*problems/,'/leetcode');h=h.substr(0,h.length-1);
  var title=e.textContent, href=e.href, name=h.replace(/.*\//,'');
  return '|'+id+'|['+title+']('+href+')|['+name+'.cc]('+name+'.cc)|'
}).toArray().join('\n'))
```

```ruby
require 'mechanize'
agent = Mechanize.new
page = agent.get 'https://leetcode.com/accounts/login/'
doc = page.form_with {|form|
  form['login'] = 'MaskRay'
  form['password'] = getpass
}.submit.parser
total = doc.css('td:nth-child(3)').size
solved = doc.css('td:has(.ac)').size
puts "You have solved #{solved}/#{total} problems."
for a in doc.css('td:nth-child(3) a')
  id = a.parent.previous_element.text
  href = a['href']
  name = href.sub(/\/problems\/(.*)\//, '\1')
  title = a.text
  puts "|#{id}|[#{title}](#{href})|[#{name}.cc](#{name}.cc)|"
end
```
