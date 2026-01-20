---
layout: post
title: gcov與LLVM中的實現
author: MaskRay
tags: [llvm,gcc]
mathjax: true
---

打算以後不定期寫一點LLVM的學習(開發)筆記。寫作上不想過多花時間(加語文水平所限...)，所以字句不作過多斟酌。

## gcov

<https://gcc.gnu.org/onlinedocs/gcc/Gcov.html>

_Optimally Profiling and Tracing Programs_描述了一個edge-frequency/edge-placement problem。
選擇control-flow graph的一些邊，加上監控代碼，推導所有邊的執行次數。
gcov是一種實現。

在gcov的模型中，一個源文件包含若干函數，一個函數包含若干基本塊，一個基本塊佔據若干行，這些信息保存在`.gcno`文件中。
Instrument程序，在基本塊間轉移時記錄邊的執行次數，程序退出時爲每個translation unit輸出一個`.gcda`文件。
`.gcda`文件可以累計多次程序執行的計數。

<!-- more -->

根據`.gcno`描述的圖信息和`.gcda`記錄的邊的執行次數，可以計算基本塊的執行次數，進而計算行的執行次數。

使用流程：

* `gcc --coverage a.c b.c` 生成`a.gcno`和`b.gcno`，instrument(記錄邊執行次數)並鏈接`libgcov.a`
* `./a.out` 程序退出時輸出`a.gcda`和`b.gcda`
* `gcov -t a. b.` 生成coverage報告

有兩個主要GCC選項：

* `-ftest-profile`: 輸出notes file (`.gcno`)，描述GCC版本、函數基本信息(名稱、起始行列號、結束行列號(GCC 8添加)、校驗碼(源文件更新後可以判斷`.gcno`/`.gcda`是否過期))、基本塊 (flags (是否樹邊、是否異常控制流添加的)、包含的行)
* `-fprofile-arcs`: instrument程序，程序執行後更新data files (`.gcda`)，描述程序執行次數和邊執行次數。可以設置環境變量`GCOV_PREFIX`控制`.gcda`文件的創建路徑

`.gcno/.gcda`均由一串uint32組成，文件格式參見`gcc/gcov-io.h`。編碼簡單，但是空間使用有點浪費。
簡單說來，`-ftest-profile`描述控制流圖而`-fprofile-arcs`instrument程序。這兩個選項在GCC 3.4定形，之前版本用`.bb{,g}`和`.da`。

`-fprofile-arcs`亦可用作鏈接選項，鏈接`libgcov.{a,so}`。通常`-ftest-profile -fprofile-arcs`同時指定，可以簡寫爲`--coverage`。(目前爲止我唯一的CPython commit就是把`-lgcov`換成了`--coverage`)

gcov有個經典前端lcov，原先爲Linux kernel設計。lcov設計的一套中間格式應用非常廣泛，比如Bazel的coverage功能就用lcov作爲所有程序設計語言coverage的中間格式。
<http://ltp.sourceforge.net/coverage/lcov/geninfo.1.php>

gcov常用的命令行選項：

* `-i`: GCC 4.9~8生成一個近似lcov的intermediate format。GCC 9之後生成`.json.gz`(有點討厭的上游改動～)。如果只是想單純看行執行次數，從JSON提取信息比提取intermediate format麻煩多了
* `-r`: 只輸出編譯時(源文件使用相對路徑名)的.gcno/.gcda的信息。可跳過系統頭文件
* `-s prefix`: 若源文件路徑名前綴爲prefix，去除後變爲相對路徑，可使`-r`生效
* `-t`: 輸出到stdout，而非`.gcov`文件

GCC 4.7、4.8、8、9有打破backward compatibility的改動。如果編譯程序用的GCC版本和gcov版本不一致會得到warning，如果沒有breaking changes，其實是良性的：

```text
# gcov removes the extension name, so you can use any of a. a.c a.gcno a.gcda
% gcc-9 --coverage a.c; gcov-10 -t a.
a.gcno:version 'A93*', prefer 'B01*'
a.gcda:version 'A93*', prefer version 'B01*'
...
```

## 行執行次數

源文件和函數的summary info都會輸出`Lines executed`。
對於某個基本塊佔據的一行，標記該基本塊所在的函數包含該行。若該基本塊執行次數大於0，則給所在的函數的lines_executed計數器+1。

GCC 10的gcov statistics實現的基本流程：

```text
add_line_counts(coverage_info *coverage, function_info *fn)
  // Accmulate info for one function
  for each block
    for every location (a pair of (source file index, lines))
      for every line
        if fn is grouped && line belongs to the range of fn
          set line to the line_info of the function
          line->exists = 1
          increment line->count
        else
          set line to the line_info of the file
          line->exists = 1
          increment function's lines & lines_executed if necessary
          increment line->count

accumulate_line_info(line_info *line, source_info *src, bool add_coverage)
  // Accumulate info for one line
  for each outgoing arc
    arc->cs_count = arc->count
  compute cycle counts
  line count = incoming counts + cycle counts
  if line->exists && add_coverage
    src->coverage.lines++
    if line->count
      src->coverage.lines_executed++

accumulate_line_counts(source_info *src)
  // Accumulate info for one file
  for each grouped function
    for each line
      call accumulate_line_info with add_coverage=false
  for each line
    call accumulate_line_info with add_coverage=true
  for each grouped function
    for each fn_line where fn_line->exists
      increment src's lines and lines_executed if necessary

generate_results(const char *filename)
  for each function
    call add_line_counts
  for each source file
    call accumulate_line_counts
    call file_summary
```

得到lines和lines_executed後，Lines executed的百分比就是簡單的lines和lines_executed的商。

`is_group`概念是用於處理在同一位置的多個函數的(如template; <https://gcc.gnu.org/PR48463>)。gcov -f會顯示多個函數：

```text
------------------
| _ZN3FooIfEC2Ev:
|    6|      2|  {
|    7|      2|    b = 123;
|    8|      2|  }
------------------
| _ZN3FooIiEC2Ev:
|    6|      1|  {
|    7|      1|    b = 123;
|    8|      1|  }
------------------
```

### 循環執行次數

行執行次數(上面僞代碼的`compute cycle counts`)使用了一個啓發式規則：incoming counts+cycle counts。
Incoming counts：不在同一行的前驅進入該基本塊的執行次數和。
Cycle counts：在同一行的基本塊之間的循環執行次數。可以這樣理解：如果有一個循環寫在同一行上(macro裏的循環也屬於這一類)，那麼循環執行次數也應該計入該行的執行次數。

對於一個for，它的執行次數部分來自上一行(entering block(s))，部分來自循環內的latch(有back edge回到header)。

對於下面的簡單程序，相關的基本塊如下：

* bb0: entry block
* bb1: exit block (not considered here)
* bb2: entering block (also preheader)
* bb5: header
* bb3: loop body
* bb4: latch

它們之間的執行次數：(bb0,bb2,1), (bb2,bb5,1), (bb5,bb3,11), (bb3,bb4,11), (bb4,bb5,11)
根據編譯器版本，bb5可能會佔據for所在行，或不佔據，有下面兩種方案。
使用incoming counts+cycle counts的策略可以計算出相同的行執行次數結果(for循環計算爲執行了12次)。

```c
int main() {                    // bb2
  for (int i = 0; i < 11; i++)  // bb2, bb4, bb5
    i = i;                      // bb3
}
```

```c
int main() {                    // bb2
  for (int i = 0; i < 11; i++)  // bb2, bb4
    i = i;                      // bb3
}
```

循環執行次數是network flow problems中circulation problem的應用：maximum circulation就是循環流量。

gcov原先使用 _J. C. Tiernan, An Efficient Search Algorithm to Find the Elementary Circuits of a Graph, Comm ACM 1970_ 描述的算法(the worst-case time bound is exponential in the number of elementary circuits)，枚舉圈(cycle, aka simple circuit, aka elementary circuit)然後cycle cancelling。
2016年[GCC PR67992](https://gcc.gnu.org/PR67992)用了改進的Donald B. Johnson算法，時間複雜度：$O((V+E)(c+1))$，其中c是圈數。最壞情況下也是exponential in the size of the graph的。
(Boost把該算法歸功於K. A. Hawick and H. A. James。gcov繼承了這一名字。但實際上這篇論文並沒有對Johnson的算法進行改進)

事實上每次消圈時至少把一條邊的流量降低爲0，因此最多有$O(E)$個圈。[PR90380 (target: GCC 8.3)](http://gcc.gnu.org/PR90380)跳過非正數邊，把時間複雜度降低到$O(V*E^2)$(理論可以做到$O(E^2)$，但實現中有一個線性的比較)。

llvm-cov gcov也有GCC PR90380修復的指數複雜度問題，該問題被<https://reviews.llvm.org/D93036>修復。
實際上，找$O(E)$個圈使用cycle enumeration的Donald B. Johnson算法意義不大，可以換成普通的cycle detection。
<https://reviews.llvm.org/D93073>實現了簡化，將複雜度降低到$O(E^2)$。

進一步思考：我們處理的是reducible flow graph(對於irreducible flow graph沒有直觀合理的cycle counts)，每一個natural loop都有一個back edge標識。因此構建dominator tree，找back edges，找出natural loops並把arcs計數器清零(清零的原因是我們還要計算從其他行進入的執行次數，要避免重複計數)，複雜度可以優化到$O(depthOfNestedLoops*E)$。實現semi-NCA算法(複雜度是$O(V^2)$)並不複雜，但找出natural loops會比較麻煩。因此找natural loops的方案沒有實用價值。

```text
construct dominator tree
find back edges (the arc tail dominates the arc header)
for each back edge
  find the associated natural loop
  increase cycle counts
  decrease arc->count for each in-loop arc
compute incoming counts
line count = incoming counts + cycle counts
```

## gcov in LLVM

最早由Nick Lewycky在2011年貢獻。

使用流程示意：

* `clang --coverage -c a.c b.c` 生成`a.gcno`和`b.gcno`，instrument(記錄邊執行次數)
* `clang --coverage a.o b.o` 鏈接`libclang_rt.profile-$arch.a` (實現在compiler-rt)
* `./a.out` 程序退出時輸出`a.gcda`和`b.gcda`
* `llvm-cov gcov -t a. b.` 生成coverage報告

llvm-cov gcov接口模擬gcov。在Clang 11之前(我施工之前)，Clang吐出的`.gcno/.gcda`是一種僞GCC 4.2格式，實際上沒有辦法用任何gcov版本閱讀:(

## Instrumentation

Instrument一條邊的指令非常簡單，給一個計數器加一。多線程程序同時操作一個計數器就會有data race。
GCC追求精確，如果編譯時指定`-pthread`，計數器會用atomic operation，多線程競爭帶來的性能損失可能很大(<https://gcc.gnu.org/PR97065>)。
Clang `-fprofile-generate`和`-fprofile-arcs`沒有用atomic operation。多線程併發執行可能會有部分計數丟失。

```
// g++
addq    $1, __gcov0._Z3fooRSt6vectorIiSaIiEE(%rip)
// g++ -pthread
lock addq       $1, __gcov0._Z3fooRSt6vectorIiSaIiEE(%rip)
```

Clang的PGO會檢測循環，把計數器+1轉換爲+N，在性能提升的同時可以大幅改善多線程衝突導致計數丟失的問題(<https://reviews.llvm.org/D34085>)。

替代atomic operation的一種思路是在function prologue處獲取CPU id(可惜沒有移植性好的函數)，獲得一個per-cpu counter array。
函數內的所有計數器都相對於該array，假設不發生process migration。
Linux上若有rseq倒是可以訪問`__rseq_abi.cpu_id`，但使用seq需要runtime在每個新建thread上執行rseq系統調用……
(假如要保護process migration，每個計數器增加都用rseq會消耗過多`__rseq_cs`)
這種方法還有一個缺點是一個寄存器被固定用作計數器用途。

PGO/coverage選項可以參閱<https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html>。
GCC的profile guided optimization `-fprofile-generate`隱含`-fprofile-arcs`，亦即PGO和coverage復用同一套instrument模塊和runtime。
這麼做也有缺點，因爲PGO和coverage取捨上有所不同。比如coverage用戶可能會更在意行執行次數的準確性，假如三個計時器有A=B+C的關係而實際上不成立，會給用戶造成困惑。
而對於PGO來說，損失10%的邊執行次數未嘗不可接受。

Clang其實還有另一套coverage機制，在2014年開發：`-fprofile-instr-generate -fcoverage-mapping`。
clangCodeGen實現了`-fprofile-instr-generate`，既用於Clang的PGO instrumentation，也用於coverage。
假如你用llvm-cov導出coverage信息，可以發現它有非常精確的列號和覆蓋區間信息，因爲它能直接操作AST，知道每一個語句的起始和結束位置。
`--coverage`則用了調試信息，大部分時候只有點信息，而沒有區間信息，甚至有些時候沒有列號信息。

`-fprofile-instr-generate` instrument一個函數時記錄所有邊的執行次數。假如你學過電路，可能知道Kirchhoff's circuit law。
若一個頂點有E條邊，那麼記錄E-1條邊的執行次數，即可計算剩下一條邊的執行次數。
對於一個V個基本塊、E條邊的圖，只需要測量E-V+1條邊即可計算所有邊的執行次數。
在控制流圖中添加一個單個後繼的basic block，不需要instrument新邊。在控制流圖中添加一個兩個後繼的basic block，只需要instrument一條新邊。
這一優化可以少instrument超過一半的邊。我最近給Clang 12添加了這一優化，可能有10%~30%的性能提升，性能和關閉value profiling的`-fprofile-generate`類似，比`-fprofile-instr-generate`快。

當然這麼做並非沒有缺陷：對於異常控制流，如fork/execve，少instrument一些邊可以導致計數不準確。

2015年Clang添加了`-fprofile-generate`(選項名模擬GCC，實際上並不兼容)，IR-level PGO。作爲一個IR pass，它在優化管線內的位置比較靈活，而且可以方便地用於其他語言前端(ldc、Rust)。
比如，在instrument前添加一個inliner可以大幅減少記錄的邊數。GCC也有類似優化。對於下面的程序，bar很有可能沒有行執行次數：

```c
int bar() {
  return 1;
}

__attribute__((noinline))
int foo() {
  return bar() + 2;
}
```

`clang --coverage`目前沒有在instrument前添加inliner，因此bar的執行次數被準確地保留下來。
你可以把bar替換成一個`/usr/include/c++/`裏的某個C++ STL，先執行inling的好處就更加明顯了：
很多時候你不想瞭解C++ STL裏複雜實現的各個部分的行執行次數，然而他們卻佔據了好多計數器。

還有一個有意思的地方，如果用了`--coverage -fsanitize=thread`，thread sanitizer會添加在優化管線非常靠後的地方。
計數器應該用atomic operation，否則會觸發data race報告。我在Clang 12中修復了這個問題。

C++的函數默認都會拋異常😹。爲了計數器精確，GCC會拆分基本塊，添加fake arcs，使得一個外部函數調用和下一條語句的計數可能不同。對於下面的程序，bb 0和bb 1分別表示entry block/exit block，bb 2~5之間有邊。

```cpp
extern void f();
void g() {   // bb 2
  puts("0"); // bb 2
  f();       // bb 3
  puts("1"); // bb 4
}            // bb 5
```

如果沒有異常、dxit、execve、fork等干涉控制流的函數，函數g只需要一個計數器，讓所有語句共享。而GCC從bb 2~4各引一條邊到bb 1，導致了三個新的計數器。Clang沒有實現fake arcs。

## 我的gcov in LLVM貢獻

大部分修改在LLVM 11.0.0中。

* 幾乎重寫了`llvm-cov gcov`的所有統計代碼，支持讀取GCC 3.4~10 `.gcno/.gcda`
* 給`llvm-cov gcov`實現了`-i -r -s -t`
* 讓Clang和`libclang_rt.profile-`支持GCC 4.7~9所有instrument改動。可以用`-Xclang -coverage-version='409*'`生成GCC 4.9兼容的`.gcno/.gcda`；`A93*`則是GCC 9.3，`B01*`則是GCC 10.1
* 添加了big-endian實現

### lcov

Create `llvm-gcov`:
```sh
#!/bin/sh
exec llvm-cov gcov "$@"
```

```sh
clang --coverage a.c -o a  # generate a.gcno
./a  # generate a.gcda
lcov -c --gcov-tool $PWD/llvm-gcov -d . -o output.lcov
genhtml output.lcov -o out
```

Specify `--base-directory` to specify the source directory.

## gcov in Linux kernel

有意思的是libgcov在Linux kernel中也有實現：https://github.com/torvalds/linux/blob/master/kernel/gcov/gcc_4_7.c
“關機輸出.gcda文件”顯然不是好的用戶體驗～`.gcno/.gcda`實際上是用debugfs提供的。
`kernel/gcov/clang.c`是Android提供的用於Clang的runtime實現。



gcov used _J. C. Tiernan, An Efficient Search Algorithm to Find the Elementary Circuits of a Graph, Comm ACM 1970_. The worst-case time bound is exponential in the number of elementary circuits. It enumerated cycles (aka simple circuit, aka elementary circuit) and performed cycle cancelling.
In 2016, the resolution to [GCC PR67992](https://gcc.gnu.org/PR67992) switched to Donald B. Johnson's algorithm to improve performance. The theoretical time complexity is $O((V+E)(c+1))$ where $c$ is the number of cycles, which is exponential in the size of the graph.
(Boost attributed the algorithm to K. A. Hawick and H. A. James, and gcov inherited this name. However, that paper did not improve Johnson's algorithm.)

Actually every step of cycle cancelling decreases the count of at lease one arc to 0, so there are at most $O(E)$ cycles.
The resolution to [PR90380 (target: GCC 8.3)](http://gcc.gnu.org/PR90380) skipped non-positive arcs and decreased the time complexity to $O(V*E^2)$ (in theory it could be $O(E^2)$ but the implementation has a linear scan).

llvm-cov gcov也有GCC PR90380修復的指數複雜度問題，該問題被<https://reviews.llvm.org/D93036>修復。
實際上，找$O(E)$個圈使用cycle enumeration的Donald B. Johnson算法意義不大，可以換成普通的cycle detection。
<https://reviews.llvm.org/D93073>實現了簡化，將複雜度降低到$O(E^2)$。

Thinking more, we are processing a reducible flow graph (there is no intuitive cycle count for an irreducible flow graph).
Every natural loop is identified by a back edge. By constructing a dominator tree, finding back edges, identifying natural loops and clearing the arc counters (we will compute incoming counts so we clear counters to prevent duplicates), the time complexity can be decreased to $O(depthOfNestedLoops*E)$.
In practice, the semi-NCA algorithm (time complexity: $O(V^2)$, but considered faster than the almost linear Lengauer-Tarjan's algorithm) is not difficult to implement, but identifying natural loops is troublesome. So the method is not useful.
