---
layout: post
title: BCTF 2016 hsab及BetaFour命題報告
author: MaskRay
tags: [ctf, bctf, docker, binary]
---

上週四、五給[BCTF 2016](http://bctf.cn)出了兩道題。

<!-- more -->

## hsab

Misc類型。向解出此題的48支隊伍表達祝賀。

選手連接服務器的指定TCP端口後，服務器會要求proof of work：給出challenge，求出使得sha256(challenge+x)前若干bits爲0的x。之後即進入到bash交互界面。不難發現裏面沒有shell utilities，常用命令都沒有。下面用0指代服務器IP。
```
╰─% nc 0 2222
-bash-4.4# ls
ls
-bash: ls: command not found
-bash-4.4# cat
cat
-bash: cat: command not found
-bash-4.4#
```

其中輸入命令在輸出中重複了。如果hexdump流量，可以發現換行符使用`\r\n`。
```
% nc 0 2222
-bash-4.4$ echo $-
echo $-
himBHs
-bash-4.4$
```

### Pseudoterminal

Interactive模式下`$-`包含`i`。

以上跡象都說明服務端很可能使用了pseudoterminal。考慮用`socat stdio,raw,echo=0 tcp:0:2222`代替`nc 0 2222`連接服務端。使用文件名補全可以發現flag文件`/home/ctf/flag.ray`：
```
% socat stdio,raw,echo=0 tcp:0:2222
-bash-4.4$ /home/ctf/
-bash: /home/ctf/: Is a directory
-bash-4.4$ a /home/ctf/flag.ray
```

### 列示目錄

另外也可以用文件名通配符找出flag文件：
```
% nc 0 2222
-bash-4.4$ echo /*
echo /*
/bin /dev /etc /home /lib /lib64 /proc /run /sys /tmp /usr /var
-bash-4.4$ echo /home/*
echo /home/*
/home/ctf
-bash-4.4$ echo /home/ctf/*
echo /home/ctf/*
/home/ctf/flag.ray
-bash-4.4$
```

沒有`cat`等shell utilities，可以考慮用bash內置的讀文件命令，如`read /home/ctf.ray`、`mapfile a < /home/ctf.ray`命令，另外可以`set -v`後`source /home/ctf/flag.ray`。但這些都不奏效，並且可以發現`<`重定向不可用。

之後有至少三個方法讀取flag：

- `bash -v /home/ctf/flag.ray`
- `history -r /home/ctf/flag.ray; history`

### ctypes.sh

另一個比較複雜。補全還能顯示內置命令，其中多了`dlopen`、`dlcall`等非bash自帶的命令。另外通過補全查看`/usr/lib/`下的文件：
```
libcrypto.so.1.0.0*         proof of work使用，sha256
ld-musl-x86_64.so.1*        libc
libseccomp.so.2.2.1*        限制系統調用
libz.so.1@                  libcrypto.so依賴
ld-linux-x86-64.so.2@       libc
libdl.so.2@                 libc
libseccomp.so.2@            限制系統調用
libffi.so.6.0.4*            不知用途
libc.so.6@                  libc
libffi.so.6@                不明用途
libncursesw.so.6*           bash依賴
libz.so.1.2.8*              libcrypto.so依賴
libncursesw.so@             bash依賴
```

`libffi.so`很可疑，也從另一方面佐證了這個bash做過手腳。`dlopen`和`dlcall`等來自fqj1994提到的[ctypes.sh](https://github.com/taviso/ctypes.sh)，這個項目給bash提供了foreign function interface，可以加載動態鏈接庫，執行其中的函數。因此讀取flag的方式很自然：
```
dlcall -n fd -r int open /home/ctf/flag.ray 0
dlcall -n buf -r pointer malloc 100
dlcall -n nread -r int read $fd $buf 100
dlcall write 1 $buf $nread
```

另外，`help`和`set`命令也可以列出`dlopen`和`dlopen`命令。
```
-bash-4.4$ set
......
builtins=([0]="callback" [1]="dlcall" [2]="dlclose" [3]="dlopen" [4]="dlsym" [5]="pack" [6]="unpack")
-bash-4.4$ help
......
 dlcall [-n name] [-a abi] [-r type] [>  source filename [arguments]
 dlclose handle [handle ...]             suspend [-f]
 dlopen [-N|-l] [-t] [-d] [-g] [-n] li>  test [expr]
 dlsym [-n name] [-h handle] symbol      time [-p] pipeline
......
```

不用終端提供的補全功能也可以列目錄：
```bash
dlopen libc.so
dlcall -n dir -r pointer opendir /home/ctf
dlcall -n dirent -r pointer readdir $dir # 跳過第一個
dlcall -n dirent -r pointer readdir $dir # 跳過第二個
dlcall -n dirent -r pointer readdir $dir # 第三個
data=(long long ushort uchar char char char char char char char char char char char char)
unpack $dirent data
echo ${data[@]}
```

PPP(Plaid Parliament of Pwning)用了ctypes.sh提供的callback，使用`ftw`遍歷目錄：

```bash
function ftw_callback {
    # int (*fn) (const char *fpath, const struct stat *sb, int typeflag)
    dlcall printf "%s\n" $2
    result=(int:0)
    pack $1 result
    return
}

callback -n ftw_callback ftw_callback int pointer pointer int
dlcall ftw "/home" $ftw_callback 128
```

出題時沒有想到`history -r /home/ctf/flag.ray; history`和`bash -v /home/ctf/flag.ray`也可以讀文件……一方面是太倉促了。治牙搬家搞了很久，一邊kelwin一直在催題，週四最後就沒剩下多少時間，倉促地patch了bash源碼`builtins/`目錄下`mapfile.def`、`source.def`、`read.def`等，本來也想過是不是應該統一地把libc.so中`open`整個`LD_PRELOAD`了，似乎有些麻煩就沒做。好吧，也不願意在出題上浪費太多時間。

![ctypes.sh解法](/static/2016-03-21-bctf-2016-hsab-and-betafour/hsab.jpg)

## BetaFour

PPC(Professional Programming and Coding)類型。向解出此題的19支隊伍表達祝賀。

選手連接服務器的指定TCP端口後，服務器會要求proof of work，之後可以發現是10局12x12的四子棋，每局勝2分平1分負0分，積14分或以上即可獲得flag。服務端每一步思考0.3秒(CPU time)，選手每一步要在4秒內給出。

AI代碼來自2013年春季學期清華大學計算機科學與技術系人工智能課的大作業。開賽兩天前kelwin說和BrieflyX商量了下可以出一題，正好與幾天前的AlphaGo與李世乭5盤圍棋的時事關聯。我說可以嘗試改一下之前大作業的代碼。當時看了<https://chessprogramming.wikispaces.com/>一些資料，苦於實現似乎並不容易……當時詢問了幾個學長，一致推薦Monte Carlo tree search，後來我們這屆計算機系同學似乎好多都用了這個方法。課程作業沒有提供Online Judge，如何保證交作業時代碼能在助教的Windows Visual Studio上編譯通過很痛苦，一不小心就有0分危險。本想把bitboard改成AVX2的256位integer，苦於服務器是Ivy Bridge不支持。似乎扯遠了。

服務端每一步都用`clock_gettime(CLOCK_THREAD_CPUTIME_ID)`計時佔足0.3秒CPU time，假設選手1秒(其中含網絡延時，延時越大CPU負擔越小)下一步，那麼`(1+0.3)/0.3~=4.33`個連接就能達到單核心100%佔用。當時感覺CPU資源消耗太大，因此後來又加了proof of work，但依然憂心忡忡。
![某段時間Container Advisor監控信息，CPU資源消耗確實很大](/static/2016-03-21-bctf-2016-hsab-and-betafour/betafour.jpg)

我們的服務在Google Compute Engine上，它的網絡有些問題，剛`close`的socket之前若緊接`write`錯誤消息，此時若對端也有併發發送來的消息，`close`很可能不會轉化爲正常情況下的TCP FIN，而會是TCP RST，並且`write`的錯誤消息客戶端也無法收到。而連接localhost則正常。
```
23:41:42.564523 IP 192.168.1.3.45206 > 104.199.132.199.2223: Flags [S], seq 3738526007, win 29200, options [mss 1460,sackOK,TS val 16225689 ecr 0,nop,wscale 7], length 0
23:41:42.836734 IP 104.199.132.199.2223 > 192.168.1.3.45206: Flags [S.], seq 1081116448, ack 3738526008, win 28960, options [mss 1400,sackOK,TS val 30229188 ecr 16225689,nop,wscale 7], length 0
23:41:42.836807 IP 192.168.1.3.45206 > 104.199.132.199.2223: Flags [.], ack 1, win 229, options [nop,nop,TS val 16225770 ecr 30229188], length 0
23:41:42.836860 IP 192.168.1.3.45206 > 104.199.132.199.2223: Flags [P.], seq 1:3, ack 1, win 229, options [nop,nop,TS val 16225771 ecr 30229188], length 2
23:41:43.446826 IP 192.168.1.3.45206 > 104.199.132.199.2223: Flags [P.], seq 1:3, ack 1, win 229, options [nop,nop,TS val 16225954 ecr 30229188], length 2
23:41:43.655346 IP 104.199.132.199.2223 > 192.168.1.3.45206: Flags [.], ack 3, win 227, options [nop,nop,TS val 30229409 ecr 16225954], length 0
23:41:43.655377 IP 104.199.132.199.2223 > 192.168.1.3.45206: Flags [P.], seq 1:195, ack 3, win 227, options [nop,nop,TS val 30229409 ecr 16225954], length 194
23:41:43.655398 IP 192.168.1.3.45206 > 104.199.132.199.2223: Flags [.], ack 195, win 237, options [nop,nop,TS val 16226016 ecr 30229409], length 0
23:41:43.655479 IP 104.199.132.199.2223 > 192.168.1.3.45206: Flags [R.], seq 195, ack 3, win 227, options [nop,nop,TS val 30229409 ecr 16225954], length 0
23:41:43.860179 IP 104.199.132.199.2223 > 192.168.1.3.45206: Flags [R], seq 1081116643, win 0, length 0
```

解決BetaFour的一種方法是實現一個更強的AI。另一種思路則是左右互搏。

AlphaGo的訓練用到了self-play給予本題啓示。BetaFour的10局棋，選手交替執先，因此也可以考慮左右互搏，創建兩個連接A和B。連接B先隨便下一局(輸)，之後服務端執先，讀取着法後發送給連接A；連接A服務端走子之後把着法發給連接B，如此交替。若一邊輸則另一邊贏。下完模仿的9局後，A再隨便下一局(輸)。除去A和B隨便下的，兩個連接的得分和爲18，有一定機率會大於等於14，從而得到flag。

代碼參見<https://github.com/MaskRay/BCTF-2016-hsab-and-BetaFour/blob/master/BetaFour/self-play.py>。

## 服務端AI簡述

Monte Carlo Tree Search把博弈樹搜索和隨機算法結合了起來，domain-specific knowledge是可選的，這意味着無需像minimax那樣實現一個對未結束局面估價的函數。

每一回合程序建立一個根節點表示當前局面。每個節點連向孩子的邊代表處於該節點表示的局面下可以選擇的着法，而相應的孩子就是選擇該着法後轉移到的局面。樹一開始只包含根節點，在之後的expansion過程中節點會被添加到樹中。

然後執行若干次迭代過程，每次迭代都會依次執行selection、expansion、simulation和backpropagation四個過程。

- Selection。從根出發，根據tree policy走到一個孩子節點，重複此過程直到碰到一個可擴展的節點爲止。
  可擴展的節點指的是帶有未訪問孩子的節點。
- Expansion。此時我們已經走到了一個可擴展節點。根據tree policy選擇未探索過的着法中最有探索價值的，給當前節點添加一個孩子並走到這個孩子節點。
- Simulation。根據default policy在當前節點表示棋盤下模擬雙方對弈。
- Backpropagation。根據對弈的結果更新當前節點及其所有祖先的統計信息。

MCTS是這一類算法的統稱，通過選取適當的tree policy和default policy就得到了一系列具體算法。對於tree policy，我使用了Upper Confidence Bounds for Trees(UCT)算法，根據公式：

$$
UCT_j = X_j + 2C\sqrt{\frac{2\ln{n}}{n_j}}
$$

來選擇具有最大$UCT$值的孩子節點$j$。其中$X_j\in [0, 1]$，是根據統計信息計算出來的表示對應着法是否優秀的量。$n$爲當前節點的訪問數，$n_j$爲孩子節點$j$的訪問數，$C$爲一個大於0的常數。

公式中第一個加數表示孩子節點的優秀程度(exploitation)，第二個加數表示探索度(exploration)。

- Exploitation高代表選擇該孩子節點對應的着法，模擬對弈的結果中有利局面多，所以該孩子節點對應的着法較爲優秀。
- Exploration高代表選擇該孩子節點對應的着法的嘗試次數不夠多(因爲$n_j$在分母中)，需要進一步探索。

移植時刪除了大作業中一些優化。

### 位棋盤

需要處理的最大棋盤規模爲$12\times 12$，可以用一個`uint64_t`表示四列，那麼12列就只需要3個`uint64\_t`來表示，可以看作一個`uint192_t`，其各個二進制位對應的棋盤上位置如下：

```
 64-bit         64-bit        64-bit

0f 1f 2f 3f | 4f 5f 6f 7f | 8f 9f af bf
.. .. .. .. | .. .. .. .. | 8. 9. .. ..
01 11 21 31 | 41 51 61 71 | 81 91 a1 b1
00 10 20 30 | 40 50 60 70 | 80 90 a0 b0
```

對於勝負的判定，只需要用移位操作和與運算即可在在$O(1)$時間內求出。比如要判斷白棋在豎直方向上是否連成四子，只需要取出其位棋盤$w$，判斷\texttt{w \& w>>1 \& w>>2 \& w>>3}，類似地可以判斷水平方向、斜線方向上是否連成四子。

另外每一列的高度也可用位運算快速求出，以加速生成候選着法。對於每一列，計算二進製表示裏從下到上連續的1的個數，這個通過加1後計算前導0可以算出。

### 省略父節點指針

父節點指針僅用於backpropagation過程，而在selection階段下溯時若記錄沿路節點，則可得到backpropation需要的祖先節點信息。

## 題目服務準備

### Linux container的文件系統鏡像

我下載Alpine Linux的二進制包<http://dl-3.alpinelinux.org/alpine/edge/main/x86_64/>，只安裝必要依賴，以爲了創建儘可能小的文件系統鏡像。
```
% tree rootfs
.
├── bin -> usr/bin
├── dev
│   ├── ptmx
│   └── pts
├── etc
│   ├── group
│   ├── passwd
│   ├── profile
│   └── shadow
├── home
│   └── ctf
│       └── flag.ray
├── lib -> usr/lib
├── lib64 -> usr/lib
└── usr
    ├── bin
    │   ├── bash
    │   └── server
    ├── lib
    │   ├── ld-linux-x86-64.so.2 -> ld-musl-x86_64.so.1
    │   ├── ld-musl-x86_64.so.1
    │   ├── libcrypto.so.1.0.0
    │   ├── libc.so.6 -> ld-musl-x86_64.so.1
    │   ├── libdl.so.2 -> ld-musl-x86_64.so.1
    │   ├── libffi.so.6 -> libffi.so.6.0.4
    │   ├── libffi.so.6.0.4
    │   ├── libncursesw.so -> libncursesw.so.6
    │   ├── libncursesw.so.6
    │   ├── libseccomp.so.2 -> libseccomp.so.2.2.1
    │   ├── libseccomp.so.2.2.1
    │   ├── libz.so.1 -> libz.so.1.2.8
    │   └── libz.so.1.2.8
    ├── lib64 -> lib
    └── local
        └── lib
            └── ctypes.so
% cd rootfs
% sudo tar c . | sudo docker import hsab.tgz hsab
......
% sudo docker run -p 2222:2222 -u ctf -d -v /dev:/dev -v /dev/pts:/dev/pts hsab /usr/bin/server --pow-bits 20
```

### 防護

編譯鏈接時生成position-independent executable。

malloc相關的一些環境變量如`MALLOC_CHECK_`等。

Resource limits如`RLIMIT_NPROC`設爲0可以阻止該用戶創建新進程；`RLIMIT_CPU`限制CPU time；`RLIMIT_FSIZE`限制創建文件的最大長度；`RLIMIT_DATA`限制進程data segment大小。

seccomp限制系統調用，可以考慮使用白名單，阻止一些可能的惡意行爲，比如創建進程、寫文件、kill/stop服務進程。

`prctl PR_SET_SECUREBITS PR_SET_DUMPABLE PR_SET_NO_NEWPRIVS`比較複雜。

unshare隔離各類namespace。

對於小型rootfs，感覺最有必要的只有resource limits和seccomp。

## IRC

`#bctf@freenode`，把自己配置成了一個autoop和proxy bot，轉發選手的問題……[znc](znc.in)的autoop模塊可以用於自動給加入channel的nick添加op。

## 代碼

<https://github.com/MaskRay/BCTF-2016-hsab-and-BetaFour>

## 感想

希望用上Proof of Work as a Service和Captcha as a Service。

如果有更多時間的話，想給BetaFour加上錄像，first blood時可以播放下視頻，提升展示效果。其他很多服務因爲私密性不方便這麼做，但Connect Four這麼搞還是不錯的呢。

比賽承辦方人工成本太高了，而收益爲零。
