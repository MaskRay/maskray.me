layout: post
title: 從-fpatchable-function-entry=N[,M]說起
author: MaskRay
tags: [llvm,gcc,binutils]
---

Linux kernel用了很多GCC選項支持ftrace。

<!-- more -->

* `-pg`
* `-mfentry`
* `-mnop-mcount`
* `-mprofile-kernel` powerpc
* `-mrecord-mcount`
* `-mhotpatch=pre-halfwords,post-halfwords`
* `-fpatchable-function-entry=N[,M]`

在當前GCC git repo的“史前”時期(Initial revision)就能看到`-pg`支持了。`-pg`在函數prologue後插入`mcount()`(Linux x86)，在其他OS或arch上可能叫不同名字，如`_mcount`、`__mcount`、`.mcount`。
trace信息可用於gprof和gcov。

```asm
# gcc -S -pg -O3 -fno-asynchronous-unwind-tables
foo:
	pushq	%rbp
	movq	%rsp, %rbp
1:	call	*mcount@GOTPCREL(%rip)
```

`-pg`作用在inlining後。

* 鏈接時GCC會選擇一個不同的crt1文件`gcrt1.o`
* libc實現`gcrt1.o` (glibc `sysdeps/x86_64/_mcount.S` `gmon/mcount.c`, FreeBSD `sys/amd64/amd64/prof_machdep.c`)。
* musl不提供`gcrt1.o` <https://www.openwall.com/lists/musl/2014/11/05/2>

glibc的用法：

* `gcrt1.o`定義`__gmon_start__`。其他`crt1.o`沒有定義
* `crti.o`用undefined weak `__gmon_start__`檢測`gcrt1.o`，是則調用
* `gcrt1.o`的`__gmon_start__`調用`__monstartup`初始化。在程序運行前初始化完可以避免call-once的同步。

[GCC r21495 (1998)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=07417085a14349cde788c5cc10663815da40c26f)引入`-finstrument-functions`，
在函數prologue後插入`__cyg_profile_func_enter(this_fn, call_site)`、epilogue前插入`__cyg_profile_func_enter(callee, call_site)`。程序實現這兩個函數後可以記錄函數調用。
這兩個函數分別有兩個參數，對code size有較大影響。另外，很多應用其實不需要`call_site`這個參數。

```c
void __cyg_profile_func_enter(void *this_fn, void *call_site);
void __cyg_profile_func_exit(void *this_fn, void *call_site);
```

```asm
# gcc -S -O3 -finstrument-functions -fno-asynchronous-unwind-tables
foo:
  subq    $8, %rsp
  leaq    foo(%rip), %rdi
  movq    8(%rsp), %rsi
  call    __cyg_profile_func_enter@PLT
  movq    8(%rsp), %rsi
  leaq    foo(%rip), %rdi
  call    __cyg_profile_func_exit@PLT
  xorl    %eax, %eax
  addq    $8, %rsp
  ret
```

`-finstrument-functions`默認作用在inlining前，能較好地體現控制流，但有很多的開銷。原因是inlining後，一個函數裏可能有多個`__cyg_profile_func_enter()`。
clang提供了`-finstrument-functions-after-inlining`在inlining後再trace。
GCC x86的`-pg -mfentry -minstrument-return=call`可以在函數返回時插入`call __return__`，可以作爲`-finstrument-functions -finstrument-functions-after-inlining`的替代品。

Linux kernel 2008年最早的ftrace實現[16444a8a40d](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901)使用`-pg`和`mcount`。
Linux定義了`mcount`，比較一個函數指針來檢查ftrace是否開啓，倘若沒有開啓，`mcount`則相當於一個空函數。
```c
#ifdef CONFIG_FTRACE
ENTRY(mcount)
	cmpq $ftrace_stub, ftrace_trace_function
	jnz trace
.globl ftrace_stub
ftrace_stub:
	...
#endif
```

所有函數的prologue後都執行`call mcount`，會產生很大的開銷。因此，後來Linux kernel在一個hash table裏記錄mcount的caller的PC，用一個一秒運行一次的daemon檢查hash table，把不需要trace的函數的`call mcount`修改成NOP。

之後，[8da3821ba56](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901)把"JIT"改成了"AOT"。
構建時，一個Perl script `scripts/recordmcount.pl`調用objdump記錄所有`call mcount`的地址，存儲在`__mcount_loc` section裏。Kernel啓動時預先把所有`call mcount`修改成NOP，免去了daemon。
由於Perl+objdump太慢，2010年，[16444a8a40d](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901)添加了一個C實現`scripts/recordmcount.c`。

mcount有一個弊端是stack frame size難以確定，ftrace不能訪問tracee的參數。
[GCC r162651 (2010) (GCC 4.6)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=3c5273a96ba8dbf98c40bc6d9d0a1587b4cfedb2)引入`-mfentry`，把prologue後的`call mcount`改成prologue前的`call __fentry__`。
2011年，[d57c5d51a30](https://github.com/torvalds/linux/commit/d57c5d51a30152f3175d2344cb6395f08bf8ee0c)添加了x86-64的`-mfentry`支持。

[GCC r206111 (2013)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d0de9e136f1dbe307e1d6ebb04b23131056cfa29)引入了SystemZ特有的`-mhotpatch`。
注意描述，function entry後僅有一個NOP，對entry前的NOP類型進行了限定。這樣缺乏通用性，其他arch用不上。後來一般化爲`-mhotpatch=pre-halfwords,post-halfwords`。

[GCC b54214fe22107618e7dd7c6abd3bff9526fcb3e5 (2013-03)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=b54214fe22107618e7dd7c6abd3bff9526fcb3e5)移植`-mprofile-kernel`到PowerPC64 ELFv2。
2016年[powerpc/ftrace: Add Kconfig & Make glue for mprofile-kernel](https://git.kernel.org/linus/8c50b72a3b4f1f7cdfdfebd233b1cbd121262e65)和之前的幾個commits用上了這個選項。

[GCC r215629 (2014)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=ecc81e33123d7ac9c11742161e128858d844b99d)引入`-mrecord-mcount`、`-mnop-mcount`。
`-mrecord-mcount`用於代替`linux/scripts/record_mcount.{pl,c}`。`-mnop-mcount`不可用於PIC，把`__fentry__`替換成NOP。
設計時沒有考慮通用性，大多數RISC都用不上不帶參數的`-mnop-mcount`。截至今天，`-mnop-mcount`只有x86和SystemZ支持。

(2019年，Linux x86移除了mcount支持[562e14f7229](https://github.com/torvalds/linux/commit/562e14f72292249e52e6346a9e3a30be652b0cf6)。)

[GCC r250521 (2017)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=417ca0117a1a9a8aaf5bc5ca530adfd68cb00399)引入`-fpatchable-function-entry=N[,M]`。
和SystemZ特有選項`-mhotpatch=`類似，在function entry前插入M個NOP，在entry後插入N-M個NOP。現在被Linux arm64和parisc採用。這個功能設計理念挺好的，可惜實現有諸多問題，僅能用於Linux kernel。

2018年GCC x86引入了`-minstrument-return=call`用於配合`-pg -mfentry`在函數返回時插入`call __return__`。
`-minstrument-return=nop5`則是插入一個5-byte nop。

```asm
# gcc -fpatchable-function-entry=3,1 -S -O3 a.c -fno-asynchronous-unwind-tables
	.section	__patchable_function_entries,"aw",@progbits
	.quad	.LPFE1
	.text
.LPFE1:
	nop
	.type	foo, @function
foo:
	nop
	nop
	xorl	%eax, %eax
	ret
```

* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93197> `__patchable_function_entries`會被`ld --gc-sections`(linker section garbage collection)收集。導致GCC的實現無法用於大部分程序。这个问题最终在添加PowerPC ELFv2支持时被完全修复<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99899> (milestone: 13.0)
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93195> `__patchable_function_entries` entry所屬的COMDAT section group被收集會產生鏈接錯誤。導致很多使用`inline`的C++程序無法使用。
* 錯誤信息寫錯選項名：`gcc -fpatchable-function-entry=a -c a.c` => `cc1: error: invalid arguments for ‘-fpatchable_function_entry’`
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93194> `__patchable_function_entries`沒有指定section alignment。我的第二個GCC patch～
* `__patchable_function_entries`的entries應用PC-relative relocations，而非absolute relocations，避免鏈接後生成`R_*_RELATIVE` dynamic relocations。這一點我一開始不能接受，因爲其他缺陷clang這邊修復後也能保持backward compatible，但relocation type是沒法改的。後來我認識到MIPS沒有提供`R_MIPS_PC64`……那麼選擇原諒GCC了。MIPS就是這樣，ISA缺陷->psABI“發明”聰明的ELF技巧繞過+引入新的問題。"mips is really the worst abi i've ever seen." "you mean worst dozen abis ;"
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=92424> AArch64 Branch Target Identification開啓時，NOP sled應在BTI後
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93492> x86 Indirect Branch Tracking開啓時，NOP sled應在ENDBR32/ENDBR64後。在開始實現-fpatchable-function-entry=前，正巧給lld加-z force-ibt。因此在看到AArch64問題很自然地想到了x86也有類似問題。
* 沒有考慮和`-fasynchronous-unwind-tables`的協作。再一次，Linux kernel使用`-fno-asynchronous-unwind-tables`。所以GCC實現時很自然地沒有思考這個問題
* Initial `.loc` directive應在NOP sled前。會導致symbolize function address得不到文件名/行號信息

修復`--gc-sections`和COMDAT比較棘手，還需要binutils這邊的GNU as和GNU ld的功能：

* <https://sourceware.org/bugzilla/show_bug.cgi?id=25380>支持unique section ID。2月2日GNU as添加了支持<https://sourceware.org/ml/binutils/2020-02/msg00020.html>
* <https://sourceware.org/bugzilla/show_bug.cgi?id=25381>支持`SHF_LINK_ORDER`。HJ Lu發了patch：<https://sourceware.org/ml/binutils/2020-02/msg00028.html>
* GNU ld --gc-sections semantics <https://sourceware.org/ml/binutils/2019-11/msg00266.html>

除AArch64 BTI外，其餘問題都是我報告的～

給clang添加`-fpatchable-function-entry=`的步驟如下：

* [D72215](https://reviews.llvm.org/D72215) 引入LLVM function attribute "patchable-function-entry"，AArch64 AsmPrinter支持
* [D72220](https://reviews.llvm.org/D72220) x86 AsmPrinter支持
* [D72221](https://reviews.llvm.org/D72221) 在clang裏實現function attribute `__attribute__((patchable_function_entry(0,0)))`
* [D72222](https://reviews.llvm.org/D72222) 給clang添加driver option `-fpatchable-function-entry=N[,0]`
* [D73070](https://reviews.llvm.org/D73070) 引入LLVM function attribute "patchable-function-prefix"
* 移動codegen passes，改變NOP sled與BTI/ENDBR的順序，順便修好了XRay、-mfentry與-fcf-protection=branch的協作。
* [D73680](https://reviews.llvm.org/D73680) AArch64 BTI，處理M=0時，patch label的位置：`bti c; .Lpatch0: nop`而不是`.Lpatch0: bti c; nop`
* x86 ENDBR32/ENDBR64，處理M=0時，patch label的位置：`endbr64; .Lpatch0: nop`而不是`.Lpatch0: endbr64; nop`

上述patches，除了x86 ENDBR的patch label位置调整，都会包含在clang 10.0.0里。

在-fpatchable-function-entry=之前，clang已經有多種在function entry插入代碼的方法了：

* `-fxray-instrument`。XRay使用類似`-finstrument-functions`的方法trace，和Linux kernel類似，運行時修改代碼
* Azul Systems引入了PatchableFunction用於JIT。我引入"patchable-function-entry"時就複用了這個pass
* IR feature: prologue data，在function entry後添加任意字節。用於function sanitizer
* IR feature: prefix data，在function entry前添加任意字節。用於GHC `TABLES_NEXT_TO_CODE`。Info table放在entry code前。GHC的LLVM後端目前仍是年久失修狀態

## PowerPC64 ELFv2

PowerPC ELFv2的實現見<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99888>

`-fpatchable-function-entry=5,2`輸出如下：

```asm
        .globl foo
        .type   foo, @function
foo:
.LFB0:
        .cfi_startproc
.LCF0:
0:      addis 2,12,.TOC.-.LCF0@ha
        addi 2,2,.TOC.-.LCF0@l
        .section        __patchable_function_entries,"awo",@progbits,foo
        .align 3
        .8byte  .LPFE1
        .section        ".text"
.LPFE1:
        nop
        nop
        .localentry     foo,.-foo
        nop
        nop
        nop
        mflr 0
        std 0,16(1)
        stdu 1,-32(1)
```

NOPs在global entry後。Local entry前後分別有M、N-M個NOPs。
因爲global entry和local entry間距有限制，M只能取0 (2-2)、2 (4-2)、6 (8-2)、14 (16-2)等少數值。

在PR99888中，我在2022年就提出沒有必要讓NOP連續。2023年末的討論也說明了當前連續NOP不方便kernel和userspace live patching。
https://gcc.gnu.org/bugzilla/show_bug.cgi?id=112980 打算修改實現。
