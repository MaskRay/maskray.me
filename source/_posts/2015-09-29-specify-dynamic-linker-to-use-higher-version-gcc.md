---
layout: post
title: 指定dynamic linker以使用高版本GCC
author: MaskRay
tags: [binary,elf,ld,gcc]
---

今天有人問怎麼在沒有root的不可更新老舊Linux環境裏使用高版本GCC，主要困難在於GCC對glibc版本有一定要求。假設使用現成的GCC二進制包，解壓到本地後執行`gcc`，報錯：``/lib64/libc.so.6: version `GLIBC_2.14' not found``，即系統glibc的`libc.so.6`中缺乏更高版本的符號(參考`info '(ld) VERSION'`)。使用新符號通常表示API有更新，不過很多時候舊版本的庫也能用，可能有兩個原因：一是編譯機器的庫版本可能較新，soname版本號較高，編譯出來的可執行文件也會有較高的版本要求；二是源文件因保守指定了較高的版本號，實際上舊版本也能用。

比如最近ncurses主版本號更新到6，`/usr/lib/libncursesw.so.5`變成了`/usr/lib/libncursesw.so.6`，導致很多可執行文件沒法用了：

<!-- more -->

```
% ldd /usr/lib/j8/bin/jconsole
        linux-vdso.so.1 (0x00007ffcdb9ec000)
        libedit.so.2 => /usr/lib/libedit.so.2 (0x00007fd995a09000)
        libdl.so.2 => /usr/lib/libdl.so.2 (0x00007fd995805000)
        libncursesw.so.5 => not found
        libc.so.6 => /usr/lib/libc.so.6 (0x00007fd9951f4000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fd995c45000)
```

不想重新編譯的話，可以軟鏈接做一個`libncursesw.so.5`，放在`ld.so`默認會查找的路徑下，或者運行時指定環境變量`LD_LIBRARY_PATH`。另外還有一種修改可執行文件的方法：用十六進制編輯工具把`.dynstr` section中的字串`libncursesw.so.5`改成`libncursesw.so.6`。再次執行時，`ld.so`將會用新的名字查找依賴的庫，就順利找到了`/usr/lib/libncursesw.so.6`，但做這樣的修改得保證新的路徑名長度小於等於原長，否則會覆蓋`.dynstr`中後面的字串。另一種辦法是構造一個新的`.dynstr`放到ELF的其他地方，並變更section header中的`.dynstr`地址信息，非常麻煩。

回到正題，xptree下載了`gcc-c++-4.8.3-9.el7.x86_64.rpm`和`gcc-4.8.3-9.el7.x86_64.rpm`，解壓後執行，報錯：``/lib64/libc.so.6: version `GLIBC_2.14' not found``。解壓`glibc-2.17-78.el7.x86_64.rpm`，報錯：

```
/usr/bin/python: relocation error: MY_LOCAL/lib64/libc.so.6: symbol
_dl_starting_up, version GLIBC_PRIVATE not defined in file ld-linux-x86-64.so.2
with link time reference
g++: relocation error: MY_LOCAL/lib64/libc.so.6: symbol _dl_starting_up,
version GLIBC_PRIVATE not defined in file ld-linux-x86-64.so.2 with link time
reference
```

看來對`ld-linux-x86-64.so.2`(dynamic linker，下面簡稱爲`ld.so`)的版本也有要求。就像修改依賴庫的路徑那樣，修改program header中的`PT_INTERP`：

```
bbe -b '/\x2flib64\x2fld-linux-x86-64.so.2/:27' -e 'L 1' -e 'r 0 /tmp/l/ld.so\0' bin/gcc -o bin/gcc.new
```

效果是把`gcc`中第一次出现的`/lib64/ld-linux-x86-64.so.2`(`PT_INTERP`項)替换成`/tmp/l/ld.so\0`，可以觀察到：

```
% ldd bin/gcc.new
        linux-vdso.so.1 (0x00007ffff85f5000)
        libm.so.6 => /usr/lib/libm.so.6 (0x00007f36788ca000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f3678526000)
        /tmp/l/ld.so (0x00007f3678bc8000)
```

有幾個被`gcc` `g++`調用的可執行文件如`cc1`、`cc1plus`等可能也得做同樣修改。之後`gcc`即可執行，但編譯生成的可執行文件還使用原先的dynamic linkter，也許會在某些情況下無法執行。一種辦法是在用`gcc` `g++`鏈接時指定`-Wl,--dynamic-linker,/tmp/l/ld.so`，這樣產生的可執行文件就會使用`/tmp/l/ld.so`作爲dynamic linker。另一種方法是修改`gcc`中的built-in specs：

```
% gcc -dumpspecs
......
*link:
...%{m16|m32|mx32:;:-dynamic-linker %{muclibc:/lib/ld64-uClibc.so.0;:%{mbionic:/system/bin/linker64;:/lib64/ld-linux-x86-64.so.2}}}...
......
```

用`bvi`、`bviplus`等十六進制編輯器在可執行文件`gcc`中找這段字串，把`/lib64/ld-linux-x86-64.so.2}}}`改成`/tmp/l/ld.so}}}`，原串多出來的地方用空格填充。這樣得到的`gcc`產生的可執行文件無需指定`-Wl,--dynamic-linker`，默認就會使用新的dynamic linker了：

```
% bin/gcc.new -xc - -o a <<< 'int main(){}'
% ldd a
        linux-vdso.so.1 (0x00007ffeff9b3000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f337fe5b000)
        /tmp/l/ld.so (0x00007f33801ff000)
```

## 跳出XY problem

如果要裝的東西多些，可以考慮Gentoo Prefix、[NixOS](http://nixos.org/)等，還有unprivileged LXC containers等。輕量省事的方法是userspace chroot [PRoot](http://proot.me/)，不需要root，但弊端是運行的進程處於ptrace下，而一個進程不能被ptrace兩次，因此gdb、strace等都沒法用。
