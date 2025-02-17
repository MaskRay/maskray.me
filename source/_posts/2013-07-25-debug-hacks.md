---
layout: post
title: 《Debug Hacks》和調試技巧
author: MaskRay
tags: [debug, c, review]
---

下篇見[調試技巧2](/blog/2015-03-14-debug-hacks-2)。

## Debug Hacks

作者爲吉岡弘隆、大和一洋、大巖尚宏、安部東洋、吉田俊輔，有中文版《Debug Hacks中文版—深入調試的技術和工具》。這本書涉及了很多調試技巧，對調試器使用、內核調試方法、常見錯誤的原因，還介紹了`systemtap`、`strace`、`ltrace`等一大堆工具，非常值得一讀。

<!-- more -->

話說我聽說過的各程序設計課程似乎都沒有強調過調試的重要性，把調試當作單獨一節課來上(就算有估計也上不好)，很多人都只會`printf`調試法，breakpoint都很少用，就不提conditional breakpoint、watchpoint、reverse execution之類的了。也看到過很多同學在調試上浪費了很長很長的時間。

下面是篇review，也包含了一些我自己整理的一些調試技巧。

## 折騰工具

繼續牢騷幾句，我接觸過的人當中感覺最執着與折騰工具的人只有兩個，[ppwwyyxx](http://ppwwyyxx.com/)和[xiaq](http://wiki.tuna.tsinghua.edu.cn/xiaq)，他們是少有的能把折騰工具當作正經工作來做的人。

很久以前我還會到處在網上搜索好的實用工具，尤其是那些CLI程序，比如`renameutils`、`xsel`、`recode`、`the_silver_searcher`，查閱文檔定製自己的配置文件。但這麼做花費的時間太多。後來就想我可以搜索一些善於折騰的人的配置文件，關注他們修改了哪些地方，我的配置只要取衆家之所長就可以了。

先厚顏自薦一下[我的配置](https://github.com/MaskRay/Config)。下面的用戶列表就是我找到的在GitHub上把`dotfiles`配置地井井有條的人(如果GitHub支持按照項目的大小排序，列表蒐集就能省很多麻煩了)：

```
alejandrogomez bhj craigbarnes dotvim hamaco joedicastro laurentb ok100 pyx roylez sjl trapd00r vodik w0ng
```

有了上述的`dotfiles`，其他人的`dotfiles`大多都不願看了。但是五嶽歸來不看山，黃山歸來不看嶽，`ppwwyyxx`的[dotfiles](https://github.com/ppwwyyxx/dotfiles)感覺與之前諸位相比更勝一籌。

無關的話到此結束，下面是正文：

## gdb

### 記錄歷史

把下面幾行添加到`~/.gdbinit`中吧，`gdb`啓動時會自動讀取裏面的命令並執行：

```
set history save on
set history size 10000
set history filename ~/.history/gdb
```

我習慣在`~/.history`堆放各個歷史文件。有了歷史，使用`readline`的`reverse-search-history (C-r)`就能輕鬆喚起之前輸入過的命令。

### 修改任意內存地址的值

```
set {int}0x83040 = 4
 ```

### 顯示intel風格的彙編指令

```
set disassembly-flavor intel
```

### 斷點在function prologue前

先說一下function prologue吧，每個函數最前面一般有三四行指令用來保存舊的幀指針(rbp)，並騰出一部分棧空間(通常用於儲存局部變量、爲當前函數調用其他函數騰出空間存放參數，有時候還會存儲字面字符串，當有nested function時也會用於保存當前的棧指針)。

在x86-64環境下典型的funcition prologue長成這樣：

```asm
push rbp
mov rbp, rsp
sub rsp, 0x10
```

可能還會有`and`指令用於對齊`rsp`。如果編譯時加上`-fomit-frame-pointer`(Visual Studio中文版似乎譯作“省略框架指針”)，那麼生成的指令就會避免使用`rbp`，function prologue就會簡化成下面一行：

```asm
sub rsp, 0x10
```

設置斷點時如果使用了`b *func`的格式，也就是說在函數名前加上`*`，`gdb`就會在執行function prologue前停下，而`b func`則是在執行function prologue後停下。參考下面的會話：

```text
% gdb a.out
Reading symbols from /tmp/a.out...done.
(gdb) b *main
Breakpoint 1 at 0x4005cc: file a.c, line 4.
(gdb) r
Starting program: /tmp/a.out
warning: Could not load shared library symbols for linux-vdso.so.1.
Do you need "set solib-search-path" or "set sysroot"?

Breakpoint 1, main () at a.c:4
4       {
(gdb) disas
Dump of assembler code for function main:
=> 0x00000000004005cc <+0>:     push   rbp
   0x00000000004005cd <+1>:     mov    rbp,rsp
   0x00000000004005d0 <+4>:     sub    rsp,0x10
   0x00000000004005d4 <+8>:     mov    DWORD PTR [rbp-0x4],0x0
   0x00000000004005db <+15>:    mov    eax,DWORD PTR [rbp-0x4]
   0x00000000004005de <+18>:    mov    esi,eax
   0x00000000004005e0 <+20>:    mov    edi,0x4006ec
   0x00000000004005e5 <+25>:    mov    eax,0x0
   0x00000000004005ea <+30>:    call   0x400454 <printf@plt>
   0x00000000004005ef <+35>:    leave
   0x00000000004005f0 <+36>:    ret
End of assembler dump.
(gdb)
```

### Checkpoint

`gdb`可以爲被調試的程序創建一個快照，即保存程序運行時的狀態，等待以後恢復。這個是非常方便的一個功能，特別適合需要探測接下來會發生什麼但又不想離開當前狀態時使用。

`ch`是創建快照，`d c ID`是刪除指定編號的快照，`i ch`是查看所有快照，`restart ID`是切換到指定編號的快照，詳細說明可以在shell裏鍵入`info '(gdb) Checkpoint/Restart'`查看。

```text
% gdb ./a.out
Reading symbols from /tmp/a.out...done.
(gdb) b 6
Breakpoint 1 at 0x4005db: file a.c, line 6.
(gdb) r
Starting program: /tmp/a.out
warning: Could not load shared library symbols for linux-vdso.so.1.
Do you need "set solib-search-path" or "set sysroot"?

Breakpoint 1, main () at a.c:6
6         printf("%d\n", a);
(gdb) ch
checkpoint: fork returned pid 6420.
(gdb) p a=3
$1 = 3
(gdb) i ch
  1 process 6420 at 0x4005db, file a.c, line 6
* 0 process 6416 (main process) at 0x4005db, file a.c, line 6
(gdb) restart 1
Switching to process 6420
#0  main () at a.c:6
6         printf("%d\n", a);
(gdb) c
Continuing.
0
[Inferior 1 (process 6420) exited with code 02]
[Switching to process 6416]
(gdb)
```

上面的會話中先用`ch`創建了一個快照，緊接着`a`被修改爲了3，隨後用`restart 1`恢復到編號爲1的快照，繼續運行程序可以發現`a`仍然爲原來的值0。

以色列的Haifa Linux club有一次講座講`gdb`，講稿值得一看：[http://haifux.org/lectures/210/gdb_-_customize_it.html]()

### 逆向技術

Long Le的[peda](https://github.com/longld/peda)很不錯，感覺比[http://reverse.put.as]()的[https://github.com/gdbinit/Gdbinit]()好用。

## gcc

### Mudflap

使用了compile-time instrumentation(CTI)的工具。編譯時加上`-fmudflap -lmudflap`選項即可，會在很多不安全代碼生成的指令前加上判斷合法性的指令。

```
% echo 'int main() { int z[1]; z[1] = 2; }' | cc -xc - -fmudflap -lmudflap
% ./a.out
*******
mudflap violation 1 (check/write): time=1376473424.792953 ptr=0x7fff2cde3150 size=8
pc=0x7fa2bacf86f1 location=`<stdin>:1:29 (main)'
      /usr/lib/gcc/x86_64-pc-linux-gnu/4.7.3/libmudflap.so.0(__mf_check+0x41) [0x7fa2bacf86f1]
      ./a.out(main+0x8f) [0x400b6b]
      /lib64/libc.so.6(__libc_start_main+0xf5) [0x7fa2ba968c35]
Nearby object 1: checked region begins 0B into and ends 4B after
mudflap object 0x7070e0: name=`<stdin>:1:18 (main) z'
bounds=[0x7fff2cde3150,0x7fff2cde3153] size=4 area=stack check=0r/3w liveness=3
alloc time=1376473424.792946 pc=0x7fa2bacf7de1
number of nearby objects: 1
```

第一行用`-xc -`讓`cc`從標準輸入讀源代碼，並當作C來編譯。接來下執行`./a.out`，可以看到運行時程序報錯了。

使用`MUDFLAP_OPTIONS`環境變量可以控制Mudflap的運行期行爲，具體參見[Mudflap Pointer Debugging](http://gcc.gnu.org/wiki/Mudflap_Pointer_Debugging)。

### AddressSanitizer

和Mudflap類似的工具，`clang`和`gcc`可以加上選項`-fsanitize=address`使用，比如：

```bash
clang -fsanitize=address a.c
```

如果想在出錯的地方斷點停下來，可以用`gdb`打開，輸入`b __asan_report_store1`回車，再輸入`r`回車運行程序。

### `-ftrapv`

這個選項是調試有符號整型溢出問題的利器。在i386環境下，gcc會把`int32_t`運算編譯成`call __addvsi3`，`__addvsi3`函數會在運行時檢查32位有符號加法運算是否產生溢出，如果是則調用`abort`函數中止程序。減法、乘法和取反運算也有類似的運行時函數檢查溢出，另外也有64位版本的`__addvdi3`等函數。但不存在對無符號整型的溢出檢測函數。比如下面這些代碼均會觸發trap：

```c
int a = INT_MAX; a++;
int b = INT_MIN; b--;
int c = INT_MAX; c *= 2;
int d = INT_MIN; d = -d;
```

這段代碼來自`gcc`項目目錄的`libgcc/libgcc2.c`：

```c
#ifdef L_subvsi3
Wtype
__subvSI3 (Wtype a, Wtype b)
{
  const Wtype w = (UWtype) a - (UWtype) b;

  if (b >= 0 ? w > a : w < a)
    abort ();

  return w;
}
```

但注意在x86-64環境下`-ftrapv`只檢查64位溢出。考慮下面這段代碼：

```c
#include <limits.h>
#include <stdio.h>

int main()
{
  int a = INT_MAX;
  a++;

  puts("barrier");

  long b = LONG_MAX;
  b++;
}
```

在x86-64下用`gcc`編譯運行，輸出`barrier`後才會執行`abort`使程序中止，因爲`int32_t`的溢出不會觸發trap。

`clang`也有`-ftrapv`，在x86-64環境下對於`int32_t`的溢出也能觸發trap。

### `_FORTIFY_SOURCE`

`gets`、`strcpy`這類函數容易造成stack mashing。`gcc`編譯時如果指定了`-D_FORTIFY_SOURCE=1`，生成的彙編程序中這些不安全的函數調用會被替代爲`libc.so`中名字類似`__gets_chk`的一類安全函數，會在運行期檢查是否產生了緩衝區溢出。比如，下面的代碼會在運行時報錯：

```c
#include <string.h>

int main()
{
  char a[2];
  strcpy(a, "meow");
}
```

Gentoo Portage從`gcc-4.3.3-r1`開始默認開啓`_FORTIFY_SOURCE`標誌了，好多發行版都開啓了，測試發現Arch Linux的`gcc`似乎沒有。shell裏執行下面代碼就可以看到Gentoo裏是怎麼定義`_FORTIFY_SOURCE`的了：

```bash
echo -e '#undef __OPTIMIZE__\nmain() { printf("%d\\n", _FORTIFY_SOURCE); }' | cpp
```

也就是當優化等級在`-O1`或以上時`_FORTIFY_SOURCE`會生效，名字爲`__$func_chk`模式的函數會被使用。這種做法造成了一些麻煩，比如`suricata` git tree裏的`src/suricata.c`使用了`#ifdef _FORTIFY_SOURCE`，會造成編譯無法通過。

### `-fstack-protector`

-fstack-protector -fstack-protector-all
gcc 4.8.1 -fstack-protector-strong

https://securityblog.redhat.com/2013/10/23/debugging-stack-protector-failures/

開啓Stack-Smashing Protector (SSP)。我的理解是在儲存的幀指針(rbp)前寫入一個magic number，函數返回的時候檢查下這個magic number是否被改動，如果是就可能產生stack smashing了。這個方法的footprint最小，但是保護力度也比較弱。

#### IA32

```
function prologue
 80484c0:       65 a1 14 00 00 00       mov    eax,gs:0x14
 80484c6:       89 45 f4                mov    DWORD PTR [ebp-0xc],eax

function epilogue
 80484d7:       8b 45 f4                mov    eax,DWORD PTR [ebp-0xc]
 80484da:       65 33 05 14 00 00 00    xor    eax,DWORD PTR gs:0x14
 80484e1:       74 05                   je     80484e8 <foo+0x33>
 80484e3:       e8 68 fe ff ff          call   8048350 <__stack_chk_fail@plt>
 80484e8:       c9                      leave
 80484e9:       c3                      ret
```

#### x86-64

```
function prologue:

  4005c9:       64 48 8b 04 25 28 00    mov    rax,QWORD PTR fs:0x28
  4005d0:       00 00
  4005d2:       48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax

function epilogue

  400618:       64 48 33 04 25 28 00    xor    rax,QWORD PTR fs:0x28
  40061f:       00 00
  400621:       74 05                   je     400628 <foo+0x3b>
  400623:       e8 88 fe ff ff          call   4004b0 <__stack_chk_fail@plt>
  400628:       48 83 c4 78             add    rsp,0x78
  40062c:       c3                      ret
```

### execinfo.h

提供了`int backtrace (void **buffer, int size)`、`char ** backtrace_symbols (void *const *buffer, int size)`在程序運行時查看函數調用棧。參見[http://www.gnu.org/software/libc/manual/html_node/Backtraces.html]()。

## Misc

### Valgrind

一系列調試和profiling工具的套件，其中的Memcheck是一個使用了dynamic binary instrumentation(DBI)的工具， 在程序指令間插入自己的指令檢查validity和addressablity。另外Memcheck替換了標準的`malloc`，這樣就可以檢測出off-by-one error、double free、內存泄漏等許多問題。

Memcheck引入的footprint極小，無需重編譯程序，也沒有繁瑣的配置。比如原來是用`./a.out`執行程序，需要Memcheck時就換成`valgrind ./a.out`。

在程序訪問某一內存地址時Memcheck會檢查是否有越界之類的錯誤，Memcheck能診斷出大量但不是全部的訪問錯誤，比如下面這樣有問題的代碼就沒法檢查出來：

```c
int main()
{
  int a[1];
  a[1992] = 12;
}
```

因爲`a[1992]`的地址在棧上，允許訪問。

Valgrind啓動時會讀取`~/.valgrindrc`，對於`memcheck`我配置了下面這幾行：

```text
--memcheck:leak-check=yes
--memcheck:show-possibly-lost=yes
--memcheck:show-reachable=yes
--memcheck:track-origins=yes
--memcheck:dsymutil=yes
--memcheck:track-fds=yes
--memcheck:track-origins=yes
--memcheck:gen-suppressions=all
```

`valgrind --vgdb-error=0 --vgdb=yes`很強大，可以在進程遇到錯誤時讓`gdb`調試。

### strace

記錄程序執行的系統調用和收到的信號，和`valgrind`類似，使用非常簡單：

```bash
strace ./a.out
```

有一些選項可以attach到現有進程上去(-p)、記錄時刻(-t)、統計系統調用使用次數(-c)、過濾特定的系統調用(-e)等。

帶上`-c`選項可以統計系統調用的使用次數：
```
% strace -c ls
chap04  chap05  chap06  chap07  chap08  chap09  chap10  chap11  chap12  chap13  chap14  chap15  chap16  chap17
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         5           read
  0.00    0.000000           0         1           write
  0.00    0.000000           0         7           open
  0.00    0.000000           0        10           close
  0.00    0.000000           0         8           fstat
  0.00    0.000000           0        20           mmap
  0.00    0.000000           0        12           mprotect
  0.00    0.000000           0         2           munmap
  0.00    0.000000           0         3           brk
  0.00    0.000000           0         2           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         2           ioctl
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           fcntl
  0.00    0.000000           0         2           getdents
  0.00    0.000000           0         1           getrlimit
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         2         1 futex
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           openat
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                    85         2 total
```

`-e`選項只跟蹤指定系統調用：

```
% strace -e read,open ls
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
open("/lib64/librt.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220(\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libacl.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\320#\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0@M\2\0\0\0\0\0"..., 832) = 832
open("/lib64/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0@}\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libattr.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`\25\0\0\0\0\0\0"..., 832) = 832
open("/usr/lib64/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
chap04  chap05  chap06  chap07  chap08  chap09  chap10  chap11  chap12  chap13  chap14  chap15  chap16  chap17
+++ exited with 0 +++
```

使用`strace`還可以做一些很可怕的事，比如有`root`權限的情況下嗅探`sshd`以得到其他嘗試SSH登錄的用戶的密碼：[SSHD password sniffing](http://blog.0x10.co.uk/2009/12/sshd-password-sniffing.html)。

`-p`很有用，比如調試CGI wrapper`fcgiwrap`，觀察它的輸出：

```
strace -s200 -p$(pidof -s fcgiwrap) -e write
```

### ltrace

記錄程序調用的動態庫中的函數。名字和`strace`很像，使用方式和很多命令行選項也如出一轍。

查看`echo test`

```
% ltrace echo test
__libc_start_main(0x401590, 2, 0x7fff2bb3d4d8, 0x403ef0 <unfinished ...>
getenv("POSIXLY_CORRECT")                                                                        = nil
strrchr("echo", '/')                                                                             = nil
setlocale(LC_ALL, "")                                                                            = "en_US.UTF-8"
bindtextdomain("coreutils", "/usr/share/locale")                                                 = "/usr/share/locale"
textdomain("coreutils")                                                                          = "coreutils"
__cxa_atexit(0x401cf8, 0, 0, 0x736c6974756572)                                                   = 0
strcmp("test", "--help")                                                                         = 71
strcmp("test", "--version")                                                                      = 71
fputs_unlocked(0x7fff2bb3f1d3, 0x7f50af982160, 0, 45)                                            = 1
putchar_unlocked(10, 116, 0x7f50afba6004, 0xfbad2a84test
)                                            = 10
exit(0 <unfinished ...>
__fpending(0x7f50af982160, 0, 4, 0x7f50af982cf0)                                                 = 0
ferror_unlocked(0x7f50af982160, 0, 4, 0x7f50af982cf0)                                            = 0
fileno(0x7f50af982160)                                                                           = 1
__freading(0x7f50af982160, 0, 4, 0x7f50af982cf0)                                                 = 0
__freading(0x7f50af982160, 0, 2052, 0x7f50af982cf0)                                              = 0
fflush(0x7f50af982160)                                                                           = 0
fclose(0x7f50af982160)                                                                           = 0
__fpending(0x7f50af982080, 0, 0, 0)                                                              = 0
ferror_unlocked(0x7f50af982080, 0, 0, 0)                                                         = 0
fileno(0x7f50af982080)                                                                           = 2
__freading(0x7f50af982080, 0, 0, 0)                                                              = 0
__freading(0x7f50af982080, 0, 4, 0)                                                              = 0
fflush(0x7f50af982080)                                                                           = 0
fclose(0x7f50af982080)                                                                           = 0
+++ exited (status 0) +++
```

[Ltrace Internals](https://www.kernel.org/doc/ols/2007/ols2007v1-pages-41-52.pdf)描述了`ltrace`的實現機制。

### SystemTap

SystemTap提供了一套底層工具用於trace/probe。用戶編寫SystemTap script語言的程序，SystemTap將其翻譯爲C代碼，再編譯成臨時的內核模塊。內核模塊加載時SystemTap script腳本裏的hook就會在特定event發生時執行。當SystemTap腳本停止運行時，相應的hook就被刪除，移除臨時的內核模塊。這一整套流程都是通過一個簡單的CLI程序`stap`驅動的。

SystemTap使用前的配置過程比較複雜，需要特製的內核，開啓`CONFIG_KPROBES=y`、`CONFIG_DEBUG_INFO=y`等諸多內核編譯選項。

比如如下的簡單腳本就能顯示各進程調用`net/socket.c`內函數的情況：

```text
probe kernel.function("*@net/socket.c").call {
  printf ("%s -> %s\n", thread_indent(1), ppfunc())
}
probe kernel.function("*@net/socket.c").return {
  printf ("%s <- %s\n", thread_indent(-1), ppfunc())
}
```

### perf

```bash
perf record -e probe_a:main -e probe_a:main_1 /home/ray/tmp/a
perf annotate
sudo perf probe -x ~/tmp/a 'main%return %ip %sp'
sudo perf record -e probe_a:main -e probe_a:main_1 /home/ray/tmp/a && sudo perf script
```

可執行文件不能在tmpfs分區。

```bash
A=~/tmp; cc -xc <(echo 'main(){}') -Wl,-rpath,$A -o a && sudo perf probe -d '*' || :; sudo perf probe -x $A/libc.so.6 malloc && sudo perf record -e probe_libc:malloc -aR ./a && sudo perf report -n
```

### 其他

書裏還介紹了很多神奇的玩意兒，比如`kaho`，用於讀取被編譯器優化掉的變量；`livepatch`，運行時動態修改變量、替換函數等。這兩個工具我在網上檢索了下，感覺是個proof of concept的東西，也沒有更新了。不夠這些思路很奇特，想到了並試圖去解決調試時常受困擾的問題，很棒。
