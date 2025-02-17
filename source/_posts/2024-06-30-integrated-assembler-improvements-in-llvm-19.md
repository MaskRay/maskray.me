layout: post
title: Integrated assembler improvements in LLVM 19
author: MaskRay
tags: [assebmly,llvm]
---

Within the LLVM project, MC is a library responsible for handling assembly, disassembly, and object file formats.
[Intro to the LLVM MC Project](https://blog.llvm.org/2010/04/intro-to-llvm-mc-project.html), which was written back in 2010, remains a good source to understand the high-level structures.

In the latest release cycle, substantial effort has been dedicated to refining MC's internal representation for improved performance and readability.
These changes have [decreased compile time significantly](#summary).
This blog post will delve into the details, providing insights into the specific changes.

<!-- more -->

## Merged `MCAsmLayout` into `MCAssembler`

`MCAssembler` manages assembler states (including sections, symbols) and implements post-parsing passes (computing a layout and writing an object file).
`MCAsmLayout`, tightly coupled with `MCAssembler`, was in charge of symbol and fragment offsets during `MCAssembler::Finish`.
`MCAsmLayout` was a wrapper of `MCAssembler` and a section order vector (actually Mach-O specific).
Many `MCAssembler` and `MCExpr` member functions have a `const MCAsmLayout &` parameter, contributing to slight overhead.
Here are some functions that are called frequently:

* `MCAssembler::computeFragmentSize` is called a lot in the layout process.
* `MCAsmBackend::handleFixup` and `MCAsmBackend::applyFixup` evaluate each fixup and produce relocations.
* `MCAssembler::fixupNeedsRelaxation` determines whether a `MCRelaxableFragment` needs relaxation due to a `MCFixup`.
* `MCAssembler::relaxFragment` and `MCAssembler::relaxInstruction` relax a fragment.

I [started to merge](https://github.com/llvm/llvm-project/commit/67957a45ee1ec42ae1671cdbfa0d73127346cc95) `MCAsmLayout` into `MCAssembler` and simplify MC code, and eventually removed [`llvm/include/llvm/MC/MCAsmLayout.h`](https://github.com/llvm/llvm-project/commit/122db8b2cb7fa43ce1d6dc17148080579fcfb55a).

## Fragments

Fragments, representing sequences of non-relaxable instructions, relaxable instruction, alignment directives, and other elements.
`MCDataFragment` and `MCRelaxableFragment`, whose sizes are crucial for memory consumption, have undergone several optimizations:

* [MCInst: decrease inline element count to 6](https://github.com/llvm/llvm-project/pull/94913)
* [[MC] Reduce size of MCDataFragment by 8 bytes](https://github.com/llvm/llvm-project/pull/95293) by @aengelke
* [[MC] Move MCFragment::Atom to MCSectionMachO::Atoms](https://github.com/llvm/llvm-project/pull/95341)

The fragment management system has also been streamlined by transitioning from a doubly-linked list (`llvm::iplist`) to a [singly-linked list](https://github.com/llvm/llvm-project/pull/95077), eliminating unnecessary overhead.
A few prerequisite commits removed backward iterator requirements.

Furthermore, I [introduced the "current fragment" concept](https://github.com/llvm/llvm-project/commit/e48c4011ca80385573f1b92793c75dc98abb228f) (`MCSteamer::CurFrag`) allowing for faster appending of new fragments.

I have also simplified and optimized fragment offset computation:

* [`[MC] Relax fragments eagerly`](https://github.com/llvm/llvm-project/commit/9d0754ada5dbbc0c009bcc2f7824488419cc5530)

<!-- 
* [`[MC] Compute fragment offsets eagerly`](https://github.com/llvm/llvm-project/commit/be5a845e4c29aadb513ae6e5e2879dccf37efdbb)
reverted due to -O0 split3.i code size increase
-->

Previously, calculating fragment offsets happened lazily in the `getFragmentOffset` function.
All sections were iteratively relaxed until they all converged. This process was inefficient as the slowest section determined the number of iterations for all others, resulting in extra calculations.

Previously, fragment offset computation was lazily performed by `getFragmentOffset`.
The section that converged the slowest determined other sections' iteration steps, leading to some unneeded computation.

The new layout algorithm assigns fragment offsets and iteratively refines them for each section until it's optimized.
Then, it moves on to the next section. If relaxation doesn't change anything, fragment offset assignment will be skipped.
This way, sections that converge quickly don't have to wait for the slowest ones, resulting in [a significant decrease in compile time for full LTO](https://llvm-compile-time-tracker.com/compare.php?from=0387cd052b081d6bc9856ef756942a5df1a2a301&to=1a47f3f3db66589c11f8ddacfeaecc03fb80c510&stat=instructions%3Au&linkStats=on).

```cpp
bool MCAssembler::relaxOnce() {
  bool ChangedAny = false;
  for (MCSection &Sec : *this) {
    auto MaxIter = NumFrags + 1;
    uint64_t OldSize = getSectionAddressSize(Sec);
    do {
      uint64_t Offset = 0;
      Changed = false;
      for (MCFragment &F : Sec) {
        if (F.Offset != Offset) {
          F.Offset = Offset;
          Changed = true;
        }
        relaxFragment(F);
        Offset += computeFragmentSize(F);
      }

      Changed |= OldSize != Offset;
      ChangedAny |= Changed;
      OldSize = Offset;
    } while (Changed && --MaxIter);
    if (MaxIter == 0)
      return false;
  }
  return ChangedAny;
}
```

## Symbols

@aengelke made two noticeable performance improvements:

* [[MC] Don't evaluate name of unnamed symbols](https://github.com/llvm/llvm-project/pull/95021)
* [[MC] Eliminate two symbol-related hash maps](https://github.com/llvm/llvm-project/pull/95464)

In `MCObjectStreamer`, newly defined labels were put into a "pending label" list and initially assigned to a `MCDummyFragment` associated with the current section.
The symbols will be reassigned to a new fragment when the next instruction or directive is parsed.
This pending label system, while necessary for aligned bundling, introduced complexity and potential for subtle bugs.

To streamline this, I [revamped the implementation](https://github.com/llvm/llvm-project/commit/75006466296ed4b0f845cbbec4bf77c21de43b40) by directly adjusting offsets of existing fragments, eliminating over 100 lines of code and reducing the potential for errors.

Details: In 2014, [[MC] Attach labels to existing fragments instead of using a separate fragment](https://reviews.llvm.org/D5915) introduced `flushPendingLabels` aligned bundling assembler extension for Native Client.
[[MC] Match labels to existing fragments even when switching sections.](https://reviews.llvm.org/D71368), built on top of `flushPendingLabels`, added further complication.

In `MCObjectStreamer`, a newly defined label was temporarily assigned to a `MCDummyFragment`.
The symbol would be reassigned to a new fragment when the next instruction or directive was parsed.
The `MCDummyFragment` was not in the section's fragment list. However, during expression evaluation, it should be considered as the temporary end of the section.

For the following code, aligned bundling requires that `.Ltmp` is defined at `addl`.
```
$ clang var.c -S -o - -fPIC -m32
...
.bundle_lock align_to_end
  calll   .L0$pb
.bundle_unlock
.L0$pb:
  popl    %eax
.Ltmp0:
  addl    $_GLOBAL_OFFSET_TABLE_+(.Ltmp0-.L0$pb), %eax
```

Worse, a lot of directive handling code had to add `flushPendingLabels` and a missing `flushPendingLabels` could lead to subtle bugs related to incorrect symbol values.

(
`MCAsmStreamer` doesn't call `flushPendingLabels` in its handlers.
This is the reason that we cannot change `MCAsmStreamer::getAssemblerPtr` to use a `MCAssembler` and change `AsmParser::parseExpression`.
)

## Sections

Section handling was also refined.
MCStreamer maintains a a section stack for features like `.push_section`/`.pop_section`/`.previous` directives.
Many functions relied on the section stack for loading the current section, which introduced overhead due to the additional indirection and nullable return values.

By leveraging the "current fragment" concept, the need for the section stack was eliminated in most cases, simplifying the codebase and improving efficiency.

I have eliminated nullable `getCurrentSectionOnly` uses and [changed `getCurrentSectionOnly` to leverage the "current fragment" concept](https://github.com/llvm/llvm-project/commit/626eef5ecf92e98cbfccfa6134e0a760e7592813).
This change also [revealed an interesting quirk](https://github.com/llvm/llvm-project/commit/4ae23bcca144b542f16d45acc8f270e156e2fa4e) in NVPTX assembly related to DWARF sections.

## Section symbols

Many section creation functions (`MCContext::get*Section`) had a [`const char *BeginSymName` parameter](https://github.com/llvm/llvm-project/commit/6b9998b3ebd4f21ba726e65e5fe2636e4eeed598) to support the section symbol concept.
This led to issues when we want to treat the section name as a symbol.
In 2017, the parameter was [removed for ELF](https://github.com/llvm/llvm-project/commit/13a79bbfe583e1d8cc85d241b580907260065eb8), streamlining section symbol handling.

I changed the way MC handles section symbols for [COFF](https://github.com/llvm/llvm-project/pull/96459) and removed the unused parameters for WebAssembly.
The work planned for XCOFF is outlined in <https://github.com/llvm/llvm-project/issues/96810>.

## Expression evaluation

Expression evaluation in `MCAssembler::layout` previously employed a complex lazy evaluation algorithm, which aimed to minize the number of fragment relaxation.
It proved difficult to understand and resulted in [complex recursion detection](https://reviews.llvm.org/D79570).

To address this, I removed lazy evaluation in favor of [eager fragment relaxation](https://github.com/llvm/llvm-project/commit/9d0754ada5dbbc0c009bcc2f7824488419cc5530).
This simplification improved the reliability of the layout process, eliminating the need for intricate workarounds like the `MCFragment::IsBeingLaidOut` flag introduced earlier.

Note: the benefit of lazy evaluation largely diminished when <https://reviews.llvm.org/D76114> invalidated all sections to fix the correctness issue for the following assembly:

```asm
.section .text1,"ax"
 .skip after-before,0x0
.L0:

  .section .text2
before:
  jmp .L0
after:
```

In addition, I removed [an overload of isSymbolRefDifferenceFullyResolvedImpl](https://github.com/llvm/llvm-project/commit/3a90c27385e9a8b50e40a307822906a1303c310b), enabling constant folding for variable differences in Mach-O.

## Target-specific features misplaced in the generic implementation

I have made efforts to relocate target-specific functionalities to their respective target implementations:

* [[MC,X86] emitInstruction: remove virtual function calls due to Intel JCC Erratum](https://github.com/llvm/llvm-project/pull/96835)
* [[MC,X86] De-virtualize emitPrefix](https://github.com/llvm/llvm-project/pull/97008)
* [[MC] Move Mach-O specific getAtom and isSectionAtomizableBySymbols to Mach-O files](https://github.com/llvm/llvm-project/commit/41a08e764aeec92703285754b5e8acd85283b1a6)
* [[MC] Move ELFWriter::createMemtagRelocs to AArch64TargetELFStreamer::finish](https://github.com/llvm/llvm-project/commit/fec1b6f9d3cf5347b67ffb2078c995eb496acf47)
* [[MC] Move MCAsmLayout::SectionOrder to MachObjectWriter::SectionOrder](https://github.com/llvm/llvm-project/commit/7840c0066837797cdeb62aab63044b964aa7f372)
* [Move MCSection::LayoutOrder to MCSectionMachO](https://github.com/llvm/llvm-project/pull/97474)
* [[MC] Move isPrivateExtern to MCSymbolMachO](https://github.com/llvm/llvm-project/commit/e299b163c78d34cf6fb90f9d928291419b5dcaaf)
* [[MC] Export llvm::WinCOFFObjectWriter and access it from MCWinCOFFStreamer](https://github.com/llvm/llvm-project/commit/9539a7796094ff5fb59d9c685140ea2e214b945c)
* [[MC] Move VersionInfo to MachObjectWriter](https://github.com/llvm/llvm-project/commit/09a399a1ddbef1e5e51b77fd6bd1792d34697187)
* [[MC] Export llvm::ELFObjectWriter](https://github.com/llvm/llvm-project/commit/70c52b62c5669993e341664a63bfbe5245e32884)
* [MCObjectWriter: Remove XCOFF specific virtual functions](https://github.com/llvm/llvm-project/commit/023c6454971dc33e409ce9a1035b20bd0ff893f2)

The class hierarchy has been cleaned up by making more `MC*ObjectWriter` public and accessing them from `MC*Streamer`.

## CREL

The integrated assembler now supports [CREL (compact relocation) for ELF](https://discourse.llvm.org/t/rfc-crel-a-compact-relocation-format-for-elf/77600).

Once the Clang and lld patches are merged, enabling compact relocations is as simple as this:

`clang -c -Wa,--crel,--allow-experimental-crel a.c && clang -fuse-ld=lld a.o`.

Note: This feature is unstable. While relocatable files created with Clang version A will work with lld version A, they might not be compatible with newer versions of lld (where A is older than B).

As [the future of the generic ABI remains uncertain](https://maskray.me/blog/2024-05-26-evolution-of-elf-object-file-format#a-future-in-flux), CREL might not get "standardized".
In that case, I will just get the section code agreed with the GNU community to ensure wider compatibility.

## Assembly parser

* `\+`, the per-macro invocation count, is [now available for `.irp/.irpc/.rept`](https://github.com/llvm/llvm-project/commit/70d6b8a358366ec2ef4e73d5809fe23b9abf527d).
* [[MCParser] .altmacro: Support argument expansion not preceded by \\](https://github.com/llvm/llvm-project/commit/da368f24050f34de0327d04068a608ba971fa47c)
* [[MC] Support .cfi_label](https://github.com/llvm/llvm-project/pull/97922)

## Summary

I've been contributing to MC for several years. Back then, while many contributed, most focused on adding specific features.
Rafael Ávila de Espíndola was the last to systematically review and improve the MC layer. Unfortunately, simplification efforts stalled after Rafael's departure in 2018.

Picking up where Rafael left off, I'm diving into the MC layer to streamline its design.
A big thanks to @aengelke for his invaluable performance centric contributions in this release cycle.
LLVM 19 introduces significant enhancements to the integrated assembler, resulting in notable performance gains, reduced memory usage, and a more streamlined codebase.
These optimizations pave the way for future improvements.

I compiled the preprocessed SQLite Amalgamation (from [llvm-test-suite](https://github.com/llvm/llvm-test-suite)) using a Release build of clang:

build  | 2024-05-14 | 2024-07-02
-------+------------+-----------
-O0    | 0.5304     | 0.4930
-O0 -g | 0.8818     | 0.7967
-O2    | 6.249      | 6.098
-O2 -g | 7.931      | 7.682

`clang -c -w sqlite3.i`

The AsmPrinter pass, which is coupled with the assembler, consumes a significant portion of the `-O0` compile time.
I have modified the `-ftime-report` mechanism to decrease the per-instruction overhead.
The decrease in compile time matches the decrease in the spent in AsmPrinter.
Coupled with a recent observation that BOLT, which heavily utilizes MC, is ~8% faster, it's clear that MC modifications have yielded substantial improvements.

## Noticeable optimizations in previous releases

[[MC] Always encode instruction into SmallVector](https://reviews.llvm.org/D145791) optimized `MCCodeEmitter::encodeInstruction` for x86 by avoiding `raw_ostream::write` overhead.
I have migrated other targets and [removed the extra overload](https://github.com/llvm/llvm-project/commit/e8934075b90a38f9dd3361472e696f11e50a8aa4).
```
raw_ostream::write =(inlinable)=> flush_tied_then_write (unneeded TiedStream check) =(virtual function call)=> raw_svector_ostream::write_impl ==> SmallVector append(ItTy in_start, ItTy in_end) (range; less efficient then push_back).
```

[[MC] Optimize relaxInstruction: remove SmallVector copy. NFC](https://github.com/llvm/llvm-project/commit/a08cbabb28e5e5f27ce802e1b26ba273a071a0f3) removes code and fixup copy for `relaxInstruction`.

## Roadmap

### Symbol redefinition

[llvm-mc: Diagnose misuse (mix) of defined symbols and labels.](https://github.com/llvm/llvm-project/commit/ae7ac010594f693fdf7b3ab879e196428d961e75) added redefinition error.
This was refined many times.
I hope to fix this in the future.

### Addressing Mach-O weakness

The Mach-O assembler lacks the robustness of its ELF counterpart. Notably, certain aspects of the Mach-O implementation, such as the conditions for constant folding in `MachObjectWriter::isSymbolRefDifferenceFullyResolvedImpl` (different for x86-64 and AArch64), warrant revisiting.

Additionally, the Mach-O has a hack to [maintain compatibility with Apple cctools assembler](https://github.com/llvm/llvm-project/issues/19577), when the relocation addend is non-zero.

```asm
.data
a = b + 4
.long a   # ARM64_RELOC_UNSIGNED(a) instead of b; This might work around the linker bug(?) when the referenced symbol is b and the addend is 4.

c = d
.long c   # ARM64_RELOC_UNSIGNED(d)

y:
x = y + 4
.long x   # ARM64_RELOC_UNSIGNED(x) instead of y
```

This leads to another workaround in `MCFragment.cpp:getSymbolOffsetImpl` ([[MC] Recursively calculate symbol offset](https://reviews.llvm.org/D109109)), which is to support the following assembly:

```asm
l_a:
l_b = l_a + 1
l_c = l_b
.long l_c
```

### Misc

* `emitLabel` at `switchSection` was for [DWARF sections](https://github.com/llvm/llvm-project/commit/f1a13f5ad5b54634989ae7d067f5364b56a5a8d4), which might be no longer useful
