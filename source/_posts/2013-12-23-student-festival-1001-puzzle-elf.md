---
layout: post
title: “1001夜”學生節線上解謎活動1001/key題解
author: MaskRay
tags: [student festival puzzle, reverse engineering]
---

[http://www.net9.org/StudentFestival/1001/]()是模仿東京大學設計的海報謎題：
[<img src='http://www.net9.org/StudentFestival/1001/OnlinePoster.jpg' style='display:block;margin:auto;max-width: 50%;'/>](http://www.net9.org/StudentFestival/1001/)

前幾關的解法可以參考<http://ppwwyyxx.com/2013/Student-Festival-Puzzle/>，下面以出題人角度給出其中的`1001/key`的題解。

[1001/key下載](/static/2013-12-23-student-festival-1001-puzzle-elf/key)

本題是一道crackme，直接運行程序會提示輸入密碼，如果密碼輸入錯誤就會顯示`meow`。

<!-- more -->

用`gdb`打開程序，輸入`run`命令執行會發現程序陷入了死循環。此時使用`C-c`停止程序可以發現當前`eip`之前的幾條指令中有`int 0x80`進行了系統調用，`eax`爲26代表`ptrace`。Linux i386的系統調用約定是當參數不超過6時，各參數分別儲存在`ebx`、`ecx`、`edx`、`esi`、`edi`、`ebp`，結合`ebx`、`ecx`、`edx`等的零值可以知道程序調用了`syscall(SYS_ptrace, PTRACE_ME, 0, 0, 0)`，當返回值小於零時就陷入死循環。一個已經被`PTRACE_ATTACH`調試的程序再次執行上述調用返回值爲-1，因此在`gdb`中直接執行會陷入死循環。繞過這個反調試技巧很容易，把`int 0x80`指令改成兩條`nop`即可，或者使用其他方式比如修改條件跳轉指令的目標到下一條指令等。

這個陷阱我是通過函數的`__attribute__((constructor))`修飾符添加到程序裏的，另外可以使用`objdump -sj.ctors 1001/key`發現引用該函數的指針(以big-endian形式儲存)：

```
% objdump -sj.ctors 1001/key

hh_stripped:     file format elf32-i386

Contents of section .ctors:
 8049f08 ffffffff 40840408 00000000           ....@.......
```

`anti`函數的基址爲`0x08048440`，可以看到出現在`.ctors`段中。`anti`的源碼如下：

```c
void anti() __attribute__((constructor));
void anti()
{
  __asm__ volatile(
                   "movl %0, %%eax;"
                   "xorl %%ebx, %%ebx;"
                   "xorl %%ecx, %%ecx;"
                   "xorl %%edx, %%edx;"
                   "xorl %%esi, %%esi;"
                   "int $0x80;"
                   "test %%eax, %%eax;"
                   "1: jl 1b;"
                   :
                   : "i"(SYS_ptrace)
                   : "eax", "ebx", "ecx", "edx", "esi"
                  );
}
```

如果直接使用`objdump -d 1001/key`查看反彙編結果，會發現輸出`meow`的那個函數(基址爲0x080486c1)的輸出很奇怪。如果在`gdb`裏動態調試就會發現進入這個函數執行完function prologue不久之後反彙編指令(`disassemble`命令)發生了變化。這是因爲這個elf添加了一個簡單的反彙編技巧讓這類反彙編器產生了錯誤的輸出。考察這個函數被調用的地方可以看到`atoi`，再根據反彙編前幾行可以發現這個函數期待一個 <987654 的參數值。後面有若干條件判斷指令檢驗參數是否被接受，之後可以通過查看彙編的方式獲取函數的keygen算法，另外也有一個取巧的辦法。因爲參數 <987654，取值範圍不大，所以我們可以嘗試brute-force枚舉參數傳遞給這個函數考察是否會輸出`meow`以外的串。由於程序使用了`getpass`從終端設備獲取輸入，所以簡單地使用重定向標準輸入的方式或者使用管道不會奏效。查看`glibc`的源碼`isc/getpass.c`可以發現我們只需讓進程失去控制終端即可，因此可以使用`socat`讓進程創建自己的會話從而丟失控制終端：

```
% echo a | socat - exec:1001/key,setsid |& xxd
0000000: 5061 7373 776f 7264 3a20 6d65 6f77 0a    Password: meow.
```

另外也可以用`LD_PRELOAD`替換掉`getpass`或者修改二進制實現替換。

這個隱藏函數的源碼如下：

```
void hidden(int pass)
{
  for (const char *p = "u|c\x92vammv\x8c\x91\x88g7"; *p; p++) {
    putchar(*p ^ (pass % 31));
    pass = pass * 71;
  }
}
```

最後會發現有幾個參數值都能使這個函數輸出`meow`以外的串：`I don't know the secret.`這個地方解決的方式有很多，我很想知道各位選手都採用了什麼方法解決的，有任何思路都歡迎告知我 <i#maskray,me>。

查看`nm -D 1001/key`會發現程序調用了一個庫函數`putchar`，但用`strace 1001/key`執行正常流程卻無法找到這個函數被調用的地方。在`objdump -d 1001/key`裏搜索`putchar`就會發現`0x08048623`處有個隱藏的函數，在正常的程序執行流程中沒有被調用，而這個函數也接受一個int32_t參數。找到`0x080486c1`被調用的地方：

```
 80484b9:       e8 5e ff ff ff          call   804841c <atoi@plt>
 80484be:       89 04 24                mov    DWORD PTR [esp],eax
 80484c1:       e8 9b 01 00 00          call   8048661 <puts@plt+0x235>
```

把`call`指令修改成調用`0x08048623`即可：

```bash
perl -pe 's/\x9b\x01\x00\x00/pack("l<",unpack("l<",$&)-0x08048661+0x08048623)/ge' 1001/key2 > 1001/key3
```

把之前得到的幾個密碼依次傳給這個函數會發現其中一個密碼能使該函數輸出可讀的串：`dajialaiwanai9`，即這個elf所要加密的信息，意義就是“大家來玩ai9”，這裏打一下廣告吧，智能體在線對戰平臺：[http://ai.net9.org]()。

## 後記

策劃同學找到我問有什麼出題思路，我表示毫無想法……後來考慮到計算機繫有個之前有個彙編小學期，大家之前都做過bomblab和buflab，於是就往這方面靠。我對於逆向技術也只是初窺門徑，第一次出題，原來題目只是一段簡單的加密。策劃同學夏雨認爲太簡單了會導致禮物不好分……說要難一些，於是我就在網上翻各種反調試和反彙編技巧，在elf中加入了兩個片段阻撓大家，那個隱藏函數出題同學也覺得不太優美，爲此可能耽誤了大家的好多時間……目前已知的是有兩位大牛浪費了好長時間熬夜做題，導致白天在澳門遊玩時精神不振，感覺過意不去，向各位致歉了！

[ppwwyyxx](http://ppwwyyxx.com)給的題解相當不錯，裏面給出的[lilydjwg](http://lilydjwg.is-programmer.com)把01串轉換成二進制的方法真棒：```python2 -c 'while True: import sys; sys.stdout.write(chr(int(sys.stdin.read(8).strip() or sys.exit(), 2)))'```去年八月有幸一睹lilydjwg行雲流水般的操作，驚異之至，最近又見到兩位如此神速的，一位是京JS 2013上見到的James Halliday (substack)，另一位是這次澳門香港之行見到的Ricky Zhou。

本題實際上是CTF風格的逆向題，CTF是信息安全領域的一類比賽的名稱，考察Web、取證、密碼學、二進制、隱寫等知識。在國內算法類競賽很流行，但信息安全受到的關注就比較少。出題人這裏當然是有一些私心雜唸的……如果認真研究了題目入口網頁[http://www.net9.org/StudentFestival/1001]()，就會發現源代碼裏有一段HTML註釋，是一段廣告，號召大家來參加[http://bctf.cn]()(BCTF“百度杯”全國網絡安全技術對抗賽)。另外也有一支源自清華大學網絡與信息安全實驗室的戰隊blue-lotus ([http://www.blue-lotus.net]())，現在也吸納了包括來自浙江大學、上海交大、青島理工、中國海洋大學、杭州電子科大等高校的多名學生，以及若干綠盟、阿里巴巴等公司的年輕安全技術人員，曾作爲中國的團隊首次闖入全球頂級的DEFCON CTF總決賽。但是作爲酒井一員還是覺得爲什麼貴系人士這麼少……大家有興趣參與進來的話請聯繫blue.lotus.ctf#gmail,com。

學生節的這個解謎活動似乎很受歡迎，我這一週都在澳門/香港，沒有人民幣沒有澳元沒有港元，網絡條件不佳，而且也因此錯過了學生節，遲交了幾個作業……傷心事還是不提了，問了幾位大牛，本來還想再添加一些PPC和Web的題目，實際上也基本出出來了，但考慮到在這道坑人的題後再加那些難度低的沒啥意思……期待這個活動明年辦得更加精彩，那些題還是明年再和大家相見吧……
