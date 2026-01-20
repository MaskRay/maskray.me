---
layout: post
title: My involvement with LLVM 19
author: MaskRay
tags: [llvm]
---

LLVM 19.1 will soon be released. This post provides a summary of my contributions in this release cycle to record my learning progress.

<!-- more -->

* LLVM MC: [Integrated assembler improvements in LLVM 19](/blog/2024-06-30-integrated-assembler-improvements-in-llvm-19)
* lld/ELF: [lld 19 ELF changes](/blog/2024-08-04-lld-19-elf-changes)
* AArch64
  + [[AArch64] Implement -fno-plt for SelectionDAG/GlobalISel](https://github.com/llvm/llvm-project/pull/78890)
  + [[AArch64] Support optional constant offset for constraint "S"](https://github.com/llvm/llvm-project/pull/80255)
  + [[AArch64AsmParser] Allow branch target symbol to have a shift/extend modifier name](https://github.com/llvm/llvm-project/pull/80571)
  + [[MC,AArch64] Create mapping symbols with non-unique names](https://github.com/llvm/llvm-project/pull/99836)
* ARM
  + [[ARM,MC] Support FDPIC relocations](https://github.com/llvm/llvm-project/pull/82187). See [MMU-less systems and FDPIC](/blog/2024-02-20-mmu-less-systems-and-fdpic)
* RISC-V
  + [[RISCV] Support constraint "s"](https://github.com/llvm/llvm-project/pull/80201)
  + [[MC,RISCV] Create mapping symbols with non-unique names](https://github.com/llvm/llvm-project/pull/99903)
* sanitizers
  + [[ubsan] Support static linking with standalone runtime](https://github.com/llvm/llvm-project/pull/80943)
  + [[sanitizer] Reject unsupported -static at link time](https://github.com/llvm/llvm-project/pull/83524)
  + [__asan_register_elf_globals: properly check the "no instrumented global variable" case](https://github.com/llvm/llvm-project/pull/96529)
  + [[asan,test] Disable _FORTIFY_SOURCE test incompatible with glibc 2.40](https://github.com/llvm/llvm-project/pull/101566)

## LLVM binary utilities

+ [[llvm-readobj,ELF] Support --decompress/-z](https://github.com/llvm/llvm-project/pull/82594)
+ [[llvm-objcopy] Improve help messages](https://github.com/llvm/llvm-project/pull/82830)
+ [[llvm-readelf] Print a blank line for the first hex/string dump](https://github.com/llvm/llvm-project/pull/85744)
+ [[llvm-objcopy] Add --compress-sections](https://github.com/llvm/llvm-project/pull/85036)
+ [[llvm-readelf] Print more information for RELR](https://github.com/llvm/llvm-project/pull/89162)

## Hashing

I optimized the bit mixer used by `llvm::DenseMap<std::pair<X, Y>>` and `llvm::DenseMap<std::tuple<X...>>`.
`llvm/ADT/Hashing.h`, used by `StringRef` hashing and `DenseMap`, was supposed to be non-deterministic. Despite this, a lot of code relied on a specific iteration order.
I made multiple fixes across the code base and landed [[Hashing] Use a non-deterministic seed if LLVM_ENABLE_ABI_BREAKING_CHECKS](https://github.com/llvm/llvm-project/pull/96282) to improve test coverage (e.g. assertion builds) and ensure future flexibility to replace the algorithm.

The change has a noticeable code size reduction
```asm
# old
movq    _ZN4llvm7hashing6detail19fixed_seed_overrideE@GOTPCREL(%rip), %rax
movq    (%rax), %rax
testq   %rax, %rax
movabsq $-49064778989728563, %rcx       # imm = 0xFF51AFD7ED558CCD
cmoveq  %rcx, %rax

# new
movabsq $-49064778989728563, %rcx
```

... and [significant compile time improvement](https://llvm-compile-time-tracker.com/compare.php?from=982c54719289c1d85d03be3ad9e95bbfd2862aee&to=ce80c80dca45c7b4636a3e143973e2c6cbdb2884&stat=instructions:u).

I [optimized `DenseMap::{find,erase}`](https://github.com/llvm/llvm-project/pull/100517), yielding compile time improvement.

Optimizations to the bit mixer in `Hashing.h` and the `DenseMap` code have yielded significant benefits, reducing both compile time and code size. This suggests there's further potential for improvement in this area.

However, the reduced code size also highlights potential significant code size increase when considering faster unordered map implementations like [`boost::unordered_flat_map`](https://bannalia.blogspot.com/2022/11/inside-boostunorderedflatmap.html), [Abseil's Swiss Table](https://abseil.io/about/design/swisstables), and [Folly's F14](https://engineering.fb.com/2019/04/25/developer-tools/f14/).
While these libraries may offer better performance, they often come with a significant increase in code complexity and size.

Introducing a new container alongside `DenseMap` to selectively replace performance-critical instances could lead to substantial code modifications.
This approach requires careful consideration to balance potential performance gains with the additional complexity.

## NumericalStabilitySanitizer

NumericalStabilitySanitizer is a new feature for the 19.x releases.
I have made many changes on the compiler-rt part.

## Clang

Driver maintenance

* [[Driver] Ensure ToolChain::LibraryPaths is not empty for non-Darwin](https://github.com/llvm/llvm-project/pull/87866)
* `[Driver] Support -Wa,--defsym similar to -Wa,-defsym`.
* [[Driver] Don't default to -mrelax-all for non-RISCV -O0](https://github.com/llvm/llvm-project/pull/90013). See [Clang's -O0 output: branch displacement and size increase](/blog/2024-04-27-clang-o0-output-branch-displacement-and-size-increase)

Options used by the LLVM integrated assembler are currently handled in an ad-hoc way. There is deduplication with and without LTO. Eventually we might want to adopt TableGen for these `-Wa,` options.

Others:

* [[Sema] -Wpointer-bool-conversion: suppress lambda function pointer conversion diagnostic during instantiation](https://github.com/llvm/llvm-project/pull/83497)
* [[Sema] Mark alias/ifunc targets used and consider mangled names](https://github.com/llvm/llvm-project/pull/87130). This patch received a lot of scrutiny because it fixed an issue reported the XZ backdoor author(s).
* [Make diagnostic pragma override -Werror=foo and DefaultError warnings](https://github.com/llvm/llvm-project/pull/93647)
* Many patches migrated code off from potentially non-deterministic `llvm/ADT/Hashing.h`
* [[CodeGen] Set attributes on resolvers emitted after ifuncs](https://github.com/llvm/llvm-project/pull/98832)

## Code review

I reviewed a wide range of patches, including areas like ADT/Support, binary utilities, MC, lld, clangDriver, LTO, sanitizers, LoongArch, RISC-V, and new features like NumericalStabilitySanitizer and RealTimeSanitizer.

To quantify my involvement, a search for patches I commented on (`repo:llvm/llvm-project is:pr -author:MaskRay commenter:MaskRay created:>2024-01-23`) yields 780 results.

Link: [My involvement with LLVM 18](/blog/2024-02-25-my-involvement-with-llvm-18)
