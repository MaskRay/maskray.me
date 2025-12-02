---
layout: post
title: "Stack walking: space and time trade-offs"
author: MaskRay
tags: [gcc,sframe]
---

On most Linux platforms (except AArch32, which uses `.ARM.exidx`), DWARF `.eh_frame` is required for [C++ exception handling](/blog/2020-12-12-c++-exception-handling-abi) and [stack unwinding](/blog/2020-11-08-stack-unwinding) to restore callee-saved registers.
While `.eh_frame` can be used for call trace recording, it is often criticized for its runtime overhead.
As an alternative, developers can enable frame pointers, or adopt SFrame, a newer format designed specifically for profiling.
This article examines the size overhead of enabling non-DWARF stack walking mechanisms when building several LLVM executables.

Runtime performance analysis will be added in a future update.

<!-- more -->

## Stack walking mechanisms

Here is a survey of mechanisms available for x86-64:

- Frame pointers: Fast and simple, but costs a register.
- DWARF `.eh_frame`: Comprehensive but slower, supports additional features like C++ exception handling
- SFrame: This is a new experimental format only support profiling. `.eh_frame` remains necessary for debugging and C++ exception handling. Check out [Remarks on SFrame](/blog/2025-09-28-remarks-on-sframe) for details.
- LLVM's Compact Unwinding Format: A highly space-efficient format, [implemented by Apple for Mach-O binaries](https://faultlore.com/blah/compact-unwinding/). This has llvm, lld/MachO, and libunwind implementation. Supports x86-64 and AArch64. This can mostly replace DWARF CFI, though some entries need DWARF escape (`__eh_frame` section would be tiny). OpenVMS modified it for their x86-64 port.
- x86 Last Branch Record (LBR): A hardware feature that captures a limited history of most recent branches (up to 32 on Skylake+). When configured to track branches for SamplePGO, the limited depth means it won't reliably capture deep stack traces. Traditionally Intel only, but AMD Zen 4 has since implemented [Last Branch Record Extension Version 2 (LbrExtV2)](https://lkml.kernel.org/lkml/b6bb0abaa8a54c0b6d716344700ee11a1793d709.1660211399.git.sandipan.das@amd.com/T/)
- Control-flow Enforcement Technology (CET) Shadow Stack: This hardware security hardening feature can be used to get stack traces. While it introduces some overhead, it offers the flexibility of process-specific enablement.

## Space overhead analysis

### Frame pointer size impact

For most architectures, GCC defaults to `-fomit-frame-pointer` in `-O` compilation to free up a register for general use.
To enable frame pointers, specify `-fno-omit-frame-pointer`, which reserves the frame pointer register (e.g., `rbp` on x86-64) and emits push/pop instructions in function prologues/epilogues.

For leaf functions (those that don't call other functions), while the frame pointer register should still be reserved for consistency, the push/pop operations are often unnecessary.
Compilers provide `-momit-leaf-frame-pointer` (with target-specific defaults) to reduce code size.

The viability of this optimization depends on the target architecture:

- On AArch64, the return address is available in the link register (X30). The immediate caller can be retrieved by inspecting X30, so `-momit-leaf-frame-pointer` does not compromise unwinding.
- On x86-64, after the prologue instructions execute, the return address is stored at RSP plus an offset. An unwinder needs to know the stack frame size to retrieve the return address, or it must utilize DWARF information for the leaf frame and then switch to the FP chain for parent frames.

Beyond this architectural consideration, there are additional practical reasons to use `-momit-leaf-frame-pointer` on x86-64:

- Many hand-written assembly implementations (including numerous glibc functions) don't establish frame pointers, creating gaps in the frame pointer chain anyway.
- In the prologue sequence `push rbp; mov rbp, rsp`, after the first instruction executes, RBP does not yet reference the current stack frame. When shrink-wrapping optimizations are enabled, the instruction region where RBP still holds the old value becomes larger, increasing the window where the frame pointer is unreliable.

Given these trade-offs, three common configurations have emerged:

- omitting FP: `-fomit-frame-pointer -momit-leaf-frame-pointer` (smallest overhead)
- reserving FP, but removing FP push/pop for leaf functions: `-fno-omit-frame-pointer -momit-leaf-frame-pointer` (frame pointer chain omitting the leaf frame)
- reserving FP: `-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer` (complete frame pointer chain, largest overhead)

The size impact varies significantly by program. Here's a [Ruby script `section_size.rb`](https://github.com/MaskRay/object-file-size-analyzer/blob/master/section_size.rb) that compares section sizes:

```
% ~/Dev/object-file-size-analyzer/section_size.rb /tmp/out/custom-{none,nonleaf,all}/bin/{llvm-mc,opt}
Filename                            |       .text size |        EH size |  VM size | VM increase
------------------------------------+------------------+----------------+----------+------------
/tmp/out/custom-none/bin/llvm-mc    |  2114687 (23.7%) |  367992 (4.1%) |  8914057 |           -
/tmp/out/custom-nonleaf/bin/llvm-mc |  2124143 (24.0%) |  301688 (3.4%) |  8856713 |       -0.6%
/tmp/out/custom-all/bin/llvm-mc     |  2149535 (24.0%) |  362408 (4.1%) |  8942729 |       +0.3%
/tmp/out/custom-none/bin/opt        | 39018511 (70.2%) | 4561112 (8.2%) | 55583965 |           -
/tmp/out/custom-nonleaf/bin/opt     | 38879897 (71.4%) | 3542288 (6.5%) | 54424789 |       -2.1%
/tmp/out/custom-all/bin/opt         | 38980905 (71.0%) | 3888624 (7.1%) | 54871285 |       -1.3%
```

For instance, `llvm-mc` is dominated by read-only data, making the relative `.text` percentage quite small, so frame pointer impact on the VM size is minimal.
("VM size" is a metric used by bloaty, representing the total `p_memsz` size of `PT_LOAD` segments, excluding [alignment padding](/blog/2023-12-17-exploring-the-section-layout-in-linker-output).)
As expected, `llvm-mc` grows larger as more functions set up the frame pointer chain.
However, `opt` actually becomes smaller when `-fno-omit-frame-pointer` is enabled—a counterintuitive result that warrants explanation.

Without frame pointer, the compiler uses RSP-relative addressing to access stack objects.
When using the register-indirect + disp8/disp32 addresing mode, RSP needs an extra SIB byte while RBP doesn't.
For larger functions accessing many local variables, the savings from shorter RBP-relative encodings can outweigh the additional `push rbp; mov rbp, rsp; pop rbp` instructions in the prologues/epilogues.

```
% echo 'mov rax, [rsp+8]; mov rax, [rbp-8]' | /tmp/Rel/bin/llvm-mc -x86-asm-syntax=intel -output-asm-variant=1 -show-encoding
        mov     rax, qword ptr [rsp + 8]        # encoding: [0x48,0x8b,0x44,0x24,0x08]
        mov     rax, qword ptr [rbp - 8]        # encoding: [0x48,0x8b,0x45,0xf8]

# ModR/M byte 0x44: Mod=01 (register-indirect addressing + disp8), Reg=0 (dest reg RAX), R/M=100 (SIB byte follows)
# ModR/M byte 0x45: Mod=01 (register-indirect addressing + disp8), Reg=0 (dest reg RAX), R/M=101 (RBP)
```

### SFrame vs .eh_frame

Oracle is advocating for SFrame adoption in Linux distributions.
The SFrame implementation is handled by the assembler and linker rather than the compiler.
Let's build the latest binutils-gdb to test it.

**Building test program**

We'll use the clang compiler from <https://github.com/llvm/llvm-project/tree/release/21.x> as our test program.

There are still issues related to garbage collection ([object file format design issue](/blog/2025-09-28-remarks-on-sframe#:~:text=garbage)), so I'll just disable `-Wl,--gc-sections`.

```patch
--- i/llvm/cmake/modules/AddLLVM.cmake
+++ w/llvm/cmake/modules/AddLLVM.cmake
@@ -331,4 +331,4 @@ function(add_link_opts target_name)
         # TODO Revisit this later on z/OS.
-        set_property(TARGET ${target_name} APPEND_STRING PROPERTY
-                     LINK_FLAGS " -Wl,--gc-sections")
+        #set_property(TARGET ${target_name} APPEND_STRING PROPERTY
+        #             LINK_FLAGS " -Wl,--gc-sections")
       endif()
```

```sh
configure-llvm custom-sframe -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS='clang' -DLLVM_ENABLE_UNWIND_TABLES=on -DLLVM_ENABLE_LLD=off -DCMAKE_{EXE,SHARED}_LINKER_FLAGS=-fuse-ld=bfd -DCMAKE_C_COMPILER=$HOME/opt/gcc-15/bin/gcc -DCMAKE_CXX_COMPILER=$HOME/opt/gcc-15/bin/g++ -DCMAKE_C_FLAGS="-B$HOME/opt/binutils/bin -Wa,--gsframe" -DCMAKE_CXX_FLAGS="-B$HOME/opt/binutils/bin -Wa,--gsframe"
ninja -C /tmp/out/custom-sframe clang
```

```
% ~/Dev/bloaty/out/release/bloaty /tmp/out/custom-sframe/bin/clang
    FILE SIZE        VM SIZE
 --------------  --------------
  63.9%  88.0Mi  73.9%  88.0Mi    .text
  11.1%  15.2Mi   0.0%       0    .strtab
   7.2%  9.96Mi   8.4%  9.96Mi    .rodata
   6.4%  8.87Mi   7.5%  8.87Mi    .sframe
   5.1%  7.07Mi   5.9%  7.07Mi    .eh_frame
   2.9%  3.96Mi   0.0%       0    .symtab
   1.4%  1.98Mi   1.7%  1.98Mi    .data.rel.ro
   0.9%  1.23Mi   1.0%  1.23Mi    [LOAD #4 [R]]
   0.7%   999Ki   0.8%   999Ki    .eh_frame_hdr
   0.0%       0   0.5%   614Ki    .bss
   0.2%   294Ki   0.2%   294Ki    .data
   0.0%  23.1Ki   0.0%  23.1Ki    .rela.dyn
   0.0%  8.99Ki   0.0%  8.99Ki    .dynstr
   0.0%  8.77Ki   0.0%  8.77Ki    .dynsym
   0.0%  7.24Ki   0.0%  7.24Ki    .rela.plt
   0.0%  6.73Ki   0.0%       0    [Unmapped]
   0.0%  6.29Ki   0.0%  3.84Ki    [21 Others]
   0.0%  4.84Ki   0.0%  4.84Ki    .plt
   0.0%  3.36Ki   0.0%  3.30Ki    .init_array
   0.0%  2.50Ki   0.0%  2.50Ki    .hash
   0.0%  2.44Ki   0.0%  2.44Ki    .got.plt
 100.0%   137Mi 100.0%   119Mi    TOTAL
% ~/Dev/object-file-size-analyzer/eh_size.rb /tmp/out/custom-sframe/bin/clang
clang: sframe=9303875 eh_frame=7408976 eh_frame_hdr=1023004 eh=8431980 sframe/eh_frame=1.2558 sframe/eh=1.1034
```

The results show that `.sframe` (8.87 MiB) is approximately 10% larger than the combined size of `.eh_frame` and `.eh_frame_hdr` (7.07 + 0.99 = 8.06 MiB).
While SFrame is designed for efficiency during stack walking, it carries a non-trivial space overhead compared to traditional DWARF unwind information.

### SFrame vs FP

Having examined SFrame's overhead compared to `.eh_frame`, let's now compare the two primary approaches for non-hardware-assisted stack walking.

* **Frame pointer approach**: Reserve FP but omit push/pop for leaf functions
  `g++ -fno-omit-frame-pointer -momit-leaf-frame-pointer`
* **SFrame approach**: Omit FP and use SFrame metadata
  `g++ -fomit-frame-pointer -momit-leaf-frame-pointer -Wa,--gsframe`

To conduct a fair comparison, we build LLVM executables using both approaches with both Clang and GCC compilers.
The following script configures and builds test binaries with each combination:

```sh
#!/bin/zsh
conf() {
  configure-llvm $1 -DCMAKE_EXE_LINKER_FLAGS='-fuse-ld=bfd -pie -Wl,-z,pack-relative-relocs' \
    -DCMAKE_SHARED_LINKER_FLAGS=-fuse-ld=bfd -DLLVM_ENABLE_UNWIND_TABLES=on -DLLVM_ENABLE_LLD=off ${@:2}
}

clang=(-DCMAKE_CXX_COMPILER=/tmp/Rel/bin/clang++ -DCMAKE_C_COMPILER=/tmp/Rel/bin/clang)
gcc=("-DCMAKE_C_COMPILER=$HOME/opt/gcc-15/bin/gcc" "-DCMAKE_CXX_COMPILER=$HOME/opt/gcc-15/bin/g++")

compact="-fomit-frame-pointer -momit-leaf-frame-pointer -B$HOME/opt/binutils/bin -mllvm -elf-compact-unwind -mllvm -x86-epilog-cfi=0"
fp="-fno-omit-frame-pointer -momit-leaf-frame-pointer -B$HOME/opt/binutils/bin -Wa,--gsframe=no"
sframe="-fomit-frame-pointer -momit-leaf-frame-pointer -B$HOME/opt/binutils/bin -Wa,--gsframe"

conf custom-compact -DCMAKE_{C,CXX}_FLAGS="$compact" ${clang[@]} \
  -DCMAKE_EXE_LINKER_FLAGS='-fuse-ld=lld -pie -Wl,-z,pack-relative-relocs' \
  -DCMAKE_SHARED_LINKER_FLAGS=-fuse-ld=lld

conf custom-fp -DCMAKE_{C,CXX}_FLAGS="-fno-integrated-as $fp" ${clang[@]}
conf custom-sframe -DCMAKE_{C,CXX}_FLAGS="-fno-integrated-as $sframe" ${clang[@]}

conf custom-fp-gcc -DCMAKE_{C,CXX}_FLAGS="$fp" ${gcc[@]}
conf custom-sframe-gcc -DCMAKE_{C,CXX}_FLAGS="$sframe" ${gcc[@]}

for i in compact fp sframe  fp-gcc sframe-gcc; do ninja -C /tmp/out/custom-$i llvm-mc opt; done
```

The results reveal interesting differences between compiler implementations:

```
% ~/Dev/object-file-size-analyzer/section_size.rb /tmp/out/custom-{fp,sframe,compact,fp-gcc,sframe-gcc}/bin/{llvm-mc,opt}
Filename                               |       .text size |        EH size |   .sframe size |  VM size | VM increase
---------------------------------------+------------------+----------------+----------------+----------+------------
/tmp/out/custom-fp/bin/llvm-mc         |  2120895 (23.5%) |  301528 (3.3%) |       0 (0.0%) |  9043221 |           -
/tmp/out/custom-sframe/bin/llvm-mc     |  2109231 (22.3%) |  367424 (3.9%) |  348041 (3.7%) |  9474085 |       +4.8%
/tmp/out/custom-compact/bin/llvm-mc    |  2109519 (24.4%) |  106288 (1.2%) |       0 (0.0%) |  8639637 |       -4.5%
/tmp/out/custom-fp-gcc/bin/llvm-mc     |  2744214 (29.2%) |  301836 (3.2%) |       0 (0.0%) |  9389677 |       +3.8%
/tmp/out/custom-sframe-gcc/bin/llvm-mc |  2705860 (27.7%) |  354292 (3.6%) |  356073 (3.6%) |  9780985 |       +8.2%
/tmp/out/custom-fp/bin/opt             | 38769545 (69.9%) | 3547688 (6.4%) |       0 (0.0%) | 55425217 |           -
/tmp/out/custom-sframe/bin/opt         | 38891295 (62.4%) | 4559644 (7.3%) | 4448874 (7.1%) | 62292133 |      +12.4%
/tmp/out/custom-compact/bin/opt        | 38898415 (74.8%) | 1200764 (2.3%) |       0 (0.0%) | 52020449 |       -6.1%
/tmp/out/custom-fp-gcc/bin/opt         | 54654215 (78.1%) | 3631196 (5.2%) |       0 (0.0%) | 70001373 |      +26.3%
/tmp/out/custom-sframe-gcc/bin/opt     | 53644895 (70.4%) | 4857364 (6.4%) | 5263676 (6.9%) | 76206149 |      +37.5%

% ruby ~/Dev/object-file-size-analyzer/eh_size.rb  /tmp/out/custom-compact/bin/opt
opt: sframe=0 eh_frame=267008 eh_frame_hdr=933756 eh=1200764 sframe/eh_frame=0.0 sframe/eh=0.0
% ruby ~/Dev/object-file-size-analyzer/eh_size.rb  /tmp/out/custom-sframe/bin/opt
opt: sframe=4448874 eh_frame=3938448 eh_frame_hdr=621196 eh=4559644 sframe/eh_frame=1.1296 sframe/eh=0.9757

% ~/Dev/object-file-size-analyzer/section_size.rb /tmp/out/custom-{fp-sync,sframe-sync,compact-sync}/bin/{llvm-mc,opt}
Filename                                 |       .text size |        EH size |   .sframe size |  VM size | VM increase
-----------------------------------------+------------------+----------------+----------------+----------+------------
/tmp/out/custom-fp-sync/bin/llvm-mc      |  2120895 (24.1%) |  263396 (3.0%) |       0 (0.0%) |  8802093 |           -
/tmp/out/custom-sframe-sync/bin/llvm-mc  |  2109231 (23.2%) |  291084 (3.2%) |  248654 (2.7%) |  9090325 |       +3.3%
/tmp/out/custom-compact-sync/bin/llvm-mc |  2109519 (24.4%) |  106288 (1.2%) |       0 (0.0%) |  8639637 |       -1.8%
/tmp/out/custom-fp-sync/bin/opt          | 38769545 (72.2%) | 2997572 (5.6%) |       0 (0.0%) | 53706041 |           -
/tmp/out/custom-sframe-sync/bin/opt      | 38891295 (66.9%) | 3425116 (5.9%) | 2951292 (5.1%) | 58091421 |       +8.2%
/tmp/out/custom-compact-sync/bin/opt     | 38898415 (74.8%) | 1200764 (2.3%) |       0 (0.0%) | 52020449 |       -3.1%
```

- SFrame incurs a significant VM size increase.
- GCC-built binaries are significantly larger than their Clang counterparts, probably due to more aggressive inlining or vectorization strategies.
- `/tmp/out/custom-compact` has significantly smaller EH size. See details below.

With Clang-built binaries, the frame pointer configuration produces a smaller `opt` executable (55.6 MiB) compared to the SFrame configuration (62.5 MiB).
This reinforces our earlier observation that RBP addressing can be more compact than RSP-relative addressing for large functions with frequent local variable accesses.

Assembly comparison reveals that functions using RBP and RSP addressing produce quite similar code.

In contrast, GCC-built binaries show the opposite trend: the frame pointer version of `opt` (70.0 MiB) is smaller than the SFrame version (76.2 MiB).

The generated assembly differs significantly between omit-FP and non-omit-FP builds,
I have compared symbol sizes between two GCC builds.
```
nvim -d =(/tmp/Rel/bin/llvm-nm -U --size-sort /tmp/out/custom-fp-gcc/bin/llvm-mc) =(/tmp/Rel/bin/llvm-nm -U --size-sort /tmp/out/custom-sframe-gcc/bin/llvm-mc)
```

Many functions, such as `_ZN4llvm15ELFObjectWriter24executePostLayoutBindingEv`, have significant more instructions in the keep-FP build.
This suggests that GCC's frame pointer code generation may not be as optimized as its default omit-FP path.

The `/tmp/out/custom-compact` build uses my llvm-project branch (<http://github.com/MaskRay/llvm-project/tree/demo-unwind>) that ports Mach-O compact unwind to ELF, allowing the majority of `.eh_frame` FDEs to replace CFI instructions with unwind descriptors.
Linker behavior:

- Split FDEs into two groups: descriptor-based (augmentation 'C') and instruction-based
- Generate `.eh_frame_hdr` version 2 with 12-byte table entries when compact FDEs are present: `(pc_ptr, unwind_descriptor_or_fde_ptr)`. Compact FDEs described by `.eh_frame_hdr` inline are removed from the output `.eh_frame` section.

Note: `.ARM.exidx` and [MIPS compact exception tables](https://maskray.me/blog/2020-11-08-stack-unwinding#:~:text=MIPS) also describe unwind descriptors inline in a binary search index table.

FDEs not representable by compact unwind (e.g. shrink wrapping optimization) use the traditional CFI instructions (called DWARF escape in the Mach-O compact unwind information).

This implementation involves several components:

- `-mllvm -elf-compact-unwind`: Emits `.eh_frame` CIEs with augmentation character 'C' and FDEs using unwind descriptors.
- `-mllvm -x86-epilog-cfi=0`: Disables epilogue CFI for x86 (primarily implemented by [D42848](https://reviews.llvm.org/D42848) in 2018, notably disabled for Darwin and Windows).
  Without this option most frames will not utilize unwind descriptors because the current Mach-O compact unwind implementation does not support `popq %rbp; .cfi_def_cfa %rsp, 8; ret`.
  I believe this is still fair as we expect to use a 8-byte descriptor, sufficient to describe epilogue CFI.
- lld/ELF changes: FDEs are split into descriptor-based (augmentation 'C') and CFI-instruction-based groups. When compact FDEs are present, `.eh_frame_hdr` version 2 is generated with 12-byte table entries containing (pc_ptr, unwind_descriptor_or_fde_ptr). The PC pointer remains 4 bytes, while the 8-byte entry indicates either an unwind descriptor (odd value) or an FDE pointer (even value).

With the current implementation, 4937 out of 77648 FDEs (6.36%) require a DWARF escape, while the remaining FDEs can be replaced with unwind descriptors.

`.eh_frame_hdr` will become even smaller if we implement the two-level page table structure in Mach-O `__unwind_info`.

## Runtime performance analysis

TODO

perf record overhead with EH

perf record overhead with FP

Here is a [benchmark run from llvm-compile-time-tracker.com](https://llvm-compile-time-tracker.com/compare.php?from=5d0f1591f8b91ac7919910c4e3e9614a8804c02a&to=76cdaf78a7d4b06130031818397da11b8985ab08&stat=instructions:u).

The `stable2-O3` benchmark is relevant. When enabling FP for non-leaf functions, the `instructions:u` metric increases by +2.44% while `wall-time` (a noisy metric) increases by just 0.56%.

## Summary

This article examines the space overhead of different stack walking mechanisms when building LLVM executables.

**Frame pointer configurations:** Enabling frame pointers (`-fno-omit-frame-pointer`) can paradoxically reduce x86-64 binary size when stack object accesses are frequent.
This occurs because RBP-relative addressing produces more compact encodings than RSP-relative addressing, which requires an extra SIB byte.
The savings from shorter instructions can outweigh the prologue/epilogue overhead.

**SFrame vs .eh_frame:** For the x86-64 `clang` executable, SFrame metadata is approximately 10% larger than the combined size of `.eh_frame` and `.eh_frame_hdr`.
Given the significant VM size overhead and the lack of clear advantages over established alternatives, I am skeptical about SFrame's viability as the future of stack walking for userspace programs.
While SFrame will receive a major revision V3 in the upcoming months, it needs to achieve substantial size reductions comparable to existing compact unwinding schemes to justify its adoption over frame pointers.
I hope interested folks can implement something similar to macOS's compact unwind descriptors (with x86-64 support) and OpenVMS's.

**ELF compact unwind**: My prototype porting Mach-O compact unwind to ELF demonstrates significant promise.
The approach reduces VM size by 4.5-6.1% compared to frame pointers, achieving the smallest binaries in my benchmarks.
By replacing verbose CFI instructions with 8-byte unwind descriptors (with DWARF escape for complex cases like shrink-wrapped functions), `.eh_frame` shrinks dramatically—only 6.36% of FDEs require the traditional CFI format.
This approach, once completed, offers a compelling alternative to SFrame: better compression, compatibility with existing `.eh_frame` infrastructure, and a clear path to implementation.

**[LLVM community: I need your support](https://discourse.llvm.org/t/rfc-adding-sframe-support-to-llvm/86900/34?u=maskray)**. I've raised technical objections to the SFrame RFC as maintainer. Some engineers dismissed them. Now they're escalating to Project Council to override technical review.
This looks OKR-driven, not merit-driven.

GCC's frame pointer code generation appears less optimized than its default omit-frame-pointer path, as evidenced by substantial differences in generated assembly.

Runtime performance analysis remains to be conducted to complete the trade-off evaluation.

## Appendix: `configure-llvm`

This script specifies common options when configuring llvm-project:
<https://github.com/MaskRay/Config/blob/master/home/bin/configure-llvm>

- `-DCMAKE_CXX_ARCHIVE_CREATE="$HOME/Stable/bin/llvm-ar qc --thin <TARGET> <OBJECTS>" -DCMAKE_CXX_ARCHIVE_FINISH=:`: Use thin archives to reduce disk usage
- `-DLLVM_TARGETS_TO_BUILD=host`: Build a single target
- `-DCLANG_ENABLE_OBJC_REWRITER=off -DCLANG_ENABLE_STATIC_ANALYZER=off`: Disable less popular components
- `-DLLVM_ENABLE_PLUGINS=off -DCLANG_PLUGIN_SUPPORT=off`: Disable `-Wl,--export-dynamic`, preventing large `.dynsym` and `.dynstr` sections

## Appendix: My SFrame build

```sh
mkdir -p out/release && cd out/release
../../configure --prefix=$HOME/opt/binutils --disable-multilib
make -j $(nproc) all-ld all-binutils all-gas
make -j $(nproc) install-ld install-binutils install-gas
```

`gcc -B$HOME/opt/binutils/bin` and `clang -B$HOME/opt/binutils/bin -fno-integrated-as` will use `as` and `ld` from the install directory.

## Appendix: Scripts

Ruby scripts used by this post are available at <https://github.com/MaskRay/object-file-size-analyzer/>
