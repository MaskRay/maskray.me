layout: post
title: 模塊內函數調用和libc符號重命名
author: MaskRay
tags: [glibc,llvm]
---

## Translation unit外的函數調用

<!-- more -->

在一個libc實現中，有時會調用自身其他translation unit定義的函數，如：

```
#include <string.h>

void foo(void *src, const void *dst) {
  memcpy(dst, src, 90);
}
```

這個translation unit用`-fPIC`編譯成.o時會生成一個memcpy函數調用。在部分架構上有少量PIC設置的開銷(見下文)。

在一個ELF文件格式的shared object中，一個定義的non-local `STB_DEFAULT`符號默認爲preemptible(interposable)，即運行時可被替換。
對於一個preemptible符號，對其的引用不可綁定到模塊內部的定義。

這個.o用`-shared`方式鏈接時，即使memcpy定義在該shared object的另一個translation unit中，鏈接器也會假設memcpy在運行時可能被可執行文件或其他shared object在替換，看到表示函數調用的relocation types時得生成PLT。

```asm
# Although memcpy is within the module, the linker cannot resolve the relocation to the address of memcpy (direct call).
# It has to resolve the relocation to the PLT entry (symbolized as memcpy@plt in objdump).
f: call memcpy@PLT  # R_X86_64_PLT32

memcpy: ...
```

運行時替換提供了靈活性，但也犧牲了性能。一個函數調用需要執行PLT entry中幾條指令，從.got.plt(多數架構)讀取一個函數指針執行間接跳轉。
還要算上ld.so解析這個符號的一次性開銷。

在99.9%的情況下Procedure Linkage Table (PLT)的作用是調用一個定義在其他shared object或可執行文件中的函數。
(什麼？你要問我剩下的0.1%是什麼？是GNU indirect function中一種接近失傳、晦澀的技巧。今天binutils添加了RISC-V的ifunc支持，我就在之前的郵件中提到了LLD使用的方法)

## Translation unit內的函數調用和`-fno-semantic-interposition`

GCC 5引入了`-fno-semantic-interposition`。如果調用的函數定義在同一個translation unit，
```c
// f is global STV_DEFAULT.
void f() {}
void g() { f(); }
```

編譯器有兩種選擇：

* `-fsemantic-interposition` (GCC默認行爲): 保守地假設可能發生運行時preemption，阻止一切相關的inter-procedural optimizations(如inlining)。
```asm
f:
  ...
g:
# Inlining f into g may change semantics if f is preempted at runtime.
  call f@PLT
```
* `-fno-semantic-interposition`: 樂觀地假設不發生preemption，允許inter-procedural optimizations。這其實是Clang長期以來的行爲。(我的貢獻)Clang 11若指定該選項，會在LLVM IR層面設置dso_local。在x86上，如果函數最終沒有被inline，編譯器會生成對local alias `.Lfoo$local`的調用，不用GOT-generating relocations。Local symbols是non-preeemptible的，可以阻止鏈接器生成GOT和PLT。
```asm
f: # STB_GLOBAL
.Lf$local: # STB_LOCAL
  ...
g:
  call .Lf$local
```
如果f定義在另一個translation unit，這個選項仍然不能阻止PLT。

在Clang 11，Serge Guelton添加了`-fsemantic-interposition`。

## Link-time solution: `--dynamic-list`

libc中有大量庫函數會被其他庫函數調用。放任它們產生PLT會有可觀的性能損失。鏈接器提供了`-Bsymbolic`, `-Bsymbolic-functions`, version script和`--dynamic-list`等幾種機制使部分符號non-preemptible。

musl採用的方法是用`--dynamic-list`精細指定libc.so中preemptible的符號列表：
```
{
environ; __environ; stdin; stdout; stderr; timezone; daylight; tzname; ...
malloc; calloc; realloc; free; ...
}
```

在可執行文件和shared object中，`--dynamic-list`的語義不同。這裏取shared object語義：

* executable: Put matched non-local defined symbols to the dynamic symbol table
* shared object: Matched defined non-local `STV_DEFAULT` symbols are non-preemptible, and others are preemptible. (Implies `-Bsymbolic` but does not set `DF_SYMBOLIC`.)
  (References to preemtible symbols cannot be bound to the definitions within the shared object.)

大多數函數都不在這個列表中，因爲大多數程序都不能被用戶程序重定義。C標準規定在程序中定義任何標準庫函數都是undefined behavior，但在實際中很多實現會放寬要求以允許可替代的malloc實現(最著名的是通過`LD_PRELOAD`使用的jemalloc和tcmalloc)。另外sanitizers也會preempt大量庫函數。

musl 1.1.20起支持用戶程序替換少量malloc相關函數，因此這些這些函數在dynamic list中。

在1.1.20之前，musl的malloc實現不可被替換。musl使用功能比`--dynamic-list`弱的`-Bsymbolic-functions`：所有的`STT_FUNC`符號non-preemptible，而所有`STT_OBJECT`符號仍是preemptible的。

`-fno-PIC`方式編譯的translation unit只能用於可執行文件。
傳統上，很多架構的`-fno-PIC`會用absolute relocation或PC-relative relocation訪問外部`STT_OBJECT`符號，不用Global Offset Table (GOT)。
當訪問的符號定義在一個shared object中時，就會在鏈接時產生copy relocation (an ugly hack)：可執行文件把shared object的若干字節複製過來，讓shared object對該符號的解析重定向到可執行文件。

對於copy relocation，只有使符號preemptible才能維持程序的一致性。倘若non-preemptible，就會產生可執行文件和shared object操作不同拷貝的情形。

```c
// On x86, direct access is generated in both -fno-PIC and -fPIE.
// stdout requires a copy relocation.
// If libc.so is linked with -Bsymbolic, modifying stdout in the executable will not be observed by libc.so, vice versa.
#include <stdio.h>
int main() { fprintf(stdout, "%d", 42); }
```

### PLT設置的開銷

使用鏈接選項的方案很優雅。對於具有PC relative訪問數據的指令的架構，這種方案是完美的。
在其他架構上，一個需要PLT的外部函數調用會有額外開銷(鏈接選項不影響編譯期)。下面展示編譯這段C程序得到的彙編指令：

```c
#ifdef HIDDEN
void ext() __attribute__((visibility("hidden")));
#else
void ext();
#endif
void f() { ext(); ext(); }
```

在具有PC relative訪問數據的指令的架構上，ext是否hidden，生成的指令序列沒有變化。

#### i386

在i386上，ABI要求訪問PLT時ebx指向GOT base，因爲PLT entry會用基於ebx的尋址加載.got.plt(或罕見的.plt.got)的函數指針。外部函數前需要設置ebx。
因爲ebx是call-saved的，如果一個函數修改了ebx，需要保證返回時ebx被還原，因此還有額外的save/restore。

```asm
# ext is STV_DEFAULT
f:
	pushl	%ebx
	call	__x86.get_pc_thunk.bx
	addl	$_GLOBAL_OFFSET_TABLE_, %ebx
	subl	$8, %esp
	call	ext@PLT
	call	ext@PLT
	addl	$8, %esp
	popl	%ebx
	ret
```

```asm
# ext is STV_HIDDEN
f:
	subl	$12, %esp
	call	ext
	addl	$12, %esp
	jmp	ext
```

#### PowerPC64

POWER10有PC-relative訪問數據的指令。之前，ELFv2用TOC (Table Of Contents)降低缺乏PC-relative指令帶來的性能開銷。
ABI要求r2指向當前module (可執行文件或shared object)的TOC base。一個外部函數調用會修改r2，因此一個bl指令後需要恢復r2。
編譯器會在每一條外部bl指令後放置一個nop(他們一定是受到了Mips delay slot的啓發)，鏈接時按需patch成ld指令。
除了多餘的nop開銷外，還導致了tail call失效。

```asm
# ext is STV_DEFAULT
f:
.Lfunc_begin0:
.Lfunc_gep0:
	addis 2, 12, .TOC.-.Lfunc_gep0@ha
	addi 2, 2, .TOC.-.Lfunc_gep0@l
.Lfunc_lep0:
	.localentry	f, .Lfunc_lep0-.Lfunc_gep0
	mflr 0
	std 0, 16(1)
	stdu 1, -32(1)
	bl ext
	nop
	bl ext
	nop
	addi 1, 1, 32
	ld 0, 16(1)
	mtlr 0
	blr
```

```asm
# ext is STV_HIDDEN
f:
.Lfunc_begin0:
.Lfunc_gep0:
	addis 2, 12, .TOC.-.Lfunc_gep0@ha
	addi 2, 2, .TOC.-.Lfunc_gep0@l
.Lfunc_lep0:
	.localentry	f, .Lfunc_lep0-.Lfunc_gep0
	mflr 0
	std 0, 16(1)
	stdu 1, -32(1)
	bl ext
	addi 1, 1, 32
	ld 0, 16(1)
	mtlr 0
	b ext
```

意外地，兩種函數調用方式在Mips上沒有差別，可能是因爲它們的指令序列已經很長了吧……

## Compile-time solution: hidden aliases

glibc採取的方式是

* 定義`STV_DEFAULT`的memcpy和一個hidden alias `__GI_memcpy`
* 內部header聲明memcpy時asm label指向hidden的`__GI_memcpy`
* 不使用`-fno-builtin-memcpy`，允許memcpy函數調用被內聯
* 引用memcpy處include內部header。memcpy或者被內聯，或者生成對`__GI_memcpy`的調用

因爲`__GI_memcpy`是hidden的，編譯器/鏈接器知道它的定義在模塊內部，且不能由其他模塊提供，因此能避免PLT設置開銷。

```c
extern void *memcpy(void *__restrict, const void *__restrict, unsigned long);
extern __typeof(memcpy) memcpy __asm__("__GI_memcpy") __attribute__((visibility("hidden")));

void f() {
  memcpy(...);
}
```

所以，爲什麼不直接調用`__GI_memcpy`呢？因爲這樣mangle函數名用戶體驗不好……

其實musl在很多地方也用了`__`開頭的hidden aliases，需要避免PLT設置開銷時就會調用這些函數，如：

```c
// src/include/sys/mman.h
__attribute__((__visibility__("hidden"))) void *__mmap(void *, size_t, int, int, int, off_t);

// src/mman/mmap.c
void *__mmap(void *start, size_t len, int prot, int flags, int fd, off_t off)
{
  ...
}

extern __typeof(__mmap) mmap __attribute__((__weak__, __alias__("__mmap")));
```

### `STV_PROTECTED`

其實，除了`STV_DEFAULT`和`STV_HIDDEN`外，還有另一種在這種場景下更適合的visibility：`STV_PROTECTED`。
有一個缺點是歷史上`STV_PROTECTED`使用非常少，缺乏測試。但缺乏測試在最近若干年應該不是問題了。

`STV_PROTECTED`在ELF specification中的定義如下，暴露給外界，且non-preemptible，看上去是完美解決方案。

> A symbol defined in the current component is protected if it is visible in other components but not preemptable, meaning that any reference to such a symbol from within the defining component must be resolved to the definition in that component, even if there is a definition in another component that would preempt by the default rules.

那麼爲什麼libc不用`STV_PROTECTED`呢？

傳統上，很多結構上`-fno-PIC`取外部函數地址就像訪問外部`STT_OBJECT`符號那樣，會用absolute relocation或PC-relative relocation，不用Global Offset Table (GOT)。
鏈接器會在可執行文件裏創建一個st_value!=0的PLT entry(相當於`STT_FUNC`的copy relocation)。可執行文件中該函數的地址就是這個PLT entry的運行時地址。
爲了pointer equality，鏈接器會試圖讓該PLT entry preempt shared object中的定義。然而，`STV_PROTECTED`是不允許preemption的，衝突導致報錯。

```c
// b.c - b.so
__attribute__((visibility("protected"))) void foo() {}
void *addr_foo() { return (void *)foo; }

// a.c - exe
#include <stdio.h>
void foo();
int main() { printf("%p\n", foo); }
```

鏈接時會報錯：
```
# ld.lld
error: cannot preempt symbol: foo

# ld.bfd
relocation R_X86_64_32 against protected symbol `foo' can not be used when making a PIE object; recompile with -fPIE
```

假如鏈接器允許可執行文件中的absolute relocation或PC-relative relocation，運行時可執行文件中foo的地址會和shared object中foo的地址不一致。

## asm label in Clang

對於大多數編譯器不認識的函數(沒有內建知識，不能內聯(expand memcpy)或替換成其他實現(printf->puts)，不能生成lower成該函數的intrinsics)，asm label的實現方式都是挺直接的。
然而，在今天之前的Clang裏，有不少庫函數(包括最重要的memset/memcpy)的asm label沒有效果。我今天修復了這個問題[D88712](https://reviews.llvm.org/D88712)。

這裏的主要難點是如果C函數foo含有內建語義X，且符號foo含有內建語義X，那麼拒絕編譯C函數foo爲符號foo是不合邏輯的。
換言之，下述三條不可同時成立。

* 如果frontend函數foo含有內建語義X
* 符號foo含有內建語義X
* C函數foo不能編譯爲符號foo

在glibc的場合下，第一條是需要的。如果編譯器假裝不認識memcpy，那麼就無法展開n爲常數的memcpy，可能會影響性能。這也表明`-fno-builtin-memcpy`(或更強的`-fno-builtin`和`-ffreestanding`)不可接受。
第三條也是需要的，因爲使用asm label的目的就是重命名啊……

這樣我們就得駁斥第二條。換言之，Clang生成LLVM IR後，IR優化不可假設符號foo具有內建語義X。然而這在GCC和Clang中都無法做到。
Clang若想實現，得引入LLVM IR特性支持重命名。倘若不支持重命名，得知道會lower成符號foo的intrinsics不可生成。
這個功能目前是缺失的。

不能駁斥第二條給整個系統帶來了一點不一致性。glibc的處理方式是加第二層重命名，給每個translation unit加一條`asm("memcpy = __GI_memcpy;")`

```asm
memcpy = __GI_memcpy;

# If __GI_memcpy is undefined, this produces a relocation referencing __GI_memcpy.
call memcpy@PLT
```

* 對於不`#include <string.h>`的translation unit，asm label不可替代這個重命名。
* 對於GCC優化過程中合成的memcpy，這行asm保證了GNU as會實施重命名。

我有另一個patch實現GNU as的這個邏輯。

另外，Clang支持繼承自Sun Studio的另一種重命名語法：`#pragma redefine_extname oldname newname`。內部這一功能是用asm label實現的。
GCC文檔中提到了這個功能<https://gcc.gnu.org/onlinedocs/gcc/Symbol-Renaming-Pragmas.html>，但我測試不可用……

## Redeclaration

A function can be redeclared multiple times. The requirement is that
an asm label is added before the first use.

```c
typedef unsigned long size_t;

// #pragma redefine_extname memcpy __GI_memcpy // before the first use, works
extern void *memcpy(void *, const void *, size_t);

#pragma redefine_extname memcpy __GI_memcpy // before the first use, works

void *test_memcpy(void *dst, const void *src, size_t n) { return memcpy(dst, src, n); }

// #pragma redefine_extname memcpy __GI_memcpy // after the first use, does not work
```

As <https://reviews.llvm.org/D88712> mentions, the asm label does not
apply to Clang generated memcpy calls.
Certain optimization passes can synthesize built-in function calls.
It's really difficult to fix, also related to the function-at-a-time
mode.
