---
layout: post
title: "LLVM integrated assembler: Improving sections and symbols"
author: MaskRay
tags: [assembler,llvm]
---

In my previous post, [LLVM integrated assembler: Improving expressions and relocations](/blog/2025-05-26-llvm-integrated-assembler-improving-expressions-and-relocations) delved into enhancements made to LLVM's expression resolving and relocation generation.
This post covers recent refinements to MC, focusing on sections and symbols.

<!-- more -->

## Sections

Sections are named, contiguous blocks of code or data within an object file.
They allow you to logically group related parts of your program.
The assembler places code and data into these sections as it processes the source file.


```cpp
class MCSection {
...
  enum SectionVariant {
    SV_COFF = 0,
    SV_ELF,
    SV_GOFF,
    SV_MachO,
    SV_Wasm,
    SV_XCOFF,
    SV_SPIRV,
    SV_DXContainer,
  };
```

In LLVM 20, the [`MCSection` class](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/include/llvm/MC/MCSection.h) used an enum called `SectionVariant` to differentiate between various object file formats, such as ELF, Mach-O, and COFF.
These subclasses are used in contexts where the section type is known at compile-time, such as in `MCStreamer` and [`MCObjectTargetWriter`](https://github.com/llvm/llvm-project/blob/release/21.x/llvm/include/llvm/MC/MCObjectWriter.h).
This change eliminates the need for runtime type information (RTTI) checks, simplifying the codebase and improving efficiency.

Additionally, the storage for fragments' fixups (adjustments to addresses and offsets) has been moved into the `MCSection` class.

## Symbols

Symbols are names that represent memory addresses or values.

```cpp
class MCSymbol {
protected:
  /// The kind of the symbol.  If it is any value other than unset then this
  /// class is actually one of the appropriate subclasses of MCSymbol.
  enum SymbolKind {
    SymbolKindUnset,
    SymbolKindCOFF,
    SymbolKindELF,
    SymbolKindGOFF,
    SymbolKindMachO,
    SymbolKindWasm,
    SymbolKindXCOFF,
  };

  /// A symbol can contain an Offset, or Value, or be Common, but never more
  /// than one of these.
  enum Contents : uint8_t {
    SymContentsUnset,
    SymContentsOffset,
    SymContentsVariable,
    SymContentsCommon,
    SymContentsTargetCommon, // Index stores the section index
  };
```

Similar to sections, the [`MCSymbol` class](https://github.com/llvm/llvm-project/blob/release/20.x/llvm/include/llvm/MC/MCSymbol.h) also used a discriminator enum, SymbolKind, to distinguish between object file formats.
This enum has also been removed.

Furthermore, the `MCSymbol` class had an `enum Contents` to specify the kind of symbol.
This name was a bit confusing, so it has been [renamed to `enum Kind`](https://github.com/llvm/llvm-project/commit/190778a8ba6d30995b7e1b4b4a556ab6444bdf3a) for clarity.

* regular symbol
* [equated symbol](/blog/2023-05-08-assemblers#symbol-equating-directives)
* [common symbol](/blog/2022-02-06-all-about-common-symbols)

A special enumerator, `SymContentsTargetCommon`, which was used by AMDGPU for a specific type of common symbol, has also been [removed](https://github.com/llvm/llvm-project/commit/aa96e20dcefa7d73229c98a7d2727696ff949459).
The functionality it provided is now handled by updating `ELFObjectWriter` to respect the symbol's section index (`SHN_AMDGPU_LDS` for this special AMDGPU symbol).

`sizeof(MCSymbol)` has been reduced to 24 bytes on 64-bit systems.

The previous blog post [LLVM integrated assembler: Improving expressions and relocations](/blog/2025-05-26-llvm-integrated-assembler-improving-expressions-and-relocations) describes other changes:

* The `MCSymbol::IsUsed` flag was a workaround for detecting a subset of invalid reassignments and is [removed](https://github.com/llvm/llvm-project/commit/e015626f189dc76f8df9fdc25a47638c6a2f3feb).
* The `MCSymbol::IsResolving` flag is added to detect cyclic dependencies of equated symbols.
