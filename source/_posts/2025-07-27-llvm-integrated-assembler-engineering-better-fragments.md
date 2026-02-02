---
layout: post
title: "LLVM integrated assembler: Engineering better fragments"
author: MaskRay
tags: [assembler,llvm]
---

In my previous assembler posts, I've discussed improvements on [expression resolving and relocation generation](/blog/2025-05-26-llvm-integrated-assembler-improving-expressions-and-relocations).
Now, let's turn our attention to recent refinements within section fragments.
Understanding how an assembler utilizes these fragments is key to appreciating the improvements we've made.
At a high level, the process unfolds in three main stages:

* Parsing phase: The assembler constructs section fragments. These fragments represent sequences of regular instructions or data, [span-dependent instructions](/blog/2024-04-27-understanding-clang-o0-size-increase-branch-displacement), alignment directives, and other elements.
* Section layout phase: Once fragments are built, the assembler assigns offsets to them and finalizes the span-dependent content.
* [Relocation decision phase](/blog/2025-03-16-relocation-generation-in-assemblers): In the final stage, the assembler evaluates fixups and, if necessary, updates the content of the fragments.

<!-- more -->

When the LLVM integrated assembler was introduced in 2009, its section and fragment design was quite basic. Performance wasn't the concern at the time.
As LLVM evolved, many assembler features added over the years came to rely heavily on this original design.
This created a complex web that made optimizing the fragment representation increasingly challenging.

Here's a look at some of the features that added to this complexity over the years:

* 2010: Mach-O `.subsection_via_symbols` and atoms
* 2012: NativeClient's bundle alignment mode. I've created a dedicated chapter for this.
* 2015: Hexagon instruction bundle
* 2016: CodeView variable definition ranges
* 2018: RISC-V linker relaxation
* 2020: x86 `-mbranches-within-32B-boundaries`
* 2023: LoongArch linker relaxation. This is largely identical to RISC-V linker relaxation. Any refactoring or improvements to the RISC-V linker relaxation often necessitate corresponding changes to the LoongArch implementation.
* 2023: z/OS [GOFF (Generalized Object File Format)](https://en.wikipedia.org/wiki/GOFF)

I've included the start year for each feature to indicate when it was initially introduced, to the best of my knowledge.
This doesn't imply that maintenance stopped after that year. On the contrary, many of these features, like RISC-V linker relaxation, require ongoing, active maintenance.

Despite the intricate history, I've managed to untangle these dependencies and implement the necessary fixes.
And that, in a nutshell, is what this blog post is all about!

## Reducing sizeof(MCFragment)

A significant aspect of optimizing fragment management involved directly reducing the memory footprint of the MCFragment object itself.
Several targeted changes contributed to making `sizeof(MCFragment)` smaller, as mentioned by my previous blog post: [Integrated assembler improvements in LLVM 19](/blog/2024-06-30-integrated-assembler-improvements-in-llvm-19#fragments).

* [MCInst: decrease inline element count to 6](https://github.com/llvm/llvm-project/pull/94913)
* [[MC] Reduce size of MCDataFragment by 8 bytes](https://github.com/llvm/llvm-project/pull/95293) by @aengelke
* [[MC] Move MCFragment::Atom to MCSectionMachO::Atoms](https://github.com/llvm/llvm-project/pull/95341)

The fragment management system has also been streamlined by transitioning from a doubly-linked list (`llvm::iplist`) to a singly-linked list, eliminating unnecessary overhead.
A few prerequisite commits removed backward iterator requirements.
It's worth noting that the complexities introduced by features like NaCl's bundle alignment mode, x86's `-mbranches-within-32B-boundaries` option, and Hexagon's instruction bundles presented challenges.

## The quest for trivially destructible fragments

Historically, `MCFragment` subclasses, specifically `MCDataFragment` and `MCRelaxableFragment`, relied on `SmallVector` member variables to store their content and fixups.
This approach, while functional, presented two key inefficiencies:

* Inefficient storage of small objects: The content and fixups for individual fragments are typically very small. Storing a multitude of these tiny objects individually within `SmallVectors` led to less-than-optimal memory utilization.
* Non-trivial destructors: When deallocating sections, the `~MCSection` destructor had to meticulously traverse the fragment list and explicitly destroy each fragment.

In 2024, @aengelke initiated a draft to store fragment content out-of-line.
Building upon that foundation, I've extended this approach to also store fixups out-of-line, and ensured compatibility with the aforementioned features that cause complexity (especially RISC-V and LoongArch linker relaxation.)

Furthermore, `MCRelaxableFragment` previously contained `MCInst Inst;`, which also necessitated a non-trivial destructor.
To address this, I've redesigned its data structure. operands are now stored within the parent MCSection, and the `MCRelaxableFragment` itself only holds references:

```cpp
  uint32_t Opcode = 0;
  uint32_t Flags = 0; // x86-only for the EVEX prefix
  uint32_t OperandStart = 0;
  uint32_t OperandSize = 0;
```

Unfortunately, we still need to encode `MCInst::Flags` to support the x86 EVEX prefix, e.g., `{evex} xorw $foo, %ax`.
My hope is that the x86 maintainers might refactor `X86MCCodeEmitter::encodeInstruction` to make this flag storage unnecessary.

The new design of `MCFragment` and `MCSection` is as follows:

```cpp
class MCFragment {
  ...
  // Track content and fixups for the fixed-size part as fragments are
  // appended to the section. The content remains immutable, except when
  // modified by applyFixup.
  uint32_t ContentStart = 0;
  uint32_t ContentEnd = 0;
  uint32_t FixupStart = 0;
  uint32_t FixupEnd = 0;

  // Track content and fixups for the optional variable-size tail part,
  // typically modified during relaxation.
  uint32_t VarContentStart = 0;
  uint32_t VarContentEnd = 0;
  uint32_t VarFixupStart = 0;
  uint32_t VarFixupEnd = 0;
};

class MCSection {
  ...
  // Content and fixup storage for fragments
  SmallVector<char, 0> ContentStorage;
  SmallVector<MCFixup, 0> FixupStorage;
  SmallVector<MCOperand, 0> MCOperandStorage;
};
```

(As a side note, [the LLVM `CamelCase` variables are odd](https://llvm.org/docs/Proposals/VariableNames.html). As the MC maintainer, I'd be delighted to see them refactored to `camelBack` or `snake_case` if people agree on the direction.)

Key changes:

* [MC: Store fragment content and fixups out-of-line](https://github.com/llvm/llvm-project/pull/146307)
* [CodeView: Move MCCVDefRangeFragment storage to MCContext/MCFragment. NFC](https://github.com/llvm/llvm-project/pull/146462)
* [MC: Store MCRelaxableFragment MCInst out-of-line](https://github.com/llvm/llvm-project/pull/147229)

## Fewer fragments: fixed-size part and variable tail

Prior to LLVM 21.1, the assembler, operated with a fragment design dating back to 2009, placed every span-dependent instruction into its own distinct fragment.
The x86 code sequence `push rax; jmp foo; nop; jmp foo` would be represented with numerous fragments: `MCDataFragment(nop); MCRelaxableFragment(jmp foo); MCDataFragment(nop); MCRelaxableFragment(jmp foo)`.

A more efficient approach emerged: storing both a **fixed-size part and an optional variable-size tail** within a single fragment.

* The fixed-size part maintains a consistent size throughout the assembly process.
* The variable-size tail, if present, encodes elements that can change in size or content, such as a span-dependent instruction, an alignment directive, a fill directive, or other similar span-dependent constructs.

The new design led to significantly fewer fragments:

```
MCFragment(fixed: push rax, variable: jmp foo)
MCFragment(fixed: nop, variable: jmp foo)
```

Key changes:

* [MC: Restructure MCFragment as a fixed part and a variable tail](https://github.com/llvm/llvm-project/pull/148544).
* [MC: Encode FT_Align in fragment's variable-size tail](https://github.com/llvm/llvm-project/pull/149030)

## Reducing instruction encoding overhead

Encoding individual instructions is the most performance-critical operation within `MCObjectStreamer`.
Recognizing this, significant effort has been dedicated to reducing this overhead since May 2023.

* [[MC] Always encode instruction into SmallVector](https://reviews.llvm.org/D145791)
* [[MC] Remove the legacy overload of encodeInstruction](https://github.com/llvm/llvm-project/commit/e8934075b90a38f9dd3361472e696f11e50a8aa4) with a lot of prior cleanups
* [[MC][ELF] Emit instructions directly into fragment](https://github.com/llvm/llvm-project/pull/94950)
* [[MC][X86] Avoid copying MCInst in emitInstrEnd](https://github.com/llvm/llvm-project/pull/94947) in 2024-06
* [X86AsmBackend: Remove some overhead from auto padding feature](https://github.com/llvm/llvm-project/commit/d77ac81e93e5e2df5275b687b53049d9acfe1357)
* [X86AsmBackend: Simplify isRightAfterData for the auto-pad feature](https://github.com/llvm/llvm-project/commit/3fe6d276dc952b3b2b487cb67a999c3981cf9563)

It's worth mentioning that x86's instruction padding features, introduced in 2020, have imposed considerable overhead.
Specifically, these features are:

* `-mbranches-within-32B-boundaries`. See [Align branches within 32-Byte boundary(NOP padding)](https://reviews.llvm.org/D70157)
* [[X86] Relax existing instructions to reduce the number of nops needed for alignment purposes](https://reviews.llvm.org/D75203)
* ["Enhanced relaxation"](https://reviews.llvm.org/D76286): The feature allows x86 prefix padding for all instructions, effectively making all instructions span-dependent and requiring its own fragment. 
  My [D94542](https://reviews.llvm.org/D94542) disabled this by default due to concern of `-g` vs `-g0` differences.

My recent optimization efforts demanded careful attention to these particularly complex and performance-sensitive code.

## Eager fragment creation

Encoding an instruction is a far more frequent operation than appending a variable-size tail to the current fragment.
In the previous design, the instruction encoder was burdened with an extra check: it had to determine if the current fragment already had a variable-size tail.

```
encodeInstruction:
  if (current fragment has a variable-size tail)
    start a new fragment
  append data to the current fragment

emitValueToAlignment:
  Encode the alignment in the variable-size tail of the current fragment

emitDwarfLocDirective:
  Encode the .loc in the variable-size tail of the current fragment
```

Our new strategy optimizes this by maintaining a current fragment that is guaranteed not to have a variable-size tail.
This means functions appending data to the fixed-size part no longer need to perform this check.
Instead, any function that sets a variable-size tail will now immediately start a new fragment.

Here's how the workflow looks with this optimization:

```
encodeInstruction:
  assert(current fragment doesn't have a variable-size tail)
  append data to the current fragment

emitValueToAlignment:
  Encode the alignment in the variable-size tail of the current fragment
  start a new fragment

emitDwarfLocDirective:
  Encode the .loc in the variable-size tail of the current fragment
  start a new fragment
```

Key changes:

* [MC: Simplify fragment reuse determination](https://github.com/llvm/llvm-project/pull/149471)
* [MC: Optimize getOrCreateDataFragment](https://github.com/llvm/llvm-project/commit/39c8cfb70d203439e3296dfdfe3d41f1cb2ec551)

It's worth noting that the first patch was made possible thanks to the removal of the bundle alignment mode.

## Fragment content in trailing data 

Our `MCFragment` class manages four distinct sets of appendable data: fixed-size content, fixed-size fixups, variable-size tail content, and variable-size tail fixups.
Of these, the fixed-size content is typically the largest.
We can optimize its storage by utilizing it as trailing data, akin to a flexible array member.

This approach offers several compelling advantages:

* Improved data locality: Storing the content after the MCFragment object enhances cache utility.
* Simplified metadata: We can replace the pair of `uint32_t ContentStart = 0; uint32_t ContentEnd = 0;` with a single `uint32_t ContentSize;`.

This optimization leverages a clever technique made possible by using a special purpose bump allocator.
After allocating `sizeof(MCFragment)` bytes for a new fragment, we know that any remaining space within the current bump allocator block immediately follows the fragment's end.
This contiguous space can then be efficiently used for the fragment's trailing data.

However, this design introduces a few important considerations:

* Tail fragment appends only: Data can only be appended to the tail fragment of a subsection. Fragments located in the middle of a subsection are immutable in their fixed-size content. Any post-assembler-layout adjustments must target the variable-size tail.
* Dynamic Allocation Management: When new data needs to be appended, a function is invoked to ensure the current bump allocator block has sufficient space.
  If not, the current fragment is closed (its fixed-size content is finalized), and a new fragment is started.
  For instance, an 8-byte sequence could be stored as one single fragment, or, if space constraints dictate, as two fragments each encoding 4 bytes.
* New block allocation: If the available space in the current block is insufficient, a new block large enough to accommodate both an MCFragment and the required bytes for its trailing data is allocated.
* Section/subsection Switching: The previously saved fragment list tail cannot be simply reused. This is because it's tied to the memory space of the previous bump allocator block. Instead, a new fragment must be allocated using the current bump allocator block and appended to the new subsection's tail.

I have thought about making the variable-size content immediately follow the fixed-size content, but leb128 and x86's potentially very long instruction (15 bytes) stopped me from doing it.
There is certainly room for future improvements, though.

Key changes:

* [GOFF: Only register sections within MCObjectStreamer::changeSection](https://github.com/llvm/llvm-project/pull/150183)
* [MC: Allocate initial fragment and define section symbol in changeSection](https://github.com/llvm/llvm-project/pull/150574)
* [MCFragment: Use trailing data for fixed-size part](https://github.com/llvm/llvm-project/pull/150846)

## Fragment fixups stored in section

TODO

MCFragment should not hold references to fixups stored in the parent MCSection. Instead, fixups reference the fragment.

The optional variable-size tail of a fragment can have at most one fixup.

## Deprecating complexity: NativeClient's bundle alignment mode

Google's now-discontinued Native Client (NaCl) project provided a sandboxing environment through a combination of Software Fault Isolation (SFI) and memory segmentation.
A distinctive feature of its SFI implementation was the "bundle alignment mode", which adds NOP padding to ensure that no instruction crosses a 32-byte alignment boundary.
The verifier's job is to check all instructions starting at 32-byte-multiple addresses.

While the core concept of aligned bundling is intriguing, its implementation within the LLVM assembler proved problematic.
Introduced in 2012, this feature imposed noticeable performance penalties on users who had no need for NaCl, perhaps more critically, significantly increased the complexity of MC's internal workings.
I was particularly concerned by its pervasive modifications to `MCObjectStreamer` and `MCAssembler`.

The complexity deepened with the introduction of

* 2014: [MCStreamer's pending labels](https://reviews.llvm.org/D5915), which led to more complexity:
  - 2015: [[MC] Ensure that pending labels are flushed when -mc-relax-all flag is used](https://reviews.llvm.org/D10325)
  - 2019: [[MC] Match labels to existing fragments even when switching sections.](https://reviews.llvm.org/D71368) by an Apple developer. In a nutshell, the pending labels mechanism was causing headache to Mach-O, requiring additional code to manage.
* 2015: [NaCl's `mc-relax-all` optimization](https://reviews.llvm.org/D8072)

In `MCObjectStreamer`, newly defined labels were put into a "pending label" list and initially assigned to a `MCDummyFragment` associated with the current section.
The symbols would be reassigned to a new fragment when the next instruction or directive was parsed.
This pending label system introduced complexity and a missing `flushPendingLabels` could lead to subtle bugs related to incorrect symbol values.
`flushPendingLabels` was called by many `MCObjectStreamer` functions, noticeably once for each new fragment, adding overhead.
It also complicated the label difference evaluation due to `MCDummyFragment` in `MCExpr.cpp:AttemptToFoldSymbolOffsetDifference`.

For the following code, aligned bundling requires that .Ltmp is defined at addl.

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

Recognizing these long-standing issues, a series of pivotal changes were undertaken:

* 2024: [[MC] Aligned bundling: remove special handling for RelaxAll](https://github.com/llvm/llvm-project/pull/95188) removed an optimization for NaCl in the [`mc-relax-all` mode](/blog/2024-04-27-understanding-clang-o0-size-increase-branch-displacement)
* 2024: [[MC] Remove pending labels](https://github.com/llvm/llvm-project/commit/75006466296ed4b0f845cbbec4bf77c21de43b40)
* 2024: [[MC] AttemptToFoldSymbolOffsetDifference: remove MCDummyFragment check. NFC](https://github.com/llvm/llvm-project/commit/8fa4fe1f995a9bc85666d63e84c094f9a09686bgn5)
* 2025: Finally, [MC: Remove bundle alignment mode](https://github.com/llvm/llvm-project/pull/148781), after Derek Schuff agreed to drop NaCl support from LLVM.

Should future features require a variant of bundle alignment, I firmly believe a much cleaner implementation is necessary.
This could potentially be achieved through a backend hook within `X86AsmBackend::finishLayout`, applied after the primary assembler layout phase, similar to how the `-mbranches-within-32B-boundaries` option is handled, though even that implementation warrants an extensive revisit itself.

## Lessons learned

**The cost of missing early optimization**

Early design choices can have a far-reaching impact on future code.
The initial LLVM MC design, while admirably simple in its inception, inadvertently created a rigid foundation.
As new features piled on, each relying more and more on the specific fragment internals, rectifying foundational inefficiencies became incredibly challenging.
The Hyrum's Law was evident: features built on this foundation inevitably depended on all its observable behaviors.
Optimizing the underlying structure required not just a change to the core, but also a thorough fix for all its unsuspecting users.
I encountered significant struggles with the deeply ingrained complexities stemming from NaCl's bundle alignment mode, x86's `-mbranches-within-32B-boundaries` option, and the intricacies of RISC-V linker relaxation.

**Cargo cult programming and snowball effect**

I observed instances of "cargo cult programming", where existing solutions were copied without a full understanding of their underlying rationale or applicability. For example:

* The WebAssembly implementation heavily mirrored that of ELF. Consequently, many improvements made to the ELF component often necessitated corresponding, sometimes redundant, changes to the WebAssembly implementation.
  In additin, the WebAssembly implementation copied ELF-specific code that was irrelevant for WebAssembly's architecture, adding unnecessary bloat and complexity.
* LoongArch's RISC-V replication: LoongArch's linker relaxation implementation directly copied the approach taken for RISC-V.
  Refactoring or improvements to RISC-V's linker relaxation frequently require mirrored changes in the LoongArch codebase, creating parallel maintenance burdens.
  I am particularly glad that I landed my foundational [[RISCV] Make linker-relaxable instructions terminate MCDataFragment](https://reviews.llvm.org/D153097) and [[RISCV] Allow delayed decision for ADD/SUB relocations](https://reviews.llvm.org/D155357) in 2023, before the LoongArch team replicated the RISC-V approach.
  This timing, I hope, mitigated some future headaches for their implementation.

These patterns illustrate how initial design choices, or the expedience of copying existing solutions, can lead to a "snowball effect" of accumulating complexity and redundant code that makes future optimization and maintenance significantly harder.
On a positive note, I'm also pleased that [the streamlining of the relocation generation framework](/blog/2025-05-26-llvm-integrated-assembler-improving-expressions-and-relocations) was completed before Apple's upstreaming of their Mach-O support for 32-bit RISC-V.
This critical work should provide a more robust and less complex base for their contributions, and reducing maintenance on my end.

**The cost of features**

Specific features, particularly those designed for niche or specialized use cases like NaCl's bundle alignment mode, introduced disproportionate complexity and performance overhead across the entire assembler.
Even though NaCl itself was deprecated in 2020, it took until 2025 to finally excise its complex support from LLVM.
This highlights a common challenge in large, open-source projects: while many developers are motivated to add new features, there's often far less incentive or dedicated effort to streamline or remove their underlying implementation complexities once they're no longer strictly necessary or have become a performance drain.

I want to acknowledge the work of individuals like Rafael Ávila de Espíndola, Saleem Abdulrasool, and Nirav Dave, whose improvements to LLVM MC were vital.
Without their contributions, the MC layer would undoubtedly be in a far less optimized state today.

## Epilogue

This extensive work on fragment optimization would not have been possible without the invaluable contributions of [Alexis Engelke](https://aengelke.net/).
My sincere thanks go to Alexis for his meticulous reviews of numerous patches, his insightful suggestions, and for contributing many significant improvements himself.

What I have learnd through the process?

## Appendix: How GNU Assembler mastered fragments decades ago

After dedicating several paragraphs to explaining the historical shortcomings of LLVM MC's fragment representation, a natural question arises: how does GNU Assembler (GAS), arguably the other most popular assembler on Linux systems, approach fragment handling?

Delving into its history reveals a fascinating answer.
The earliest commit I could locate is a cvs2svn-generated record from April 1991.
Given the 1987 copyright notice within the code, it's highly probable that this foundational work on fragments was laid down as early as 1987.

You can explore this initial structure in as.h here:
<https://github.com/bminor/binutils-gdb/commit/3a69b3aca678a3caf3ade7f9d42d18233b097ec6#diff-0771d3312685417eb5061a8f0856da4f0406ca8bd6c7d68b6a50a026a4e48c9dR212>.
Please check out `as.h` and `frags.c`.

Observing the `frag` struct, a few points stand out:

* While the exact purpose of `fr_offset` isn't immediately clear to me, `fr_fix` and `fr_var` bear a striking resemblance to the concepts we've recently introduced in MCFragment.
  It might make the variable-size content immediately follow the fixed-size content, though.
* The `char fr_literal[1]` demonstrates an early use of what we now call a flexible array member. Today, GCC and Clang's `-fstrict-flex-arrays=2` would report a warning.
* `fr_symbol` could be more appropriately placed within a union
* `fr_pcrel_adjust` and `fr_bsr` would ideally be architecture-specific data.
* Fragments are allocated using [obstacks](https://www.gnu.org/software/libc/manual/html_node/Obstacks.html), which appear to be a more sophisticated form of a bump allocator, with additional bookkeeping overhead.

But truly, I should stop the minor nit-picking. What astonishingly impresses me is the sheer foresight demonstrated in GAS's fragment allocator design.
Conceived in 1987 or even earlier, it masterfully anticipated solutions that LLVM MC, first conceived in 2009, has only now achieved decades later.
This design held the lead on fragment architecture for nearly four decades!

My greatest tribute goes to the original authors of GNU Assembler for this remarkable piece of engineering.

```c
/*
 * A code fragment (frag) is some known number of chars, followed by some
 * unknown number of chars. Typically the unknown number of chars is an
 * instruction address whose size is yet unknown. We always know the greatest
 * possible size the unknown number of chars may become, and reserve that
 * much room at the end of the frag.
 * Once created, frags do not change address during assembly.
 * We chain the frags in (a) forward-linked list(s). The object-file address
 * of the 1st char of a frag is generally not known until after relax().
 * Many things at assembly time describe an address by {object-file-address
 * of a particular frag}+offset.

 BUG: it may be smarter to have a single pointer off to various different
notes for different frag kinds. See how code pans 
 */
struct frag                        /* a code fragment */
{
        unsigned long fr_address; /* Object file address. */
        struct frag *fr_next;        /* Chain forward; ascending address order. */
                                /* Rooted in frch_root. */

        long fr_fix;        /* (Fixed) number of chars we know we have. */
                                /* May be 0. */
        long fr_var;        /* (Variable) number of chars after above. */
                                /* May be 0. */
        struct symbol *fr_symbol; /* For variable-length tail. */
        long fr_offset;        /* For variable-length tail. */
        char        *fr_opcode;        /*->opcode low addr byte,for relax()ation*/
        relax_stateT fr_type;   /* What state is my tail in? */
        relax_substateT        fr_subtype;
                /* These are needed only on the NS32K machines */
        char        fr_pcrel_adjust;
        char        fr_bsr;
        char        fr_literal [1];        /* Chars begin here. */
                                /* One day we will compile fr_literal[0]. */
};
```
