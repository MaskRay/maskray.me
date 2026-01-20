---
layout: post
title: "Mapping symbols: rethinking for efficiency"
author: MaskRay
tags: [arm,assembler,linker,llvm,riscv]
---

In object files, certain code patterns embed data within instructions or transitions occur between instruction sets.
This can create hurdles for disassemblers, which might misinterpret data as code, resulting in inaccurate output.
Furthermore, code written for one instruction set could be incorrectly disassembled as another.
To address these issues, some architectures (Arm, C-SKY, NDS32, RISC-V, etc) define mapping symbols to explicitly denote state transition.
Let's explore this concept using an AArch32 code example:

<!-- more -->

```asm
.text
  adr     r0, .LJTI0_0
  vldr    d0, .LCPI0_0
  bl thumb_callee
.LBB0_1:
  nop
.LBB0_2:
  nop

.LCPI0_0:
  .long   3367254360                      @ double 1.234
  .long   1072938614

  nop

.LJTI0_0:
  .long   .LBB0_1
  .long   .LBB0_2

.thumb
.type thumb_callee, %function
thumb_callee:
  bx lr
```

**Jump tables** (`.LJTI0_0`): Jump tables can reside in either data or text sections, each with its trade-offs.
Here we see a jump table in the text section (`MachineJumpTableInfo::EK_Inline` in LLVM), allowing a single instruction to take its address.
Other architectures generally prefer to place jump tables in data sections. While avoiding data in code, RISC architectures typically require two instructions to materialize the address,
since text/data distance can be pretty large.

**Constant pool (`.LCPI0_0`)**: The `vldr` instruction loads a 16-byte floating-point literal to the SIMD&FP register.

**ISA transition**: This code blends A32 and T32 instructions (the latter used in `thumb_callee`).

In these cases, a dumb disassembler might treat data as code and try disassembling them as instructions.
Assemblers create mapping symbols to assist disassemblers. For this example, the assembled object file looks like the following:

```asm
$a:
  ...

$d:
  .long   3367254360                      @ double 1.234
  .long   1072938614

$a:
  nop

$d:
  .long   .LBB0_1
  .long   .LBB0_2
  .long   .LBB0_3

$t:
thumb_callee:
  bx lr
```

## Toolchain

Now, let's delve into how mapping symbols are managed within the toolchain.

### Disassemblers

llvm-objdump sorts symbols, including mapping symbols, relative to the current section, presenting interleaved labels and instructions.
Mapping symbols act as signals for the disassembler to switch states.

```
% llvm-objdump -d --triple=armv7 --show-all-symbols a.o

a.o:    file format elf32-littlearm

Disassembly of section .text:

00000000 <$a.0>:
       0: e28f0018      add     r0, pc, #24
       4: ed9f0b02      vldr    d0, [pc, #8]            @ 0x14
       8: ebfffffe      bl      0x8                     @ imm = #-0x8
       c: e320f000      hint    #0x0
      10: e320f000      hint    #0x0

00000014 <$d.1>:
      14: 58 39 b4 c8   .word   0xc8b43958
      18: 76 be f3 3f   .word   0x3ff3be76

0000001c <$a.2>:
      1c: e320f000      nop

00000020 <$d.3>:
      20: 0c 00 00 00   .word   0x0000000c
      24: 10 00 00 00   .word   0x00000010

00000028 <$t.4>:
00000028 <thumb_callee>:
      28: 4770          bx      lr
```

I [changed llvm-objdump 18](https://reviews.llvm.org/D156190) to not display mapping symbols as labels unless `--show-all-symbols` is specified.

### nm

Both llvm-nm and GNU nm typically conceal mapping symbols alongside `STT_FILE` and `STT_SECTION` symbols.
However, you can reveal these special symbols using the `--special-syms` option.

```
% cat a.s
foo:
  bl thumb_callee
.long 42
.thumb
thumb_callee:
  bx lr
% clang --target=arm-linux-gnueabi -c a.s
% llvm-nm a.o
00000000 t foo
00000008 t thumb_callee
% llvm-nm --special-syms a.o
00000000 t $a.0
00000004 t $d.1
00000008 t $t.2
00000000 t foo
00000008 t thumb_callee
```

GNU nm behaves similarly, but with a slight quirk.
If the default BFD target isn't AArch32, mapping symbols are displayed even without `--special-syms`.

```
% arm-linux-gnueabi-nm a.o
00000000 t foo
00000008 t thumb_callee
% nm a.o
00000000 t $a.0
00000004 t $d.1
00000008 t $t.2
00000000 t foo
00000008 t thumb_callee
```

### Symbolizers

Mapping symbols, being non-unique and lacking descriptive names, are intentionally omitted by symbolizers like addr2line and llvm-symbolizer.
Their primary role lies in guiding the disassembly process rather than providing human-readable context.

## Size problem: symbol table bloat

While mapping symbols are useful, they can significantly inflate the symbol table, particularly in 64-bit architectures (`sizeof(Elf64_Sym) == 24`) with larger programs.
This issue becomes more pronounced when using `-ffunction-sections -fdata-sections`, which generates numerous small sections.

```
% cat a.c
void f0() {}
void f1() {}
void f2() {}
int d1 = 1;
int d2 = 2;
% clang -c --target=aarch64 -ffunction-sections -fdata-sections a.c
% llvm-objdump -d --show-all-symbols a.o  # GNU objdump --show-all-symbols does no display mapping symbols

a.o:    file format elf64-littleaarch64

Disassembly of section .text.f0:

0000000000000000 <$x>:
0000000000000000 <f0>:
       0: d65f03c0      ret

Disassembly of section .text.f1:

0000000000000000 <$x>:
0000000000000000 <f1>:
       0: d65f03c0      ret

Disassembly of section .text.f2:

0000000000000000 <$x>:
0000000000000000 <f2>:
       0: d65f03c0      ret
% llvm-readelf -sX a.o

Symbol table '.symtab' contains 16 entries:
   Num:    Value          Size Type    Bind   Vis+Other Ndx(SecName) Name [+ Version Info]
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS          a.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     3 (.text.f0) .text.f0
     3: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     3 (.text.f0) $x
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT     4 (.text.f1) .text.f1
     5: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     4 (.text.f1) $x
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT     5 (.text.f2) .text.f2
     7: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     5 (.text.f2) $x
     8: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     6 (.data)  $d
     9: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     7 (.comment) $d
    10: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT     9 (.eh_frame) $d
    11: 0000000000000000     4 FUNC    GLOBAL DEFAULT     3 (.text.f0) f0
    12: 0000000000000000     4 FUNC    GLOBAL DEFAULT     4 (.text.f1) f1
    13: 0000000000000000     4 FUNC    GLOBAL DEFAULT     5 (.text.f2) f2
    14: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     6 (.data)  d1
    15: 0000000000000004     4 OBJECT  GLOBAL DEFAULT     6 (.data)  d2
```

Except the trivial cases (e.g. empty section), in both GNU assembler and LLVM integrated assemble's AArch64 ports:

* A non-text section (data, debug, etc) almost always starts with an initial `$d`.
* A text section almost always starts with an initial `$x`. ABI requires a mapping symbol at offset 0.

The behaviors ensure that each function or data symbol has a corresponding mapping symbol, while extra mapping symbols might occur in rare cases.
Thereore, the number of mapping symbols in the output symbol table usually exceeds 50%.

Most text sections have 2 or 3 symbols:

* A `STT_FUNC` symbol.
* A `STT_SECTION` symbol due to a referenced from `.eh_frame`. This symbol is absent if `-fno-asynchronous-unwind-tables`.
* A `$x` mapping symbol.

During the linking process, the linker combines input sections and eliminates `STT_SECTION` symbols.

Note: LLVM integrated assemblers used to create unique `$x.<digit>` due to an assembler limitation.
I have updated LLVM 19 to [drop `.<digit>` suffixes](https://github.com/llvm/llvm-project/pull/99836).

In LLVM's ARM port, data sections do not have mapping symbols, unless there are A32 or T32 instructions ([D30724](https://reviews.llvm.org/D30724)).

## Alternative mapping symbol scheme

I have proposed an alternaive scheme to address the size concern.

* Text sections: Assume an implicit `$x` at offset 0. Add an ending `$x` if the final data isn't instructions.
* Non-text sections: Assume an implicit `$d` at offset 0. Add an ending `$d` only if the final data isn't data directives.

This approach eliminates most mapping symbols while ensuring correct disassembly. Here is an illustrated assembler example:

```asm
.section .text.f0,"ax"
  ret
// emit $d
  .long 42
// emit $x. Without this, .text.f1 might be interpreted as data.

.section .text.f1,"ax"
  ret
```

The ending mapping symbol is to ensure the subsequent section in the linker output starts with the desired state.
The data in code case is extremely rare for AArch64 as jump tables are placed in `.rodata`.

### Impressive results

I have developed a LLVM patches to add an opt-in option `clang -Wa,-mmapsyms=implicit`.
Experiments with a Clang build using this alternative scheme have shown impressive results, eliminating over 50% of symbol table entries.

```
 .o size   |   build |
 261345384 |   a64-0 | standard
 260443728 |   a64-1 | optimizing $d
 254106784 |   a64-2 | optimizing both $x
```

```
% bloaty a64-2/bin/clang -- a64-0/bin/clang
    FILE SIZE        VM SIZE
 --------------  --------------
  -5.4% -1.13Mi  [ = ]       0    .strtab
 -50.9% -4.09Mi  [ = ]       0    .symtab
  -4.0% -5.22Mi  [ = ]       0    TOTAL
```

However, omitting a mapping symbol at offset 0 for sections with instructions is currently non-conformant.
An ABI update has been [requested](https://github.com/ARM-software/abi-aa/issues/274) to address this, though it unlikely has an update in the near term due to lack of GNU toolchain support and interoperability concern.

I'll elaborate interoperability concerns below. Note, they're unlikely to impact a majority of users.

For instance, if a text section with trailing data is assembled using the traditional behavior, the last mapping symbol will be `$d`.
When linked with another text section assembled using the new behavior (lacking an initial `$x`), disassemblers might misinterpret the start of the latter section as data.

Similarly, linker scripts that combine non-text and text sections could lead to text sections appearing in a data state.

```
SECTIONS {
  ...
  mix : { *(.data.*) *(.text.foo) }
}
```

However, many developers would classify these scenarios as error conditions.

A text section may rarely start with data directives (e.g., `-fsanitize=function`, [LLVM prefix data](https://llvm.org/docs/LangRef.html#prefix-data)).
When the linker combines two such sections, the ending `$x` of the first section and the initial `$d` of the second might have the same address.

```asm
.section .text.0, "ax"
// $d
.word 0
// $x this

.section .text.1, "ax"
// $d may have the same address
.word 0
// $x
```

In a straightforward implementation, symbols are stable-sorted by address and the last symbol at an address wins. Ideally we want `$d $x $d $x`.
If the sections are in different files, a linker that respects input order will naturally achieves this.
If they're in the same file, the assembler should output `$d $x $d $x` instead of `$d $d $x $x`.
This works if `.text.0` precedes `.text.1` in the linker output, but the other section order might be unexpected.
In the worst case where the linker's section order mismatches the assembler's section order (`--symbol-ordering-file=`, `--call-graph-profile-sort`, linker scripts), the initial data directives could be mistakenly identified as code. But the following code won't, making this an acceptable risk for certain users.

Teaching linkers to scan and insert missing mapping symbols is technically possible but inelegant and [impacts performance](https://github.com/llvm/llvm-project/pull/99718#issuecomment-2285389378).
There's a strong emphasis on the philosophy of "smart format, dumb linker," which favors keeping the format itself intelligent and minimizing the complexity of the linker.

Ultimately, the proposed alternative scheme effectively addresses symbol table bloat, but requires careful consideration for compliance and interoperability.
With this optimization enabled, the remaining symbols would primarily stem from range extension thunks, prebuilt libraries, or highly specialized assembly code.

## Mapping symbols for range extension thunks

When lld creates an [AArch64 range extension thunk](/blog/2023-03-05-linker-notes-on-aarch64#range-extension-thunks), it defines a `$x` symbol to signify the A64 state.
This symbol is only relevant when the preceding section ends with the data state, a scenario that's only possible with the traditional assembler behavior.

Given the infrequency of range extension thunks, the `$x` symbol overhead is generally tolerable.

## Peculiar alignment behavior in GNU assembler

In contrast to LLVM's integrated assembler, which restricts state transitions to instructions and data directives, GNU assembler introduces additional state transitions for alignments.
These alignments can be either implicit (arising from alignment requirements) or explicit (specified through directives).
This behavior has led to some interesting edge cases and bug fixes over time.
(See related code beside [[PATCH][GAS][AARCH64]Fix "align directive causes MAP_DATA symbol to be lost"](https://sourceware.org/pipermail/binutils/2015-March/088214.html) https://sourceware.org/bugzilla/show_bug.cgi?id=20364)

```asm
.section .foo1,"a"
// no $d
.word 0

.section .foo2,"a"
// $d
.balign 4
.word 0

.section .foo3,"a"
// $d
.word 0
// $a
nop
```

In the example, `.foo1` only contains data directives and there is no `$d`.
However, `.foo2` includes an alignment directive, triggering the creation of a `$d` symbol.
Interestingly, `.foo3` starts with data but ends with an instruction, necessitating both a `$d` and an `$a` mapping symbol.

It's worth noting that DWARF sections, typically generated by the compiler, don't include explicit alignment directives.
They behave similarly to the `.foo1` example and lack an associated `$d` mapping symbol.

## AArch32 `ld --be8`

The BE-8 mode (byte-invariant addressing big-endian mode) requires the linker to convert big-endian code to little-endian.
This is [implemented](https://reviews.llvm.org/D150870) by scanning mapping symbols. See [Linker notes on AArch32#--be8](/blog/2023-04-23-linker-notes-on-aarch32#be8) for context.

## RISC-V ISA extension

RISC-V mapping symbols are similar to AArch64, but with a notable extension:

```
$x<ISA>       | Start of a sequence of instructions with <ISA> extension.
$x<ISA>.<any>
```

The alternative scheme for optimizing symbol table size can be adapted to accommodate RISC-V's `$x<ISA>` symbols.
The approach remains the same: add an ending `$x<ISA>` only if the final data in a text section doesn't belong to the desired ISA.

The alternative scheme can be adapted to work with `$x<ISA>`: Add an ending `$x<ISA>` if the final data isn't of the desired ISA.

This adaptation works seamlessly as long as all relocatable files provided to the linker share the same baseline ISA.
However, in scenarios where the relocatable files are more heterogeneous, a crucial question arises: which state should be restored at section end?
Would the subsequent section in the linker output be compiled with different ISA extensions?

Technically, we could teach linkers to insert `$x` symbols, but scanning each input text section isn't elegant.

## Mach-O `LC_DATA_IN_CODE` load command

In contrast to ELF's symbol pair approach, Mach-O employs the `LC_DATA_IN_CODE` load command to store non-instruction ranges within code sections.
This method is remarkably compact, with each entry requiring only 8 bytes.
ELF, on the other hand, needs two symbols (`$d` and `$x`) per data region, consuming 48 bytes (in ELFCLASS64) in the symbol table.

```c
struct data_in_code_entry {
  uint32_t offset;  /* from mach_header to start of data range*/
  uint16_t length;  /* number of bytes in data range */
  uint16_t kind;    /* a DICE_KIND_* value  */
};
```

In llvm-project, the possible `kind` values are defined in `llvm/include/llvm/BinaryFormat/MachO.h`.
I recently refactored the generic `MCAssembler` to place this Mach-O specific thing, alongside others, to `MachObjectWriter`.

```c
enum DataRegionType {
  // Constants for the "kind" field in a data_in_code_entry structure
  DICE_KIND_DATA = 1u,
  DICE_KIND_JUMP_TABLE8 = 2u,
  DICE_KIND_JUMP_TABLE16 = 3u,
  DICE_KIND_JUMP_TABLE32 = 4u,
  DICE_KIND_ABS_JUMP_TABLE32 = 5u
};
```

## Achieving Mach-O's efficiency in ELF

Given ELF's symbol table bloat due to the `st_size` member ([my previous analysis](/blog/2024-01-14-exploring-object-file-formats#symbols)), how can it attain Mach-O's level of efficiency?
Instead of introducing a new format, we can leverage the standard ELF feature: [`SHF_COMPRESSED`](/blog/2023-07-07-compressed-arbitrary-sections).

Both `.symtab` and `.strtab` lack the `SHF_ALLOC` flag, making them eligible for compression without requiring any changes to the ELF specification.

* [LLVM discussion](https://discourse.llvm.org/t/rfc-compressed-sht-symtab-sht-strtab-for-elf/77608)
* A [feature request](https://sourceware.org/bugzilla/show_bug.cgi?id=31884) has already been submitted to binutils to explore this possibility.

The implementation within LLVM shouldn't be overly complex, and I'm more than willing to contribute if there's interest from the community.
