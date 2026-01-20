---
layout: post
title: lld 17 ELF changes
author: MaskRay
tags: [linker,llvm]
---

LLVM 17 will be released.
As usual, I maintain lld/ELF and have added some notes to <https://github.com/llvm/llvm-project/blob/release/17.x/lld/docs/ReleaseNotes.rst>.
Here I will elaborate on some changes.

<!-- more -->

* When `--threads=` is not specified, the number of concurrency is now capped to 16.
  A large `--thread=` can harm performance, especially with some system
  malloc implementations like glibc's.
  ([D147493](https://reviews.llvm.org/D147493))
* `--remap-inputs=` and `--remap-inputs-file=` are added to remap input files.
  ([D148859](https://reviews.llvm.org/D148859))
* `--lto=` is now available to support `clang -funified-lto`
  ([D123805](https://reviews.llvm.org/D123805))
* `--lto-CGO[0-3]` is now available to control `CodeGenOpt::Level` independent of the LTO optimization level.
  ([D141970](https://reviews.llvm.org/D141970))
* `--check-dynamic-relocations=` is now correct 32-bit targets when the addend is larger than 0x80000000.
  ([D149347](https://reviews.llvm.org/D149347))
* `--print-memory-usage` has been implemented for memory regions.
  ([D150644](https://reviews.llvm.org/D150644))
* `SHF_MERGE`, `--icf=`, and `--build-id=fast` have switched to 64-bit xxh3.
  ([D154813](https://reviews.llvm.org/D154813))
* Quoted output section names can now be used in linker scripts.
  (`#60496 <https://github.com/llvm/llvm-project/issues/60496>`_)
* `MEMORY` can now be used without a `SECTIONS` command.
  ([D145132](https://reviews.llvm.org/D145132))
* `REVERSE` can now be used in input section descriptions to reverse the order of input sections.
  ([D145381](https://reviews.llvm.org/D145381))
* Program header assignment can now be used within `OVERLAY`. This functionality was accidentally lost in 2020.
  ([D150445](https://reviews.llvm.org/D150445))
* Operators `^` and `^=` can now be used in linker scripts.
* LoongArch is now supported.
* `DT_AARCH64_MEMTAG_*` dynamic tags are now supported.
  ([D143769](https://reviews.llvm.org/D143769))
* AArch32 port now supports BE-8 and BE-32 modes for big-endian.
  ([D140201](https://reviews.llvm.org/D140201))
  ([D140202](https://reviews.llvm.org/D140202))
  ([D150870](https://reviews.llvm.org/D150870))
* `R_ARM_THM_ALU_ABS_G*` relocations are now supported.
  ([D153407](https://reviews.llvm.org/D153407))
* `.ARM.exidx` sections may start at non-zero output section offset.
  ([D148033](https://reviews.llvm.org/D148033))
* Arm Cortex-M Security Extensions is now implemented.
  ([D139092](https://reviews.llvm.org/D139092))
* BTI landing pads are now added to PLT entries accessed by range extension thunks or relative vtables.
  ([D148704](https://reviews.llvm.org/D148704))
  ([D153264](https://reviews.llvm.org/D153264))
* AArch64 short range thunk has been implemented to mitigate the performance loss of a long range thunk.
  ([D148701](https://reviews.llvm.org/D148701))
* `R_AVR_8_LO8/R_AVR_8_HI8/R_AVR_8_HLO8/R_AVR_LO8_LDI_GS/R_AVR_HI8_LDI_GS` have been implemented.
  ([D147100](https://reviews.llvm.org/D147100))
  ([D147364](https://reviews.llvm.org/D147364))
* `--no-power10-stubs` now works for PowerPC64.
* `DT_PPC64_OPT` is now supported.
  ([D150631](https://reviews.llvm.org/D150631))
* `PT_RISCV_ATTRIBUTES` is added to include the SHT_RISCV_ATTRIBUTES section.
  ([D152065](https://reviews.llvm.org/D152065))
* `R_RISCV_PLT32` is added to support C++ relative vtables.
  ([D143115](https://reviews.llvm.org/D143115))
* RISC-V global pointer relaxation has been implemented. Specify `--relax-gp` to enable the linker relaxation.
  ([D143673](https://reviews.llvm.org/D143673))
* The symbol value of `foo` is correctly handled when `--wrap=foo` and RISC-V linker relaxation are used.
  ([D151768](https://reviews.llvm.org/D151768))
* x86-64 large data sections are now placed away from code sections to alleviate relocation overflow pressure.
  ([D150510](https://reviews.llvm.org/D150510))

When using glibc malloc with a larger `std::thread::hardware_concurrency` (say, more than 16), parallel relocation scanning can be quite slower without the `--threads=16` throttling.

I usually try to make extensions, unless too LLVM internal specific (e.g. `--lto-*`), accepted by the binutils community.
The [feature request](https://sourceware.org/PR30374) for `--remap-inputs=` and `--remap-inputs-file=` was a success story, implemented by GNU ld 2.41.

`PT_RISCV_ATTRIBUTES` output is still not quite right. I also question about its usefulness.
Unfortunately, at this stage, it's difficult to [get rid of it](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/386).

This cycle has a surprising number of new features, and I have spent lots of spare time reviewing them to ensure that they are robust and properly tested.
Most stuff is completely unrelated to my day job.

There are quite a few AArch32 changes from Arm engineers, primarily about big-endian support and Cortex-M Security Extensions.

I was firm that the RISC-V global pointer relaxation needs to be opt-in.
I had a GNU ld `--relax-gp` patch last year and utilitized this opportunity (ld.lld feature proposal) to move forward GNU ld `--relax-gp`.
It's unfortunately opt-out, but having an option is a step forward.

This release adds support for LoongArch, which is a relatively new architecture that took inspiration from Mips and RISC-V.

## Speed

Unlike previous versions, there is just a minor performance improvement compared with lld 15.0.0.
I added a simplified version of 64-bit xxh3 into the `LLVMSupport` library and [utilized it in lld](https://github.com/llvm/llvm-project/issues/63750).

Linking a `-DCMAKE_BUILD_TYPE=Debug` build of clang 16:
```
% hyperfine --warmup 2 --min-runs 25 "numactl -C 20-27 "{/tmp/out/custom-16/bin/ld.lld,/tmp/out/custom-17/bin/ld.lld}" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/out/custom-16/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.159 s ±  0.035 s    [User: 7.089 s, System: 3.076 s]
  Range (min … max):    3.095 s …  3.250 s    25 runs

Benchmark 2: numactl -C 20-27 /tmp/out/custom-17/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.131 s ±  0.027 s    [User: 6.851 s, System: 3.101 s]
  Range (min … max):    3.080 s …  3.198 s    25 runs

Summary
  'numactl -C 20-27 /tmp/out/custom-17/bin/ld.lld @response.txt --threads=8' ran
    1.01 ± 0.01 times faster than 'numactl -C 20-27 /tmp/out/custom-16/bin/ld.lld @response.txt --threads=8'
```

This influence to the total link time is small.
However, if I test the time proportion of the hash function in the total link time, I can see that the proportion has been reduced to nearly one third.
On some workload and some machines this effect may be larger.

Link: [lld 16 ELF changes](/blog/2023-03-19-lld-16-elf-changes)
