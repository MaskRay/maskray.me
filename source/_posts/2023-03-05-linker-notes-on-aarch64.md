---
layout: post
title: Linker notes on AArch64
author: MaskRay
tags: [linker,llvm]
---

This article describes target-specific details about AArch64 in ELF linkers.
AArch64 is the 64-bit execution state for the Arm architecture.
The AArch64 execution state runs the A64 instruction set.
The AArch32 and AArch64 execution states use very different instruction sets, so many pieces of software use two ports for the two execution states of the Arm architecture.

There were the "ARM architecture" and the "ARM instruction set", leading to many software projects using "ARM" or "arm" as their port names.
In 2011, ARMv8 introduced two execution states, AArch32 and AArch64. The previous instruction sets "ARM" and "Thumb" were renamed to "A32" and "T32", respectively.
In 2017, the architecture was renamed to the "Arm architecture" to reflect the rebranding of the company name.
So, the "ARMv8-A" architecture profile is now named "Armv8-A".

For the AArch64 execution state, while many projects use "AArch64" as their port name, for legacy reasons, macOS, Windows, the Linux kernel, and some BSD operating systems unfortunately use "arm64".
(Support for AArch64 was added to the Linux kernel in version 3.7. Initially, the patch set was named "aarch64", but it was later changed at the [request](https://lkml.org/lkml/2012/7/6/624) of kernel developers.)

<!-- more -->

## ABI documents

* [ELF for the Arm® 64-bit Architecture (AArch64)](https://github.com/ARM-software/abi-aa/blob/main/aaelf64/aaelf64.rst)
* [System V ABI for the Arm® 64-bit Architecture (AArch64)](https://github.com/ARM-software/abi-aa/blob/main/sysvabi64/sysvabi64.rst)

## Global Offset Table

The Global Offset Table consists of two sections:

* `.got.plt` holds code addresses for PLT.
* `.got` holds other addresses and offsets.

The symbol `_GLOBAL_OFFSET_TABLE_` is defined at the beginning of the `.got` section.
GNU ld reserves a single entry for `.got` and `.got[0]` holds the link-time address of `_DYNAMIC` for a legacy reason
Versions of glibc prior to 2.35 have the `_DYNAMIC` requirement. See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table#global_offset_table_0).

`.got.plt[1]` and `.got.plt[2]` are for lazy binding PLT.
Linkers communicate the address of `.got.plt` to rtld with the dynamic tag `DT_PLTGOT`.

## Procedure Linkage Table

The registers `x16` (IP0) and `x17` (IP1) are the first and second intra-procedure-call temporary registers.
They may be used by PLT entries and veneers.

The PLT header looks like:
```asm
bti  c       // If BTI
stp  x16, x30, [sp,#-16]!
adrp x16, &.got.plt[2]
ldr  x17, [x16, :lo12: &.got.plt[2]]
add  x16, x16, :lo12: &.got.plt[2]
br   x17
```

The Nth PLT entry looks like:
```asm
bti  c       // If BTI
adrp x16, &.got.plt[N + 3]
ldr  x17, [x16, :lo12: &.got.plt[N + 3]]
add  x16, x16, :lo12: &.got.plt[N + 3]
autia1716    // If PAC-PLT
br   x17
```

When BTI is enabled for the output file, the code sequence starts with `bti c`.
When PAC-PLT is enabled, the code sequence includes `autia1716` before `br x17`.

`x16` is used by the lazy PLT resolver. For `ld -z now`, setting `x16` is unneeded.
<https://github.com/ARM-software/abi-aa/issues/202> discusses making `x16` setting optional.

## Relocation optimization

See [All about Global Offset Table#GOT optimization](/blog/2021-08-29-all-about-global-offset-table#got-optimization) for GOT optimization.

There are a few optimization schemes beside GOT optimization, e.g.

```asm
add  x2, x2, 0  // R_<CLS>_ADD_ABS_LO12_NC

=>

nop
```

```asm
adrp x0, symbol
add  x0, x0, :lo12: symbol

=>

nop
adr  x0, symbol
```

`--no-relax` disables the optimization.

See [ELF for the Arm® 64-bit Architecture (AArch64)#Relocation optimization](https://github.com/ARM-software/abi-aa/blob/main/aaelf64/aaelf64.rst#relocation-optimization).

## Thread Local Storage

AArch64 uses a variant of TLS Variant I: the static TLS blocks are placed above the thread pointer. The thread pointer points to the end of the thread control block.

The linker performs TLS optimization.

The traditional general dynamic and local dynamic TLS models are obsoleted and not supported by ld.lld

See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage).

## Program Property

A `.note.gnu.property` section contains program property notes that describe special handling requirements for the linker and the dynamic loader.

The linker parses input `.note.gnu.property` sections and recognizes command line options `-z force-bti` and `-z pac-plt` to compute the output `.note.gnu.property` (type is `SHT_NOTE`) section.
Without these options, linkers only set the feature bit in the output file if all the input relocatable object files have the corresponding feature set.

```cpp
  for (ELFFileBase *f : ctx.objectFiles) {
    uint32_t features = f->andFeatures;
    if (!(features & GNU_PROPERTY_AARCH64_FEATURE_1_BTI)) {
      if (config->zBtiReport == "error")
        error(toString(f) + ": -z bti-report: file does not have GNU_PROPERTY_AARCH64_FEATURE_1_BTI property");
      else if (config->zBtiReport == "warning")
        warn(toString(f) + ": -z bti-report: file does not have GNU_PROPERTY_AARCH64_FEATURE_1_BTI property");
    }

    if (config->zForceBti && !(features & GNU_PROPERTY_AARCH64_FEATURE_1_BTI)) {
      if (config->zBtiReport == "none")
        warn(toString(f) + ": -z force-bti: file does not have "
                           "GNU_PROPERTY_AARCH64_FEATURE_1_BTI property");
      features |= GNU_PROPERTY_AARCH64_FEATURE_1_BTI;
    }
    if (config->zPacPlt && !(features & GNU_PROPERTY_AARCH64_FEATURE_1_PAC)) {
      warn(toString(f) + ": -z pac-plt: file does not have "
                         "GNU_PROPERTY_AARCH64_FEATURE_1_PAC property");
      features |= GNU_PROPERTY_AARCH64_FEATURE_1_PAC;
    }
    ret &= features;
  }
```

## Range extension thunks

Function calls typically use `B` and `BL` instructions.
The two instructions have a range of +/-128MiB and may use 2 relocation types: `R_AARCH64_CALL26` and `R_AARCH64_JUMP26`.
The range is larger than the branch range for many other instruction sets.
If the destination is not reachable by a single `B`/`BL`, linkers may insert a veneer (range extension thunk).

`-no-pie` links may use a thunk with absolute addressing targeting any location in the 64-bit address space.
```asm
<caller>:
  bl      __AArch64AbsLongThunk_nonpreemptible
  b       __AArch64AbsLongThunk_nonpreemptible

<__AArch64AbsLongThunk_nonpreemptible>:
  ldr     x16, .+8
  br      x16

<$d>:
  .word   0x00000000
  .word   0x00000010

<.plt>:
```

`-pie` and `-shared` links need to use a thunk with PC-relative addressing targeting a range of +/-4GiB.
```asm
<caller>:
  bl      __AArch64ADRPThunk_nonpreemptible
  b       __AArch64ADRPThunk_nonpreemptible

<__AArch64ADRPThunk_nonpreemptible>:
  adrp    x16, nonpreemptible
  add     x16, x16, :lo12: nonpreemptible
  br      x16
```

The branch target of a thunk may be a PLT entry:
```asm
<caller>:
  bl      __AArch64ADRPThunk_preemptible

<__AArch64ADRPThunk_preemptible>:
  adrp    x16, preemptible@plt
  add     x16, x16, :lo12: preemptible@plt
  br      x16

...

<preemptible@plt>:
  adrp    x16, &.got.plt[N + 3]
  ldr     x17, [x16, :lo12: &.got.plt[N + 3]]
  add     x16, x16, :lo12: &.got.plt[N + 3]
  br      x17
```

In many situations we don't need the long code sequence (`ldr/br` or `adrp/add/br`). Instead, the thunk ([short range thunk](https://reviews.llvm.org/D148701)) can use one single `b/bl` to reach the destination.

### BTI thunks

Long range thunks use indirect branches (`br x16`).
When BTI is enabled, indirect branches must land on a BTI-compatible instruction (`bti c`, `bti j`, `bti jc`, `paciasp`, or `pacibsp`).
GCC may elide the BTI instruction at function entry when it can prove there are no indirect branches from outside the translation unit.
A static linker is responsible for generating a landing pad when the target does not have a BTI.

When the output sets `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` and a long range thunk targets a function without a BTI-compatible instruction, ld.lld ([since 2024-10](https://github.com/llvm/llvm-project/commit/c4d9cd8b747cb399a61dd987eb95ad518eb15448)) generates a landing pad thunk (`AArch64BTILandingPadThunk`) near the destination:
```asm
<__AArch64BTIThunk_fn>:
  bti  c
  b    fn       // Can be elided if the landing pad immediately precedes fn
<fn>:
  ...
```

The `b` instruction is elided when the landing pad can be placed so that control flow falls through to the destination, which is common with `-ffunction-sections`.

Short range thunks use direct branches (`b`/`bl`) and do not need landing pads.
A destination needs at most one landing pad, shared across all callers.
PLT entries already start with `bti c`, so they do not need landing pads either.

## `--fix-cortex-a53-843419`

This option enables a linker workaround for Arm Cortex-A53 Errata 843419.
Full details are available in the ARM-EPM-048406 document.
Linkers scan `adrp` in the last two instructions of a 4KiB page, followed by a load or store instruction and two other instructions.
Oncea erratum condition is detected, linkers try to rewrite it into an alternative code sequence.
See the comments in the implementations for detail.

In ld.lld this is implemented as a thunk, similar to a range extension thunk.
ld.lld additionally sets a [workaround](https://github.com/llvm/llvm-project/commit/20489ec5639e01c3c05ca5a39e6e43cac66c202f) when relocating `R_AARCH64_JUMP26`.

## `--android-memtag-{mode,stack,heap}`

The options instruct ld.lld to create `DT_AARCH64_MEMTAG_*` dynamic tags.
See [Memtag ABI Extension to ELF for the Arm® 64-bit Architecture (AArch64)](https://github.com/ARM-software/abi-aa/blob/main/memtagabielf64/memtagabielf64.rst).

## PAuth ABI

`-z pac-plt` enables PAC PLT and sets the `DT_AARCH64_PAC_PLT` dynamic tag.
