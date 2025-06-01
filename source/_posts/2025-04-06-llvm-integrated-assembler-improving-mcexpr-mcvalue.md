---
layout: post
title: "LLVM integrated assembler: Improving MCExpr and MCValue"
author: MaskRay
tags: [assembler,llvm]
---

In my previous post, [_Relocation Generation in Assemblers_](/blog/2025-03-16-relocation-generation-in-assemblers),
I explored some key concepts behind LLVM’s integrated assemblers. This post dives into recent improvements I’ve made to refine that system.

The LLVM integrated assembler handles fixups and relocatable expressions as distinct entities. Relocatable expressions, in particular, are encoded using the `MCValue` class, which originally looked like this:

```cpp
class MCValue {
  const MCSymbolRefExpr *SymA = nullptr, *SymB = nullptr;
  int64_t Cst = 0;
  uint32_t RefKind = 0;
};
```

<!-- more -->

In this structure:

* `RefKind` acts as an optional relocation specifier, though only a handful of targets actually use it.
* `SymA` represents an optional symbol reference (the addend).
* `SymB` represents another optional symbol reference (the subtrahend).
* `Cst` holds a constant value.

While functional, this design had its flaws. For one, the way relocation specifiers were encoded varied across architectures:

* Targets like COFF, Mach-O, and ELF's PowerPC, SystemZ, and X86 embed the relocation specifier within `MCSymbolRefExpr *SymA` as part of `SubclassData`.
* Conversely, ELF targets such as AArch64, MIPS, and RISC-V store it as a target-specific subclass of `MCTargetExpr`, and convert it to `MCValue::RefKind` during `MCValue::evaluateAsRelocatable`.

Another issue was with `SymB`. Despite being typed as `const MCSymbolRefExpr *`, its `MCSymbolRefExpr::VariantKind` field went unused.
This is because expressions like `add - sub@got` are not relocatable.

Over the weekend, I tackled these inconsistencies and reworked the representation into something cleaner:

```cpp
class MCValue {
  const MCSymbol *SymA = nullptr, *SymB = nullptr;
  int64_t Cst = 0;
  uint32_t Specifier = 0;
};

```
This updated design not only aligns more closely with the concept of relocatable expressions but also shaves off some compiler time in LLVM.
The ambiguous `RefKind` has been renamed to `Specifier` for clarity. Additionally, targets that previously encoded the relocation specifier within `MCSymbolRefExpr` (rather than using `MCTargetExpr`) can now access it directly via `MCValue::Specifier`.

To support this change, I made a few adjustments:

* [Introduced `getAddSym` and `getSubSym` methods](https://github.com/llvm/llvm-project/commit/83c3ec1b07c6c6857379cbdc6819262f2813b8e3), returning `const MCSymbol *`, as replacements for `getSymA` and `getSymB`.
* Eliminated dependencies on the old accessors, `MCValue::getSymA` and `MCValue::getSymB`.
* [Reworked the expression folding code that handles + and -](https://github.com/llvm/llvm-project/commit/33246f79e87a0e629ae776d1811a1175a3f10065)
* [Stored the `const MCSymbolRefExpr *SymA` specifier at `MCValue::Specifier`](https://github.com/llvm/llvm-project/commit/94821ce45fe93aa78cc5ea03cd9deac91b7af127)
* Some targets relied on PC-relative fixups with explicit specifiers forcing relocations. I have [defined `MCAsmBackend::shouldForceRelocation` for SystemZ](https://github.com/llvm/llvm-project/commit/38c3ad36be1facbe6db2dede7e93c0f12fb4e1dc) and [cleaned up ARM and PowerPC](https://github.com/llvm/llvm-project/commit/4182d2dcb5ecbfc34d41a6cd11810cd36844eddb)
* [Changed the type of `SymA` and `SymB` to `const MCSymbol *`](https://github.com/llvm/llvm-project/commit/d5893fc2a7e1191afdb4940469ec9371a319b114)
* [Replaced the temporary `getSymSpecifier` with `getSpecifier`](https://github.com/llvm/llvm-project/commit/e5923936109ce4ce7be2c8fb3372b14d33c385d9)
* [Replaced the legacy `getAccessVariant` with `getSpecifier`](https://github.com/llvm/llvm-project/commit/8fa5b6cc0293d806e36b90d4116e5925fa5d7f2e)

## Streamlining Mach-O support

Mach-O assembler support in LLVM has accumulated significant technical debt, impacting both target-specific and generic code.
One particularly nagging issue was the `const SectionAddrMap *Addrs` parameter in `MCExpr::evaluateAs*` functions.
This parameter existed to handle cross-section label differences, primarily for generating (compact) unwind information in Mach-O. A typical example of this can be seen in assembly like:

```asm
        .section        __TEXT,__text,regular,pure_instructions
Leh_func_begin0:
        .section        __TEXT,__eh_frame,coalesced,no_toc+strip_static_syms+live_support
Ltmp3:
Ltmp4 = Leh_func_begin0-Ltmp3
        .long   Ltmp4
```

The `SectionAddrMap *Addrs` parameter always felt like a clunky workaround to me.
It wasn’t until I dug into the [Mach-O AArch64 object writer](https://github.com/llvm/llvm-project/tree/main/llvm/lib/Target/AArch64/MCTargetDesc/AArch64MachObjectWriter.cpp) that I realized this hack wasn't necessary for that writer.
This discovery prompted a cleanup effort to remove the dependency on `SectionAddrMap` for ARM and X86 and eliminate the parameter:

* [[MC,MachO] Replace SectionAddrMap workaround with cleaner variable handling](https://github.com/llvm/llvm-project/commit/1b7759de8e6979dda2d949b1ba1c742922e5c366)
* [MCExpr: Remove unused SectionAddrMap workaround](https://github.com/llvm/llvm-project/commit/b90a92687f399df5afe3e1a2493b0d9c6295ac8c)

While I was at it, I also tidied up `MCSymbolRefExpr` by [removing the clunky `HasSubsectionsViaSymbolsBit`](https://github.com/llvm/llvm-project/commit/768ccf69f3febe962e0d63dc87fbee31e59547a7), further simplifying the codebase.

## Stremlining InstPrinter

The MCExpr code also determines how expression operands in assembly instructions are printed.
I have made improvements in this area as well:

* [[MC] Don't print () around $ names](https://github.com/llvm/llvm-project/commit/3acccf042ab8a7b7e663bb2b2fac328d9bf65b38)
* [[MC] Simplify MCBinaryExpr/MCUnaryExpr printing by reducing parentheses](https://github.com/llvm/llvm-project/commit/04a67528d303ac4be7943b2ae57222f9c9fd509a)
