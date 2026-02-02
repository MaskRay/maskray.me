---
layout: post
title: lld 18 ELF changes
author: MaskRay
tags: [linker,llvm,release]
---

LLVM 18 will be released.
As usual, I maintain lld/ELF and have added some notes to <https://github.com/llvm/llvm-project/blob/release/18.x/lld/docs/ReleaseNotes.rst>.
I've meticulously reviewed nearly all the patches that are not authored by me.
I'll delve into some of the key changes.

<!-- more -->

* `--fat-lto-objects` option is added to support LLVM FatLTO. Without `--fat-lto-objects`, LLD will link LLVM FatLTO objects using the relocatable object file.
  (`D146778 <https://reviews.llvm.org/D146778>`_)
* `-Bsymbolic-non-weak` is added to directly bind non-weak definitions.
  ([D158322](https://reviews.llvm.org/D158322))
* `--lto-validate-all-vtables-have-type-infos`, which complements
  `--lto-whole-program-visibility`, is added to disable unsafe whole-program
  devirtualization. `--lto-known-safe-vtables=<glob>` can be used
  to mark known-safe vtable symbols.
  ([D155659](https://reviews.llvm.org/D155659))
* `--save-temps --lto-emit-asm` now derives ELF/asm file names from bitcode file names.
  `ld.lld --save-temps a.o d/b.o -o out` will create ELF relocatable files
  `out.lto.a.o`/`d/out.lto.b.o` instead of `out1.lto.o`/`out2.lto.o`.
  ([#78835](https://github.com/llvm/llvm-project/pull/78835))
* `--no-allow-shlib-undefined` now reports errors for DSO referencing
  non-exported definitions.
  ([#70769](https://github.com/llvm/llvm-project/pull/70769))
* common-page-size can now be larger than the system page-size.
  ([#57618](https://github.com/llvm/llvm-project/issues/57618))
* When call graph profile information is available due to instrumentation or
  sample PGO, input sections are now sorted using the new `cdsort` algorithm,
  better than the previous `hfsort` algorithm.
  ([D152840](https://reviews.llvm.org/D152840))
* Symbol assignments like `a = DEFINED(a) ? a : 0;` are now handled.
  ([#65866](https://github.com/llvm/llvm-project/pull/65866))
* `OVERLAY` now supports optional start address and LMA
  ([#77272](https://github.com/llvm/llvm-project/pull/77272))
* Relocations referencing a symbol defined in `/DISCARD/` section now lead to
  an error.
  ([#69295](https://github.com/llvm/llvm-project/pull/69295))
* For AArch64 MTE, global variable descriptors have been implemented.
  ([D152921](https://reviews.llvm.org/D152921))
* `R_AARCH64_GOTPCREL32` is now supported.
  ([#72584](https://github.com/llvm/llvm-project/pull/72584))
* `R_LARCH_PCREL20_S2`/`R_LARCH_ADD6`/`R_LARCH_CALL36` and extreme code
  model relocations are now supported.
* `--emit-relocs` is now supported for RISC-V linker relaxation.
  ([D159082](https://reviews.llvm.org/D159082))
* Call relaxation respects RVC when mixing +c and -c relocatable files.
  ([#73977](https://github.com/llvm/llvm-project/pull/73977))
* `R_RISCV_GOT32_PCREL` is now supported.
  ([#72587](https://github.com/llvm/llvm-project/pull/72587))
* `R_RISCV_SET_ULEB128`/`R_RISCV_SUB_ULEB128` relocations are now supported.
  ([#72610](https://github.com/llvm/llvm-project/pull/72610))
  ([#77261](https://github.com/llvm/llvm-project/pull/77261))
* RISC-V TLSDESC is now supported.
  ([#79239](https://github.com/llvm/llvm-project/pull/79239))
* SystemZ (s390x) is now supported.
  ([#75643](https://github.com/llvm/llvm-project/pull/75643))

Although a substantial feature, the s390x port benefited from Ulrich Weigand's meticulously prepared patch, complete with comprehensive tests from the outset.
While I typically provide extensive feedback for such large additions, the patch's exceptional quality minimized the number of comment rounds needed in this instance.
I truly appreciate Ulrich taking the time to reply to [my remarks on the s390x ABI](/blog/2024-02-11-toolchain-notes-on-z-architecture). His guidance was instrumental.

The RISC-V port has received a few new relocations for `.uleb128` label differences and TLSDESC.
I'm glad I made these assembler and linker changes in time, allowing LoongArch developers to port the code for LoongArch.
LoongArch borrows many designs from RISC-V.
If they were to implement these features first, I'd probably have to spend more time on code reviews, and the outcome would be less rewarding since I wouldn't be the original patch author.

`R_AARCH64_GOTPCREL32` (`G(GDAT(S))+A-P`) and `R_RISCV_GOT32_PCREL`, similar to `R_X86_64_GOTPCREL`, are new ABI additions used to optimize preemptible `_ZTI*` symbol references for `clang -fexperimental-relative-c++-abi-vtables`.
The simplification utilizes the [optimization originally done for Mach-O](https://reviews.llvm.org/D6922), which applies to GOT equivalent global variables (global unnamed, constant, discardable linkage).
```diff
        .long   0                               # 0x0
-       .long   (_ZTI1A.rtti_proxy-.L_ZTV1A.local)-8
+       .long   _ZTI1A@GOTPCREL-4
        .long   (_ZN1A3fooEv@PLT-.L_ZTV1A.local)-8

-       .hidden _ZTI1A.rtti_proxy               # @_ZTI1A.rtti_proxy
-       .type   _ZTI1A.rtti_proxy,@object
-       .section        .data.rel.ro._ZTI1A.rtti_proxy,"aGw",@progbits,_ZTI1A.rtti_proxy,comdat
-       .globl  _ZTI1A.rtti_proxy
-       .p2align        3, 0x0
-_ZTI1A.rtti_proxy:
-       .quad   _ZTI1A
-       .size   _ZTI1A.rtti_proxy, 8
```

To the best of my knowledge, there is no performance-specific change.

Link: [lld 17 ELF changes](/blog/2023-07-30-lld-17-elf-changes)
