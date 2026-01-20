---
layout: post
title: -fpatchable-function-entry=N[,M]
author: MaskRay
tags: [llvm,gcc,binutils]
---

The Linux kernel uses many GCC options to support ftrace.

<!-- more -->

* `-pg`
* `-mfentry`
* `-mnop-mcount`
* `-mprofile-kernel` powerpc
* `-mrecord-mcount`
* `-mhotpatch=pre-halfwords,post-halfwords`
* `-fpatchable-function-entry=N[,M]`

In the "prehistoric" period (Initial revision) of the current GCC git repo, `-pg` support was already visible. `-pg` inserts `mcount()` after the function prologue (Linux x86). On other OSes or architectures, it may have different names like `_mcount`, `__mcount`, `.mcount`.
Trace information can be used for gprof and gcov.

```asm
# gcc -S -pg -O3 -fno-asynchronous-unwind-tables
foo:
	pushq	%rbp
	movq	%rsp, %rbp
1:	call	*mcount@GOTPCREL(%rip)
```

`-pg` takes effect after inlining.

* At link time, GCC will choose a different crt1 file `gcrt1.o`
* libc implements `gcrt1.o` (glibc `sysdeps/x86_64/_mcount.S` `gmon/mcount.c`, FreeBSD `sys/amd64/amd64/prof_machdep.c`).
* musl does not provide `gcrt1.o` <https://www.openwall.com/lists/musl/2014/11/05/2>

Usage in glibc:

* `gcrt1.o` defines `__gmon_start__`. Other `crt1.o` files don't define it
* `crti.o` uses undefined weak `__gmon_start__` to detect `gcrt1.o`, and calls it if present
* `__gmon_start__` in `gcrt1.o` calls `__monstartup` for initialization. Initializing before program execution avoids call-once synchronization.

[GCC r21495 (1998)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=07417085a14349cde788c5cc10663815da40c26f) introduced `-finstrument-functions`,
inserting `__cyg_profile_func_enter(this_fn, call_site)` after the function prologue and `__cyg_profile_func_exit(callee, call_site)` before the epilogue. Programs can implement these two functions to record function calls.
These two functions each have two parameters, which significantly impacts code size. Additionally, many applications don't actually need the `call_site` parameter.

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

`-finstrument-functions` by default acts before inlining, providing good control flow representation but with significant overhead. The reason is that after inlining, a function may contain multiple `__cyg_profile_func_enter()` calls.
Clang provides `-finstrument-functions-after-inlining` to trace after inlining.
GCC x86's `-pg -mfentry -minstrument-return=call` can insert `call __return__` on function returns, serving as an alternative to `-finstrument-functions -finstrument-functions-after-inlining`.

The earliest ftrace implementation in Linux kernel 2008 [16444a8a40d](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901) used `-pg` and `mcount`.
Linux defined `mcount`, comparing a function pointer to check if ftrace is enabled. If not enabled, `mcount` acts like an empty function.

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

All functions execute `call mcount` after their prologue, creating significant overhead. Therefore, later Linux kernel recorded mcount caller PCs in a hash table, using a daemon that runs once per second to check the hash table and modify `call mcount` to NOP for functions that don't need tracing.

Later, [8da3821ba56](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901) changed from "JIT" to "AOT".
At build time, a Perl script `scripts/recordmcount.pl` calls objdump to record all `call mcount` addresses, storing them in the `__mcount_loc` section. During kernel startup, all `call mcount` instructions are preemptively modified to NOPs, eliminating the need for a daemon.
Since Perl+objdump was too slow, in 2010, [16444a8a40d](https://github.com/torvalds/linux/commit/16444a8a40d4c7b4f6de34af0cae1f76a4f6c901) added a C implementation `scripts/recordmcount.c`.

mcount has the drawback that stack frame size is hard to determine, preventing ftrace from accessing tracee arguments.
[GCC r162651 (2010) (GCC 4.6)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=3c5273a96ba8dbf98c40bc6d9d0a1587b4cfedb2) introduced `-mfentry`, changing the post-prologue `call mcount` to pre-prologue `call __fentry__`.
In 2011, [d57c5d51a30](https://github.com/torvalds/linux/commit/d57c5d51a30152f3175d2344cb6395f08bf8ee0c) added x86-64 `-mfentry` support.

[GCC r206111 (2013)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d0de9e136f1dbe307e1d6ebb04b23131056cfa29) introduced SystemZ-specific `-mhotpatch`.
Note the description: only one NOP after function entry, with restrictions on NOP types before entry. This lacks generality and can't be used by other architectures. Later generalized to `-mhotpatch=pre-halfwords,post-halfwords`.

[GCC b54214fe22107618e7dd7c6abd3bff9526fcb3e5 (2013-03)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=b54214fe22107618e7dd7c6abd3bff9526fcb3e5) ported `-mprofile-kernel` to PowerPC64 ELFv2.
In 2016, [powerpc/ftrace: Add Kconfig & Make glue for mprofile-kernel](https://git.kernel.org/linus/8c50b72a3b4f1f7cdfdfebd233b1cbd121262e65) and several previous commits adopted this option.

[GCC r215629 (2014)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=ecc81e33123d7ac9c11742161e128858d844b99d) introduced `-mrecord-mcount` and `-mnop-mcount`.
`-mrecord-mcount` replaces `linux/scripts/record_mcount.{pl,c}`. `-mnop-mcount` cannot be used with PIC, replacing `__fentry__` with NOP.
The design didn't consider generality; most RISC architectures can't use parameterless `-mnop-mcount`. As of today, `-mnop-mcount` is only supported by x86 and SystemZ.

(In 2019, Linux x86 removed mcount support [562e14f7229](https://github.com/torvalds/linux/commit/562e14f72292249e52e6346a9e3a30be652b0cf6).)

[GCC r250521 (2017)](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=417ca0117a1a9a8aaf5bc5ca530adfd68cb00399) introduced `-fpatchable-function-entry=N[,M]`.
Similar to SystemZ-specific `-mhotpatch=`, it inserts M NOPs before function entry and N-M NOPs after entry. Now adopted by Linux arm64 and parisc. This feature has good design philosophy, but the implementation has many issues and can only be used in Linux kernel.

In 2018, GCC x86 introduced `-minstrument-return=call` for use with `-pg -mfentry` to insert `call __return__` on function returns.
`-minstrument-return=nop5` inserts a 5-byte nop.

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

* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93197> `__patchable_function_entries` gets collected by `ld --gc-sections` (linker section garbage collection). This makes GCC's implementation unusable for most programs. This issue was finally completely fixed when adding PowerPC ELFv2 support <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99899> (milestone: 13.0)
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93195> Collection of COMDAT section groups containing `__patchable_function_entries` entries causes linking errors. This makes many C++ programs using `inline` unusable.
* Incorrect option name in error message: `gcc -fpatchable-function-entry=a -c a.c` => `cc1: error: invalid arguments for '-fpatchable_function_entry'`
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93194> `__patchable_function_entries` doesn't specify section alignment. This was my second GCC patch~
* `__patchable_function_entries` entries should use PC-relative relocations, not absolute relocations, to avoid generating `R_*_RELATIVE` dynamic relocations after linking. I initially couldn't accept this, because other defects could be fixed on the clang side while maintaining backward compatibility, but relocation type cannot be changed. Later I realized MIPS doesn't provide `R_MIPS_PC64`... So I chose to forgive GCC. That's just how MIPS is: ISA defects -> psABI "invents" clever ELF tricks to work around + introduces new problems. "mips is really the worst abi i've ever seen." "you mean worst dozen abis ;"
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=92424> When AArch64 Branch Target Identification is enabled, NOP sled should be after BTI
* <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=93492> When x86 Indirect Branch Tracking is enabled, NOP sled should be after ENDBR32/ENDBR64. Before starting to implement -fpatchable-function-entry=, I happened to add -z force-ibt to lld. So seeing the AArch64 issue naturally made me think of similar issues on x86.
* No consideration for cooperation with `-fasynchronous-unwind-tables`. Again, Linux kernel uses `-fno-asynchronous-unwind-tables`. So during GCC implementation, this issue wasn't naturally considered
* Initial `.loc` directive should be before NOP sled. This causes symbolizing function addresses to fail to get filename/line number information

Fixing `--gc-sections` and COMDAT is quite tricky and requires functionality from GNU as and GNU ld on the binutils side:

* GNU Assembler 2.35 implemented unique section ID (<https://sourceware.org/bugzilla/show_bug.cgi?id=25380>)
* GNU Assembler 2.35 implemented `SHF_LINK_ORDER` (<https://sourceware.org/bugzilla/show_bug.cgi?id=25381>)
* GNU ld --gc-sections semantics <https://sourceware.org/ml/binutils/2019-11/msg00266.html>

Except for AArch64 BTI, all other issues were reported by me~

Steps to add `-fpatchable-function-entry=` to clang:

* [D72215](https://reviews.llvm.org/D72215) Introduce LLVM function attribute "patchable-function-entry", AArch64 AsmPrinter support
* [D72220](https://reviews.llvm.org/D72220) x86 AsmPrinter support
* [D72221](https://reviews.llvm.org/D72221) Implement function attribute `__attribute__((patchable_function_entry(0,0)))` in clang
* [D72222](https://reviews.llvm.org/D72222) Add driver option `-fpatchable-function-entry=N[,0]` to clang
* [D73070](https://reviews.llvm.org/D73070) Introduce LLVM function attribute "patchable-function-prefix"
* Move codegen passes, change NOP sled order with BTI/ENDBR, fixing cooperation between XRay, -mfentry and -fcf-protection=branch
* [D73680](https://reviews.llvm.org/D73680) AArch64 BTI, handle patch label position when M=0: `bti c; .Lpatch0: nop` instead of `.Lpatch0: bti c; nop`
* x86 ENDBR32/ENDBR64, handle patch label position when M=0: `endbr64; .Lpatch0: nop` instead of `.Lpatch0: endbr64; nop`

All the above patches, except for the x86 ENDBR patch label position adjustment, are included in clang 10.0.0.

Before `-fpatchable-function-entry=`, clang already had multiple methods to insert code at function entry:

* `-fxray-instrument`. XRay uses a method similar to `-finstrument-functions` for tracing, and like Linux kernel, modifies code at runtime
* Azul Systems introduced PatchableFunction for JIT. I reused this pass when introducing "patchable-function-entry"
* IR feature: prologue data, adds arbitrary bytes after function entry. Used for function sanitizer
* IR feature: prefix data, adds arbitrary bytes before function entry. Used for GHC `TABLES_NEXT_TO_CODE`. Info table is placed before entry code. GHC's LLVM backend is currently still in a state of disrepair

## PowerPC64 ELFv2

PowerPC ELFv2 implementation see <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99888>

`-fpatchable-function-entry=5,2` outputs:

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

NOPs are after global entry. There are M and N-M NOPs before and after local entry respectively.
Due to distance restrictions between global entry and local entry, M can only take a few values like 0 (2-2), 2 (4-2), 6 (8-2), 14 (16-2).

In PR99888, I proposed in 2022 that there's no need to make NOPs consecutive. The discussion at the end of 2023 also showed that current consecutive NOPs are inconvenient for kernel and userspace live patching.
In Nov 2024, `-msplit-patch-nops` is implemented to place `-msplit-patch-nops` before the function label. (<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=112980>)
