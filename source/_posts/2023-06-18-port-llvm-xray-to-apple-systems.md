---
layout: post
title: Port LLVM XRay to Apple systems
author: MaskRay
tags: [llvm]
---

I do not use Apple products myself, but I sometimes delve into Mach-O due to my interest in object file formats.
Additionally, my LLVM/Clang changes sometimes require some understanding of Mach-O. Occasionally, I need to understand the format to some extent to work around its quirks (the old format inherited many problems of "a.out").

Recently, there has been interest (from Oleksii Lozovskyi) in enabling [XRay](https://llvm.org/docs/XRay.html), a function call tracing system in LLVM, to work on Apple systems.
Intrigued by this, I decided to delve into the details and investigate the necessary changes.
XRay supports many 64-bit architectures on Linux and some BSDs.
I became acquainted with XRay back in 2017 and made some casual contributions since then.

<!-- more -->

If the target triple is `x86_64-apple-darwin`, you may notice that Clang will allow you to perform compilation, but linking will fail.

```
% clang --target=x86_64-apple-darwin -fxray-instrument -fxray-instruction-threshold=1 -c a.c
% clang --target=x86_64-apple-darwin -fxray-instrument -fxray-instruction-threshold=1 a.o
ld: in section __DATA,xray_instr_map reloc 0: X86_64_RELOC_SUBTRACTOR must have r_extern=1 file 'a.o' for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

For `arm64-apple-darwin`, Clang rejected `-fxray-instrument`, but we can bypass this driver detection with `-Xclang`.
(This is now [accepted](https://reviews.llvm.org/D145849).)
```
% clang --target=arm64-apple-darwin -fxray-instrument -fxray-instruction-threshold=1 -c a.c
clang: error: the clang compiler does not support '-fxray-instrument on aarch64-apple-darwin'
```
```
% clang --target=arm64-apple-darwin -Xclang -fxray-instrument -Xclang -fxray-instruction-threshold=1 -c a.c
fatal error: error in backend: unsupported relocation of local symbol ''. Must have non-local symbol earlier in section.
```

I noticed that this fatal error is caused by an old workaround (2015) for ld64 and proposed to remove it: <https://reviews.llvm.org/D152831>.

## SUBTRACTOR relocation types require `r_extern==1`

Let's examine `clang -S --target=x86_64-apple-darwin -fxray-instrument -fxray-instruction-threshold=1` generated assembly.
```asm
        .section        __DATA,xray_instr_map
Lxray_sleds_start0:
Ltmp0:
        .quad   Lxray_sled_0-Ltmp0      // problematic
        .quad   Lfunc_begin0-(Ltmp0+8)  // problematic
        .byte   0x00
        .byte   0x00
        .byte   0x02
        .space  13
Ltmp1:
        .quad   Lxray_sled_1-Ltmp1      // problematic
        .quad   Lfunc_begin0-(Ltmp1+8)  // problematic
        .byte   0x01
        .byte   0x00
        .byte   0x02
        .space  13
Lxray_sleds_end0:
```

I have marked 4 label differences as problematic. Let's examine the first one, `.quad Lxray_sled_0-Ltmp0`, which is represented as a pair of relocations (`llvm-readobj -r a.o`)
```
    0x0 0 3 0 X86_64_RELOC_SUBTRACTOR 0 xray_instr_map
    0x0 0 3 1 X86_64_RELOC_UNSIGNED 0 _foo
```

`X86_64_RELOC_SUBTRACTOR` is an external relocation (`r_extern==1`) where `r_symbolnum` references a symbol table entry. If `r_extern==0`, Linkers will produce an error.
Symbols prefixed with "L" are considered temporary symbols in LLVMMC and are not included in the symbol table.
LLVM integrated assembler attempts to rewrite `A-B` into `A-B'+offset` where `B'` can be included in the symbol table.
B' is called an atom and should be a non-temporary symbol in the same section.
However, since `xray_instr_map` does not define a non-temporary symbol, the `X86_64_RELOC_SUBTRACTOR` relocation will have no associated symbol, and its `r_extern` value will be 0.

To fix this issue, we need to define a non-temporary symbol in the section. We can accomplish this by renaming `Lxray_sleds_start0` to `lxray_sleds_start0` ("L" to "l").
In LLVMMC, `LinkerPrivateGlobalPrefix` is set to "l" for Apple targets.
We can define an overload of `MCContext::createLinkerPrivateTempSymbol(const Twine &Name)` to allow LLVMMC to select an unused symbol starting with `lxray_sleds_start`. (There is a pitfall: we should not use "ltmp[0-9]+" which may collide with other compiler internal symbols.)
For ELF targets, the `MCContext::createLinkerPrivateTempSymbol` function creates a temporary symbol starting with ".L".

Note: GNU assembler calls `.L` and `L` system-specific local label prefixes.

## Function index section

Oleksii Lozovskyi reported that the `-fxray-function-index` option was not functioning as expected.

* (default): no function index section
* `-fxray-function-index`: no function index section
* `-fno-xray-function-index`: function index section (`xray_fn_idx`) is present

Initially, the function index section was always present.
The introduction of the `-f[no-]xray-function-index` option caused confusion due to the use of a negatively named variable, leading to potential misunderstandings.
A [clangDriver refactoring](https://github.com/llvm/llvm-project/commit/d8a8e5d6240a1db809cd95106910358e69bbf299) accidentally introduced this regression.
I have [resolved the issue](https://github.com/llvm/llvm-project/commit/63bd6d9e644335c8138a59281aafbf65a82fc47a) for Clang 17.

Now that `-fxray-function-index` is back, we get the `xray_fn_idx` section by default. The section contains entries like the following:
```asm
        .section        __DATA,xray_fn_idx
        .p2align        4, 0x90
        .quad   Lxray_sleds_start0
        .quad   Lxray_sleds_end0
```

These absolute addresses require rebase opcodes in the special section `__LINKEDIT,__rebase`.
This is not great and I [wanted to fix it](https://github.com/llvm/llvm-project/commit/c5d38924dc6688c15b3fa133abeb3626e8f0767c) back in 2020 but never got around to do it.
This motivated me to actually fix the issue and create <https://reviews.llvm.org/D152661> to change the `[start,end)` representation to the `(pc_relative_start, size)` representation.

My initial attempt somehow wrote something like this. I took a difference of two labels, and right shifted it by 5 to get the number of sleds.
```asm
        .section        __DATA,xray_fn_idx,regular,live_support
        .p2align        4, 0x90
lxray_fn_idx0:
        .quad   lxray_sleds_start0-lxray_fn_idx0
        .quad   (Lxray_sleds_end0-lxray_sleds_start0)>>5
```

This approach works on ELF targets but not on Mach-O targets due to a pile of assembler issues.

```
% clang -c --target=x86_64-apple-darwin a.s
/tmp/a-b0645c.s:53:8: error: expected relocatable expression
 .quad (Lxray_sleds_end0-lxray_sleds_start0)>>5
       ^
```

## Assembler issues

When assembling an assembly file into an object file, an expression can be evaluated in multiple steps. Two steps are particularly important:

* Parsing time. At this stage, We have a `MCAssembler` object but no `MCAsmLayout` object. Instruction operands and certain directives like `.if` require the ability to evaluate an expression early.
* Object file writing time. At this stage, we have both a `MCAssembler` object and a `MCAsmLayout` object. The `MCAsmLayout` object provides information about the offset of each fragment.

The first issue is not specific to this case and is also encountered in ELF. The following assembly code should assemble to the hex pairs 01000001, but Clang fails to compute `.if .-1b == 3`.
```
% cat x.s
1:
.byte 1
.space 2
.if .-1b == 3
  .byte 1
.else
  .byte 0
.endif
% clang -c x.s
x.s:4:5: error: expected absolute expression
.if .-1b == 3
    ^
```

In 2019, Jian Cai [implemented](https://reviews.llvm.org/D69411) limited expression folding support to LLVM integrated assembler to support the Linux kernel arm use case.
```
arch/arm/mm/proc-v7.S:169:143: error: expected absolute expression
.pushsection ".alt.smp.init", "a" ; .long 9998b ;9997: orr r1, r1, #((1 << 0) | (1 << 6))|(3 << 3) ; .if . - 9997b == 2 ; nop ; .endif ; .if . - 9997b != 4 ; .error "ALT_UP() content must assemble to exactly 4 bytes"; .endif ; .popsection
```

I have just added support for `MCFillFragment` (`.space` and `.fill`) and for A-B, where A is a pending label (which will be reassigned to a real fragment in `flushPendingLabels()`).
Latest LLVM integrated assembler (milestone: LLVM 17) can successfully assemble `x.s` when a `MCAssembler` object is present. However, evaluation still does not work without a `MCAssembler` object, which is expected.
```
% llvm-mc x.s -filetype=null
x.s:4:5: error: expected absolute expression
.if .-1b == 3
    ^
```

Then I noticed a potential pitfall for `Mach-O` in `MCSection::flushPendingLabels`. When flushing pending labels, it did not ensure that the new fragment inherits the previous atom symbol.
I fixed this issue, although I haven't been able to create a test case to verify this behavior.

After this fix, `.quad   (Lxray_sleds_end0-lxray_sleds_start0)>>5` can be successfully assembled.
However, during ["direct object emission"](/blog/2023-05-08-assemblers#:~:text=direct%20object%20emission), an error `expected relocatable expression` will be reported.
The issue is quite subtle.

In the case of direct object emission, where LLVM IR is directly lowered to an object file bypassing assembly (e.g., using `clang -c a.c` instead of `clang -c a.s` or `clang -c --save-temps a.c`), the assembler information is not used at parse time (`MCStreamer::UseAssemblerInfoForParsing`).
As a result, the assembly code `.quad (Lxray_sleds_end0-lxray_sleds_start0)>>5` will be transformed into a fixup.
`Lxray_sleds_end0` is a pending label which will then be assigned to the next fragment when AsmPrinter emits new piece of data to the `xray_instr_map` section.

During object writing time, we have a `MCAsmLayout` object and atom information for fragments.
LLVMMC evaluates the fixup with the `MCAsmLayout` object. However, since `lxray_sleds_start0` and `Lxray_sleds_end0` have different atoms, causing the condition in `MachObjectWriter::isSymbolRefDifferenceFullyResolvedImpl` to fail.
In my opinion, it may be necessary to relax the condition in this case.

## Linker dead stripping

Both `xray_instr_map` and `xray_fn_idx` sections contain independent pieces.
To support linker dead stripping, also known as [linker garbage collection](/blog/2021-02-28-linker-garbage-collection), we need to add the `S_ATTR_LIVE_SUPPORT` attribute to the two sections.

## Runtime library issues

The runtime library is located at [`compiler-rt/lib/xray`](https://github.com/llvm/llvm-project/tree/main/compiler-rt/lib/xray) and there are a number of issues.

First, `compiler-rt/lib/xray/xray_trampoline_x86_64.S` used `.Ltmp*` symbols which are temporary for ELF but non-temporary for Mach-O.
The non-temporary labels become atoms and can cause bad dead stripping behaviors.
I fixed the problem by using the `LOCAL_LABEL` macro, which generates an "L" symbol specifically for Mach-O.

`compiler-rt/lib/xray/xray_trampoline_AArch64.S` had a similar problem. I fixed it to look like the following:
```asm
#include "../sanitizer_common/sanitizer_asm.h"
...
.p2align 2
.global ASM_SYMBOL(__xray_FunctionEntry)
ASM_HIDDEN(__xray_FunctionEntry)
ASM_TYPE_FUNCTION(__xray_FunctionEntry)
ASM_SYMBOL(__xray_FunctionEntry):
```

For Apple targets, `# define ASM_SYMBOL(symbol) _##symbol` prepends an underscore to the symbol name. For ELF targets, `ASM_SYMBOL` is the identity function.

Compiling `compiler-rt/lib/xray/xray_trampoline_AArch64.S` will give us an external definition (`N_SECT | N_EXT`) `___xray_FunctionEntry`.
The undefined reference from C++ code gives an undefined symbol `___xray_FunctionEntry` (`N_UNDF | N_EXT`), which can be resolved by the definition.
```cpp
    Success = patchFunctionEntry(Enable, FuncId, Sled, __xray_FunctionEntry); // extern "C" void __xray_FunctionEntry();
```

Second, we need <https://reviews.llvm.org/D153221> to build Mach-O runtimes for multiple architectures.

After the build system, we may discover new issues.

## Driver change

Normally, we want to enable a Clang driver option when a feature is fully ready.
In this case, we know the code generation has been working for a long time, and the full support (metadata section refactoring and runtime) will be implemented soon.
Therefore, I believe it's acceptable to make the driver accept `--target=arm64-apple-darwin -fxray-instrument` as it makes testing the runtime more convenient.

## Epilogue

As the work to port LLVM XRay to Apple systems continues, it is encouraging to witness the progress achieved thus far.
The endeavor has given me opportunities to explore the Mach-O object file format.

There is a [pending patch](https://reviews.llvm.org/D117929) for ELF RISC-V support.
