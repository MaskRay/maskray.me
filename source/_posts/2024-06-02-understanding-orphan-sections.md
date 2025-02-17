layout: post
title: Understanding orphan sections
author: MaskRay
tags: [binutils,elf,llvm]
---

GNU ld's output section layout is determined by a linker script, which can be either internal (default) or external (specified with `-T` or `-dT`).
Within the linker script, `SECTIONS` commands define how input sections are mapped into output sections.

Input sections not explicitly placed by `SECTIONS` commands are termed ["orphan sections"](https://sourceware.org/binutils/docs/ld/Orphan-Sections.html).

> Orphan sections are sections present in the input files which are not explicitly placed into the output file by the linker script. The linker will still copy these sections into the output file by either finding, or creating a suitable output section in which to place the orphaned input section.

GNU ld's default behavior is to create output sections to hold these orphan sections and insert these output sections into appropriate places.

Orphan section placement is crucial because GNU ld's built-in linker scripts, while understanding common sections like `.text`/`.rodata`/`.data`, are unaware of custom sections.
These custom sections should still be included in the final output file.

* Grouping: Orphan input sections are grouped into orphan output sections that share the same name.
* Placement: These grouped orphan output sections are then inserted into the output sections defined in the linker script. They are placed near similar sections to minimize the number of `PT_LOAD` segments needed.

<!-- more -->

### GNU ld's algorithm

GNU ld's orphan section placement algorithm is primarily specified within `ld/ldlang.c:lang_place_orphans` and `ld/ldelf.c:ldelf_place_orphan`.
`lang_place_orphans` is a linker pass that is between `INSERT` processing and `SHF_MERGE` section merging.

The algorithm utilizes a structure (`orphan_save`) to associate desired BFD flags (e.g., `SEC_ALLOC, SEC_LOAD`) with special section names (e.g., `.text, .rodata`) and a reference to the associated output section.
The associated output section is initialized to the special section names (e.g., `.text, .rodata`), if present.

For each orphan section:

* If an output section of the same name is present and `--unique` is not specified, the orphan section is placed in it.
* Otherwise, GNU ld identifies the matching `orphan_save` element based on the section's flags.
* If an associated output section exists related to the `orphan_save` element, the orphan section is placed after it.
* Otherwise, heuristics are applied to place the orphan section after a similar existing section. For example:
  + .text-like sections follow `SHF_ALLOC|SHF_WRITE|SHF_EXECINSTR` sections.
  + .rodata-like sections follow .text-like sections.
  + .tdata-like sections follow .data-like sections.
  + .sdata-like sections follow .data-like sections.
  + .data-like sections can follow .rodata-like sections.
* The associated output section is replaced with the new output section. The next orphan output section of similar flags will be placed after the current output section.

As a special case, if an orphan section is placed after the last output section (`else if (as != snew && as->prev != snew)`), it will be adjusted to be placed after all trailing commands (`sym = expr`, `. = expr`, etc).

For example, custom code section `mytext` (with `SHF_ALLOC | SHF_EXECINSTR`) would typically be placed after `.text`, and custom data section mydata (with `SHF_ALLOC | SHF_WRITE`) after `.data`.

```c
  static struct orphan_save hold[] = {
      { ".text", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_READONLY | SEC_CODE, 0, 0, 0, 0 },
      { ".rodata", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_READONLY | SEC_DATA, 0, 0, 0, 0 },
      { ".tdata", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_DATA | SEC_THREAD_LOCAL, 0, 0, 0, 0 },
      { ".data", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_DATA, 0, 0, 0, 0 },
      { ".bss", SEC_ALLOC, 0, 0, 0, 0 },
      { 0, SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_READONLY | SEC_DATA, 0, 0, 0, 0 },
      { ".interp", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_READONLY | SEC_DATA, 0, 0, 0, 0 },
      { ".sdata", SEC_HAS_CONTENTS | SEC_ALLOC | SEC_LOAD | SEC_DATA | SEC_SMALL_DATA, 0, 0, 0, 0 },
      { ".comment", SEC_HAS_CONTENTS, 0, 0, 0, 0 },
  };
```

Noteworthy details:

`.interp` and `.rodata` have the same BFD flags, but they are anchors for different sections.
`SHT_NOTE` sections go after `.interp`, while other read-only sections go after `.rodata`.

Consider a scenario where a linker script defines `.data` and `.rw1` sections with identical BFD flags.
If we have orphan sections that share the same flags, GNU ld would insert these orphans after `.data`, even if it might seem more logical to place them after `.rw1`.

```
.data : { *(.data .data.*) }
// .rw2 .rw3 orphans are inserted here
.rw1 : { *(.rw1) }
```

Renaming the output section `.data` will achieve the desired placement:

```
.mydata : { *(.data .data.*) }
.rw1 : { *(.rw1) }
// .rw2 .rw3 orphans are inserted here
```

### lld's algorithm

The LLVM linker lld implements a large subset of the GNU ld linker script.
However, due to the complexity of GNU ld and lack of an official specification, there can be subtle differences in behavior.

While lld strives to provide a similar linker script behavior, it occasionally makes informed decisions to deviate where deemed beneficial.
We balance compatibility with practicality and interpretability.

Users should be aware of these potential discrepancies when transitioning from GNU ld to lld, especially when dealing with intricate linker script features.

lld does not have built-in linker scripts. When no `SECTIONS` is specified, all input sections are orphan sections.

**Rank-based sorting**

lld assigns a rank to each output section, calculated using various flags like `RF_NOT_ALLOC, RF_EXEC, RF_RODATA`, etc.
Orphan output sections are then sorted by these ranks.

```cpp
enum RankFlags {
  RF_NOT_ADDR_SET = 1 << 27,
  RF_NOT_ALLOC = 1 << 26,
  RF_PARTITION = 1 << 18, // Partition number (8 bits)
  RF_LARGE_ALT = 1 << 15,
  RF_WRITE = 1 << 14,
  RF_EXEC_WRITE = 1 << 13,
  RF_EXEC = 1 << 12,
  RF_RODATA = 1 << 11,
  RF_LARGE = 1 << 10,
  RF_NOT_RELRO = 1 << 9,
  RF_NOT_TLS = 1 << 8,
  RF_BSS = 1 << 7,
};
```

The important ranks are:

* `.interp`
* `SHT_NOTE`
* read-only (non-`SHF_WRITE` non-`SHF_EXECINSTR`)
* `SHF_EXECINSTR`
* `SHF_WRITE` (RELRO, `SHF_TLS`)
* `SHF_WRITE` (RELRO, non-`SHF_TLS`)
* `SHF_WRITE` (non-RELRO)
* Sections without the `SHF_ALLOC` flag

**Special case: non-alloc sections**

Non-alloc sections are placed at the end.

**Finding the most similar section**

For each orphan section, lld identifies the output section with the most similar rank.
The similarity is determined by counting the number of leading zeros in the XOR of the two ranks.

```cpp
// We want to find how similar two ranks are.
// The more branches in getSectionRank that match, the more similar they are.
// Since each branch corresponds to a bit flag, we can just use
// countLeadingZeros.
static int getRankProximity(OutputSection *a, SectionCommand *b) {
  auto *osd = dyn_cast<OutputDesc>(b);
  return (osd && osd->osec.hasInputSections)
             ? llvm::countl_zero(a->sortRank ^ osd->osec.sortRank)
             : -1;
}
```

When multiple output sections share the maximum similarity with an orphan section, resolving the ambiguity is crucial.
I [refined the behavior for lld 19](https://github.com/llvm/llvm-project/pull/94099): if the orphan section's rank is not lower than the similar sections, the *last* similar section is chosen for placement.

```cpp
  // Find the most similar output section as the anchor. Rank Proximity is a
  // value in the range [-1, 32] where [0, 32] indicates potential anchors (0:
  // least similar; 32: identical). -1 means not an anchor.
  //
  // In the event of proximity ties, we select the first or last section
  // depending on whether the orphan's rank is smaller.
  int maxP = 0;
  auto i = e;
  for (auto j = b; j != e; ++j) {
    int p = getRankProximity(sec, *j);
    if (p > maxP ||
        (p == maxP && cast<OutputDesc>(*j)->osec.sortRank <= sec->sortRank)) {
      maxP = p;
      i = j;
    }
  }
  if (i == e)
    return e;
```

For example, when inserting `.bss` orphan sections (`SHF_ALLOC|SHF_WRITE`, `SHT_NOBITS`), lld should find the last output section that carries the flags/type `SHF_ALLOC|SHF_WRITE` `SHT_PROGBITS`.

```
WA PROGBITS    (not here)
A
WA PROGBITS
AX
WA PROGBITS    (here)
<== WA NOBITS
```

**Placement decision**

The orphan section is placed either before or after the most similar section, based on a complex rule involving:

* The relative ranks of the orphan and similar section.
* The presence of [`PHDRS`](https://sourceware.org/binutils/docs/ld/PHDRS.html) or [`MEMORY`](https://sourceware.org/binutils/docs/ld/MEMORY.html) commands in the linker script.
* Scanning backward or forward through the script for a suitable insertion point.

In essence:

* If the orphan section's rank is lower than the similar section's rank, and no `PHDRS` or `MEMORY` commands exist, it's placed *before* the similar section.
* Otherwise, it's placed *after* the similar section, potentially skipping symbol assignments or output sections without input sections in the process.

```cpp
  auto isOutputSecWithInputSections = [](SectionCommand *cmd) {
    auto *osd = dyn_cast<OutputDesc>(cmd);
    return osd && osd->osec.hasInputSections;
  };

  // If i's rank is larger, the orphan section can be placed before i.
  //
  // However, don't do this if custom program headers are defined. Otherwise,
  // adding the orphan to a previous segment can change its flags, for example,
  // making a read-only segment writable. If memory regions are defined, an
  // orphan section should continue the same region as the found section to
  // better resemble the behavior of GNU ld.
  bool mustAfter = script->hasPhdrsCommands() || !script->memoryRegions.empty();
  if (cast<OutputDesc>(*i)->osec.sortRank <= sec->sortRank || mustAfter) {
    for (auto j = ++i; j != e; ++j) {
      if (!isOutputSecWithInputSections(*j))
        continue;
      if (getRankProximity(sec, *j) != proximity)
        break;
      i = j + 1;
    }
  } else {
    for (; i != b; --i)
      if (isOutputSecWithInputSections(i[-1]))
        break;
  }

  // As a special case, if the orphan section is the last section, put
  // it at the very end, past any other commands.
  // This matches bfd's behavior and is convenient when the linker script fully
  // specifies the start of the file, but doesn't care about the end (the non
  // alloc sections for example).
  if (std::find_if(i, e, isOutputSecWithInputSections) == e)
    return e;

  while (i != e && shouldSkip(*i))
    ++i;
  return i;
```

**Special case: last section**

If the orphan section happens to be the last one, it's placed at the very end of the output, mimicking GNU ld's behavior for cases where the linker script fully specifies the beginning but not the end of the file.

**Special case: skipping symbol assignments**

It is common to surround an output section description with encapsulation symbols.
lld has a special case to not place orphans between `foo` and a following symbol assignment.

Backward scan example:

```
previous_start = .;
previous : { *(previous) } // Found output section with a backward scan
previous_end = .;          // The orphan should be after here

similar : { *(similar) }   // The most similar section found by the first step
```

Forward scan example:

```
similar0 : { *(similar0) }
similar1_start = .;
similar1 : { *(similar1) } // The most similar section found by the first step
similar1_end = .;          // The orphan should be after here
```

However, an assignment to the location counter serves as a barrier to stop the forward scan.

```
previous_start = .;
previous : { *(previous) } // Found output section with a backward scan
previous_end = .;          // The orphan should be after here
symbol = .;                // We conservatively assume any symbol as a probable "end" symbol.
. = ALIGN(CONSTANT(MAXPAGESIZE)); // Barrier

similar : { *(similar) }   // The most similar section found by the first step
```

**Special case: initial location counter**

In addition, if there is a location counter assignment before the first output section, orphan sections cannot be inserted before the initial location counter assignment.
This is to recognize the common pattern that the initial location counter assignments specifies the load address.

```
sym0 = .;
. = initial_location; // Initial assignment to DOT. Orphan sections cannot be inserted before here.
.text : {}            // First output section
```

**Presence of `PHDRS` or `MEMORY`**

The presence of `PHDRS` or `MEMORY` commands disallows lld to place the orphan section before the anchor.
This condition is introduced in two patches:

* [[ELF] Avoid adding an orphan section to a less suitable segment](https://reviews.llvm.org/D111717)
* [[ELF] Better resemble GNU ld when placing orphan sections into memory regions](https://reviews.llvm.org/D112925)

When a linker script defines `PHDRS`, it typically specifies the initial section within each `PT_LOAD` segment.
These sections often have address requirements, indicated by a preceding `. = expr` statement.
If an orphan section is associated with such a section as its anchor, lld avoids inserting the orphan before the anchor to maintain the intended segment structure and address alignment.

For instance, consider this linker script excerpt:
```
PHDRS {
  ...
  rodata PT_LOAD;
}

SECTIONS {
  ...
  . = ALIGN(CONSTANT(MAXPAGESIZE));
  .rodata : { *(.rodata .rodata.*) } : rodata
```

Here, `.rodata` is the first section in a `PT_LOAD` segment, and it's aligned to `MAXPAGESIZE`.
If an orphan section is inserted before `.rodata`, it would inherit the previous segment's flags and break the intended address requirement.

**Program headers propagation**

After orphan section placement, if the `PHDRS` command is specified, lld will propagate program headers to output sections that do not specify `:phdr`.

### Case study

By employing this rank-based approach, lld provides an elegant implementation that does not hard code specific section names (e.g., `.text`/`.rodata`/`.data`).
In GNU ld, if you rename special section names `.text`/`.rodata`/`.data` in the linker script, the output could become subtle different.

**Orphan sections matching an output section name**

The following output section description does not match `.foo` input sections, but `.foo` orphan sections will still be placed inside `.foo`.

```
.foo : { *(.bar) }
```

**Read-only sections**

Among read-only sections (e.g., `.dynsym`, `.dynstr`, `.gnu.hash`, `.rela.dyn`, `.rodata`, `.eh_frame_hdr`, `.eh_frame`), lld prioritizes the placement of `SHT_PROGBITS` sections (`.rodata`, `.eh_frame_hdr`, and `.eh_frame`) closer to code sections.
This is achieved by [assigning them a higher rank](https://reviews.llvm.org/D48406).
The rationale behind this design is to mitigate the risk of relocation overflow in the absence of an explicit linker script.

These non-`SHT_PROGBITS` sections do not contain relocations to code sections and can be placed away from code sections.

```
.dynsym
.gnu.version
.gnu.version_r
.gnu.hash
.dynstr
.rela.dyn
.rela.plt
.rodata         // closer to .text
.eh_frame_hdr   // closer to .text
.eh_frame       // closer to .text
.text
```

If a linker script explicitly includes a `SECTIONS` command specifying `.rodata` without mentioning other read-only sections, orphan sections like `.dynsym` might be placed before `.rodata`.

```
.rodata : { *(.rodata .rodata.*) }
```

This behavior can be further influenced by the presence of `PHDRS` commands.
If an [output section phdr](https://sourceware.org/binutils/docs/ld/Output-Section-Phdr.html) is specified with `.rodata`, orphan sections like `.dynsym` would not be placed before `.rodata`, ensuring that the orphans would not affect the flags of the preceding program header.

```
PHDRS {
  text PT_LOAD;
}

SECTIONS {
  ...
  .rodata : { *(.rodata .rodata.*) } : text
  // .dynsym cannot be placed before .rodata with specified program headers
}
```

**Symbol assignment between two output sections**

A symbol assignment placed between output sections can be interpreted in two ways: as marking the end of the preceding section or the start of the following section.
lld doesn't attempt to guess the intended meaning, leading to potential ambiguity in scenarios like this:

```
.data : { *(.data .data.*) }
bss_start = .;
.bss : { *(.bss .bss.*) }
```

versus

```
.data : { *(.data .data.*) }
data_end = .;
.bss : { *(.bss .bss.*) }
```

In both cases, lld might place `SHF_ALLOC|SHF_WRITE` `SHT_PROGBITS` orphan sections before `.bss`, potentially disrupting the intended behavior if the goal was to mark the start of the .bss section with `bss_start = .`.

To avoid this ambiguity and ensure consistent behavior, the recommended practice is to place symbol assignments within the output section descriptions ([FreeBSD example](https://github.com/freebsd/freebsd-src/commit/6e764e36da019837d90e3b4b712871ee4442637a)):

```
.data : { *(.data) }
.bss : { bss_start = .; *(.data) bss_end = .; }
```

### Portability

To maximize portability of linker scripts across different linkers, it's essential to establish clear boundaries for PT_LOAD segments. This can be achieved by:

* Explicit alignment: Utilizing `MAXPAGESIZE` alignment to distinctly separate sections within the linker script.
* Anchoring sections: Ensuring that the first section in each `PT_LOAD` segment includes at least one input section, preventing ambiguous placement decisions by the linker.
  When the `PHDRS` command is present, ensure that the first sections have `:phdr`.

By adhering to these guidelines, you can reduce reliance on linker-specific orphan section placement algorithms, promoting consistency across GNU ld and lld.

When linking a regular position-dependent executable, you may also supply a minimal linker script like the following for a `-no-pie` link:

```
SECTIONS
{
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;

  . = DATA_SEGMENT_ALIGN (CONSTANT (MAXPAGESIZE), CONSTANT (COMMONPAGESIZE));
  .init_array    :
  {
    PROVIDE_HIDDEN (__init_array_start = .);
    KEEP (*(SORT_BY_INIT_PRIORITY(.init_array.*)))
    KEEP (*(.init_array))
    PROVIDE_HIDDEN (__init_array_end = .);
  }
  .fini_array    :
  {
    PROVIDE_HIDDEN (__fini_array_start = .);
    KEEP (*(SORT_BY_INIT_PRIORITY(.fini_array.*)))
    KEEP (*(.fini_array))
    PROVIDE_HIDDEN (__fini_array_end = .);
  }
  . = DATA_SEGMENT_RELRO_END (0, .);
  .data           : { *(.data .data.*) }
  . = .;
  .bss            : { *(.bss .bss.*) *(COMMON) }
  . = DATA_SEGMENT_END (.);
}
```

A better style may define `.text` and `.rodata` as well.
This linker script works with both GNU ld and lld.

```sh
clang -fuse-ld=bfd -Wl,-T,a.lds -no-pie a.c -o a.lld
clang -fuse-ld=lld -Wl,-T,a.lds -no-pie a.c -o a.bfd
```

### Disabling orphan sections

For projects that require absolute control over section placement, GNU ld version 2.26 and later provides `--orphan-handling=[place|warn|error|discard]`.
This allows you to choose how orphan sections are handled:

* place (default): The linker places orphan sections according to its internal algorithm.
* warn: The linker places orphan sections but also issues warnings for each instance.
* error: The linker treats orphan sections as errors, preventing the linking process from completing.
* discard: The linker discards orphan sections entirely.

### `--unique`

TODO
