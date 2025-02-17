layout: post
title: SECTIONS and OVERWRITE_SECTIONS
author: MaskRay
tags: [binutils,llvm,linker]
---

The main task of a linker script is to define extra symbols and define how input sections map into output sections.
It has other miscellaneous features which can be implemented via command line options:

* The `ENTRY` command can be replaced by `--entry`.
* The `OUTPUT_FORMAT` command can usually be replaced by `-m`.
* The `SEARCH_DIRS` command can be replaced by `-L`.
* The `VERSION` command can be replaced by `--version-script`.
* The `INPUT` and `GROUP` commands can add other files as input. This provides a mechanism to split an archive/shared object into multiple files.

<!-- more -->

This article focuses on the `SECTIONS` command. Here is the syntax:

```text
SECTIONS {
  section-command
  section-command
  ...
} [INSERT [AFTER|BEFORE] anchor_section;]
```

The `INSERT` part (in the bracket) is optional.

Each `section-command` can be a symbol assignment, an output section description, or an overlay description.
The linker processes `section-command`s in order and maintains the current location (which can be referenced via `.`).

`a = .;` is a symbol assignment. It defines a symbol called `a` at the current location.
If the symbol is also defined by a relocatable object file or an extracted archive member, the symbol assignment takes precedence.
(In GNU ld, linker scripts and archives are processed in order, so a symbol assignment can suppress the extraction of a subsequent archive member.)

## Output section descriptions

An output section description defines an output section.
An output section represents a section in the linker output.
A simple output section description is `.text : {}` where the section name is mandatory.
Section attributes (type, `AT`, `ALIGN`, VMA memory region, LMA memory region, etc) are optional:

```text
section [address] [(type)] : [AT(lma)] [ALIGN(section_align)] [SUBALIGN](subsection_align)] {
  output-section-command
  ...
} [>region] [AT>lma_region] [:phdr ...] [=fillexp] [,]
```

An output section description consists of multiple `output-section-command`s. An `output-section-command` is one of:

* a symbol assignment
* an input section description
* a `BYTE, SHORT, LONG, QUAD` command
* an output section keyword

Input section descriptions and symbol assignments are common.

`.text : { *(.text .text.*) BYTE(0) a = .; }` is an example involving an input section description, a data command, and a symbol assignment.
`BYTE(0)` and `a = .;` are pretty self-descriptive.

### Input section descriptions

An input section description consists of a file name pattern and a list of space-separated section patterns.
The most common file name pattern is the wildcard `*`.
Section patterns may be exact names (`.text`) or a glob (`.text.*`).

The section patterns are unordered: `*(.text .text.*)` does not place an ordering requirement on `.text` and `.text.*`.
If the linker sees a `.text.a` before a `.text`, it will place the `.text.a` before the `.text`.

The `KEEP` keyword can be used to retain some input sections, e.g. `.retain : { KEEP(*(.retain)) }`.

### Symbol assignments

Symbol assignments can be inside an output section description.
They are commonly used to define encapsulation symbols, e.g. `foo : { __start_foo = .; *(foo); __stop_foo = .; }`.

An anti-pattern is:

```text
__start_foo = .;
foo : { *(foo) }
__stop_foo = .;
```

This is problematic because the linker's orphan section placement rules may place orphans between `__start_foo` and `foo`, breaking the intention of `__start_foo`:

```text
__start_foo = .;
/* Conceptually */ bar : { *(bar) }
foo : { *(foo) }
__stop_foo = .;
```

As an example:

```text
# RUN: split-file %s %t
# RUN: cc -c %t/a.s -o %t/a.o
# RUN: ld.lld %t/lds %t/a.o

#--- a.s
.section bar,"ax"; .byte 0
.section foo,"aw"; .byte 0

#--- lds
sections {
  . = SIZEOF_HEADERS;

  __start_foo = .;
  foo : {}
  __stop_foo = .;
}
```

The orphans and `foo` have different ranks.
The linker does have a special rule to skip non-dot assignments like `__stop_foo` after an output section description.
So sections of the same rank are usually benign.

## Orphan sections

Orphan sections are sections present in an input file which are not mapped by an output section description.
The linker will find or create a suitable output section to contain such orphan sections.
GNU ld, gold, and ld.lld have different rules.
`ld/ldlang.c:lang_insert_orphan` describes GNU ld's rules.

In ld.lld, in the absence of a `SECTIONS` command, there are default rules mapping `.text.*` into `.text`.
After the output section name is fixed, ld.lld tries to place it in a suitable place.
Every output section is assigned a rank. The important ranks are:

* `.interp`
* `SHT_NOTE`
* read-only (non-`SHF_WRITE` non-`SHF_EXECINSTR`)
* `SHF_EXECINSTR`
* `SHF_WRITE` (RELRO, `SHF_TLS`)
* `SHF_WRITE` (RELRO, non-`SHF_TLS`)
* `SHF_WRITE` (non-RELRO)

If the output section for the orphan shares the rank with some output section descriptions, the output section will be placed at the end of these output section descriptions.
Otherwise, an output section description whose rank is the closest to the incoming one will be picked.
If the incoming section has a lower rank, ld.lld will place it just there, otherwise ld.lld will skip some contiguous output sections with the same rank and then non-`.` symbol assignments.

## `INSERT BEFORE` and `INSERT AFTER`

We have delved into the details of output section descriptions for a while.
Let's return to the optional `INSERT` keyword for which we omitted a description.
This feature gives us a way to reorder output sections.

```text
SECTIONS {
  .foo : { *(.foo) }
  .bar : { *(.bar) }
} INSERT AFTER .bss;
```

`.bss` can be defined by an output section description or implicitly defined by orphan section placement.
I added the orphan section support in [D74375](https://reviews.llvm.org/D74375) (target: ld.lld 11.0.0).

This is cool for CUDA sections like `.nv_fatbin`: by moving them after `.bss` we can mitigate `R_X86_64_PC32` relocation overflows for large executables.

While the documentation is not clear, I think for `INSERT AFTER` the output section descriptions should be in order, i.e. `.bss .foo .bar` instead of `.bss .bar .foo`.
I recently fixed this in [D105158](https://reviews.llvm.org/D105158) (target: ld.lld 13.0.0).

## Internal linker script vs external linker script

GNU ld has the concept of an internal linker script.
Many layout rules are implemented in a file under `ldscripts/` installed along with `ld.bfd` (e.g. `/usr/lib/x86_64-linux-gnu/ldscripts/elf_x86_64.*` on Debian).

GNU ld picks a linker script according to whether certain command line options are enabled (e.g. `-z combreloc`, `-z separate-code`, `-z now`).
Note that some options may be enabled at configure time. For example `-z combreloc` is usually the default. `-z separate-code` is the default on Linux x86. Many GCC installations pass `-z relro -z now` to ld by default.

Appending `--verbose` to a GNU ld command line can tell whether an internal linker script or an external one is used.

If you specify a linker script without an option, its `SECTIONS` commands will be appended to the internal linker script's `SECTIONS` command.
Non-`INSERT` concatenated `SECTIONS` are usually not desired.

You may specify `-T` (`--script`) or `-dT` (`--default-script`) to provide an external linker script.
The internal linker script will be completely ignored.

A linker script cannot represent conditions like `-z separate-code`/`-z noseparate-code` distinction, or `-z now`/`-z lazy` distinction.

`-z relro`/`-z norelro` does not affect the internal linker script.

* `-z lazy` internal linker script can be used by `-z norelro` and `-z relro -z now` links.
* `-z now` internal linker script can be used by `-z relro -z now` links but cannot be used by `-z norelro` or `-z relro -z lazy` links.

### gold and ld.lld

For more than two decades GNU ld was the only usable linker on Linux systems.
Then gold was developed as a faster alternative. gold encodes many layout rules in code instead of `ldscripts/` files.
ld.lld inherited this property. IMO This is a superior design.

In the case where no linker script has been provided or every ``SECTIONS`` command is followed by ``INSERT``, ld.lld applies built-in rules which are similar to GNU ld's internal linker scripts.

* Align the first section in a ``PT_LOAD`` segment according to ``-z noseparate-code``,
  ``-z separate-code``, or ``-z separate-loadable-segments``
* Define ``__bss_start``, ``end``, ``_end``, ``etext``, ``_etext``, ``edata``, ``_edata``
* Sort ``.ctors.*``/``.dtors.*``/``.init_array.*``/``.fini_array.*`` and PowerPC64 specific ``.toc``
* Place input ``.text.*`` into output ``.text``, and handle certain variants (``.text.hot.``, ``.text.unknown.``, ``.text.unlikely.``, etc) in the presence of ``-z keep-text-section-prefix``.

ld.lld can read GNU ld's internal linker scripts.

When linking a regular program, you may also supply a minimal linker script like the following:

```text
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

To create a minimal `-nostdlib` program without runtime features like `.init_array`, but with distinct text and data program segments. Use:

```text
PHDRS {
  text PT_LOAD FLAGS(0x5);
  data PT_LOAD FLAGS(0x6);
}

SECTIONS {
  . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
  .text : { *(.text .text.*) } :text
  . = ALIGN(., CONSTANT(MAXPAGESIZE)) + . % CONSTANT(MAXPAGESIZE);
  .data : { *(.data .data.*) } :data
}
```

## `OVERWRITE_SECTIONS`

An `INSERT`-flavored `SECTIONS` command does not suppress built-in layout rules, so it is useful to extend an existing internal linker script.
However, there are two properties which are not ideal:

* The output section descriptions impose an order.
* The anchor section (as in `INSERT AFTER where`) must be present.

Sometimes we just want built-in orphan section placement and don't want to specify an order.
The `OVERWRITE_SECTIONS` command is useful in this case.
I proposed it in <https://sourceware.org/bugzilla/show_bug.cgi?id=26404> and implemented it for ld.lld in [D103303](https://reviews.llvm.org/D103303) (target: ld.lld 13.0.0).

This feature is versatile. To list a few usage:

* Use `section : { KEEP(...) }` to retain input sections under GC
* Define encapsulation symbols (start/end) for an output section
* Use `section : ALIGN(...) : { ... }` to overalign an output section (similar to ld64 `-sectalign`)

## Notes on GNU ld

The internal linker script design of GNU ld makes the input section matching performance critical.
Michael Matz introduced a global prefix tree to speed up section selection in 2022. See `resolve_wilds`.
