layout: post
title: Metadata sections, COMDAT and SHF_LINK_ORDER
author: MaskRay
tags: [llvm]
---

## Metadata sections

Many compiler options intrument or annotate text sections, and need to create a metadata section for every candidate text section.
Such metadata sections have the following property:

<!-- more -->

* All relocations from the metadata section reference the associated text section or (if present) the associated auxiliary metadata sections.

In many applications a metadata section does not need other auxiliary sections.

Without inlining (discussed in detail later), many sections additionally have this following property:

* The metadata section is only referenced by the associated text section or not referenced at all.

Below is an example:

```asm
.section .text.foo,"ax",@progbits

.section .meta.foo,"a",@progbits
.quad .text.foo-.   # PC-relative relocation
```

Real world examples include:

* <a id="nonalloc"></a> non-`SHF_ALLOC`: `.debug_*` (DWARF debugging information), `.stack_sizes` (stack sizes)
* <a id="alloc-reloc"></a> `SHF_ALLOC`, not referenced via relocation by code: `.eh_frame` (unwind table), `.gcc_except_table` (language-specific data area for exception handling), `__patchable_function_entries` (`-fpatchable-function-entry=`)
* <a id="alloc-noreloc"></a> `SHF_ALLOC`, referenced via relocation by code: `__llvm_prf_cnts`/`__llvm_prf_data` (`clang -fprofile-generate/-fprofile-instr-generate`), `__sancov_bools` (`clang -fsanitize-coverage=inline-bool-flags`), `__sancov_cntrs` (`clang -fsanitize-coverage=inline-8bit-counters`), `__sancov_guards` (`clang -fsanitize-coverage=trace-pc-guard`)

Non-`SHF_ALLOC` metadata sections need to use absolute relocation types. There is no program counter concept for a section not loaded into memory, so PC-relative relocations cannot be used.

```asm
# Without 'w', text relocation.
.section .meta.foo,"",@progbits
.quad .text.foo      # link-time constant
# Absolute relocation types have different treatment in SHF_ALLOC and non-SHF_ALLOC sections.
```

For `SHF_ALLOC` sections, PC-relative relocations are recommended. If absolute relocations (with the width equaling the word size) are used, `R_*_RELATIVE` dynamic relocations will be produced and the section needs to be writable.

```asm
.section .meta.foo,"a",@progbits
.quad .text.foo-.    # link-time constant

# Without 'w', text relocation.
.section .meta.foo,"aw",@progbits
.quad .text.foo      # R_*_RELATIVE dynamic relocation if -pie or -shared
```

### C identifier name sections

The runtime usually needs to access all the metadata sections.
Metadata section names typically consist of pure C-like identifier characters (isalnum characters in the C locale plus `_`) to leverage a linker magic.
Let's use the section name `meta` as an example.

* If `__start_meta` is not defined, the linker defines it to the start of the output section `meta`.
* If `__stop_meta` is not defined, the linker defines it to the end of the output section `meta`.

`__start_meta` and `__stop_meta` are sometimed called encapsulation symbols.

Note: C11 7.1.3 `[Reserved identifiers]` says

> All identifiers that begin with an underscore and either an uppercase letter or another underscore are always reserved for any use.
>
> No other identifiers are reserved. If the program declares or defines an identifier in a context in which it is reserved (other than as allowed by 7.1.4), or defines a reserved identifier as a macro name, the behavior is undefined.

Clang `-Wreserved-identifier` warns for the usage. That said, compilers don't punish you for the undefined behavior.

### Garbage collection on metadata sections

Users want GC for metadata sections: if `.text.foo` is retained, `meta` (for `.text.foo`) is retained; if `.text.foo` is discarded, `meta` is discarded.
There are three use cases:

* If `meta` does not have the `SHF_ALLOC` flag, it is usually retained under `--gc-sections`. <a href="#alloc">{alloc}</a>
* If `meta` has the `SHF_ALLOC` flag and `.text.foo` does not reference `meta`, `meta` will be discarded, because `meta` is not referenced by other sections (prerequisite). <a href="#nonalloc-noreloc">{nonalloc-noreloc}</a>
* If `meta` has the `SHF_ALLOC` flag and `.text.foo` references `meta`, traditional GC semantics work as intended. <a href="#nonalloc-reloc">{nonalloc-reloc}</a>

The first case is undesired, because the metadata section is unnecessarily retained.
The second case has a more serious correctness issue.

To make the two cases work, we can place `.text.foo` and `meta` in a section group.
If `.text.foo` is already in a COMDAT group, we can place `meta` into the same group; otherwise we can create a non-COMDAT section group (LLVM>=13.0.0, `comdat noduplicates` support for ELF).

```asm
# Zero flag section group
.section .text.foo,"aG",@progbits,foo
.globl foo
foo:

.section .meta.foo,"a?",@progbits
.quad .text.foo-.


# GRP_COMDAT section group, common with C++ inline functions and template instantiations
.section .text.foo,"aG",@progbits,foo,comdat
.globl foo
foo:

.section .meta.foo,"a?",@progbits
.quad .text.foo-.
```

A section group requires an extra section header (usually named `.group`), which requires 40 bytes on ELFCLASS32 platforms and 64 bytes on ELFCLASS64 platforms.
The size overhead is concerning in many applications, so people were looking for better representations.
(AArch64 and x86-64 define ILP32 ABIs and use ELFCLASS32, but technically they can use ELFCLASS32 for small code model with regular ABIs, if the kernel allows.)

Another approach is `SHF_LINK_ORDER`. There are separate chapters introducing section groups (COMDAT) and `SHF_LINK_ORDER` in this article.

### Metadata sections referenced by text sections

Let's discuss the third case in detail. We have these conditions:

* The metadata sections have the `SHF_ALLOC` flag.
* The metadata sections have a C identifier name, so that the runtime can collect them via `__start_`/`__stop_` symbols.
* Each text section references a metadata section.

Since the runtime uses `__start_`/`__stop_`, `__start_`/`__stop_` references are present in a live section.

Now let's introduce the unfortunate special rule about `__start_`/`__stop_`:

* If a live section has a `__start_meta` or `__stop_meta` reference, all `meta` input section will be retained by `ld.bfd --gc-sections`.
  Yes, all, even if the input section is in a different object file.

```asm
# a.s
.global _start
.text
_start:
  leaq __start_meta(%rip), %rdi
  leaq __stop_meta(%rip), %rsi

.section meta,"a"
.byte 0

# b.s
.section meta,"a"
.byte 1
```

`a.o:(meta)` and `b.o:(meta)` are not referenced via regular relocations. Nevertheless, they are retained by the `__start_meta` reference.
(The `__stop_meta` reference can retain the sections as well.)

Now, it is natural to ask: how can we make GC for `meta`?

In ld.lld<=12, the user can set the `SHF_LINK_ORDER` flag, because the rule is refined:

> __start_/__stop_ references from a live input section retains all non-SHF_LINK_ORDER C identifier name sections.

(Example `SHF_LINK_ORDER` C identifier name sections: `__patchable_function_entries` (`-fpatchable-function-entry`), `__sancov_guards` (`clang -fsanitize-coverage=trace-pc-guard`, before clang 13))

In ld.lld>=13, the user can also use a section group, because the rule is further refined:

> __start_/__stop_ references from a live input section retains all non-SHF_LINK_ORDER non-SHF_GROUP C identifier name sections.

GNU ld does not implement the refinement ([PR27259](https://sourceware.org/bugzilla/show_bug.cgi?id=27259)).

A section group has size overhead, so `SHF_LINK_ORDER` may be attempting.
However, it ceases to be a solution when inlining happens.
Let's walk through an example demonstrating the problem.

Our first design uses a plain `meta` for each text section.
We use `,unique` (binutils 2.35) to keep separate sections, otherwise the assembler would combine `meta` into a monolithic section.
```asm
# Monolithic meta.
.globl _start
_start:
  leaq __start_meta(%rip), %rdi
  leaq __stop_meta(%rip), %rsi
  call bar

.section .text.foo,"ax",@progbits
.globl foo
foo:
  leaq .Lmeta.foo(%rip), %rax
  ret

.section .text.bar,"ax",@progbits
.globl bar
bar:
  call foo
  leaq .Lmeta.bar(%rip), %rax
  ret

.section meta,"a",@progbits,unique,0
.Lmeta.foo:
  .byte 0

.section meta,"a",@progbits,unique,1
.Lmeta.bar:
  .byte 1
```

The `__start_meta`/`__stop_meta` references retain `meta` sections, so we add the `SHF_LINK_ORDER` flag to defeat the rule.
Note: we can omit `,unique` because sections with different linked-to sections are not combined by the assembler.

```asm
.section meta,"ao",@progbits,foo
.Lmeta.foo:
  .byte 0

.section meta,"ao",@progbits,bar
.Lmeta.bar:
  .byte 1
```

This works as long as inlining is not concerned.

However, in many instrumentations, the metadata references are created before inlining.
With LTO, if the instrumentation is preformed before LTO, inlining can naturally happen after instrumentation.
If foo is inlined into bar, the `meta` for `.text.foo` may get a reference from another text section `.text.bar`, breaking an implicit assumption of `SHF_LINK_ORDER`: a `SHF_LINK_ORDER` section can only be referenced by its linked-to section.
```asm
# Both .text.foo and .text.bar reference meta.
.section .text.foo,"ax",@progbits
.globl foo
foo:
  leaq .Lmeta.foo(%rip), %rax
  ret

.section .text.bar,"ax",@progbits
.globl bar
bar:
  leaq .Lmeta.foo(%rip), %rax
  leaq .Lmeta.bar(%rip), %rax
  ret
```

Remember that `_start` calls `bar` but not `foo`, `.text.bar` (caller) will be retained while `.text.foo` (callee) will be discarded.
The `meta` for `foo` will link to the discarded `.text.foo`.
This will be recjected by linkers. ld.lld will report: `{{.*}}:(meta): sh_link points to discarded section {{.*}}:(.text.foo)`.

#### Reflection

Here is the history behind the GNU ld rule.

* In 2006-10, [BZ3400](https://sourceware.org/bugzilla/show_bug.cgi?id=3400) reported that stdio flushing did not work with static linking && `--gc-sections`.
* In 2010-01, the problem was re-raised and gold got the rule (<https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=f1ec9ded5c740c22735843025e5d3a8ff4c4079e>).
* [PR11133#c13](https://sourceware.org/bugzilla/show_bug.cgi?id=11133#c13) installed a rule for GNU ld, but it did not appear to work or did not make `__start_meta` in `a.o` retain `meta` in `b.o`.
* In 2015-10, the rule was properly installed ([PR19161](https://sourceware.org/bugzilla/show_bug.cgi?id=19161) [PR19167](https://sourceware.org/bugzilla/show_bug.cgi?id=19167)).

ld.lld had dropped the behavior for a while until r294592 restored it. ld.lld refined the rule by excluding `SHF_LINK_ORDER`.

I am with Alan Modra in a 2010 comment:

> I think this is a glibc bug.  There isn't any good reason why a reference to a __start_section/__stop_section symbol in an output section should affect garbage collection of input sections, except of course that it works around this glibc --gc-sections problem.  I can imagine other situations where a user has a reference to __start_section but wants the current linker behaviour.

Anyhow, GNU ld installed a workaround and made it apply to all C identifier name sections, not just the glibc sections.

Making each `meta` part of a zero flag section group can address this problem, but why do we need a section group to work around a problem which should not exist?
I added `-z start-stop-gc` to ld.lld so that we can drop the rule entirely ([D96914](https://reviews.llvm.org/D96914)).
In [PR27451](https://sourceware.org/bugzilla/show_bug.cgi?id=27451), Alan Modra and I implemented `ld.bfd -z start-stop-gc`.

Due to [PR27491](https://sourceware.org/bugzilla/show_bug.cgi?id=27491), in a `-shared` link, `__start_meta` undefined weak references may get spurious ``relocation R_X86_64_PC32 against undefined protected symbol `__start_meta' can not be used when making a shared object`` if all `meta` sections are discarded.

* In 2021-04, the glibc bug <https://sourceware.org/PR27492> was fixed by me.
* In 2021-04, ld.lld defaulted to `-z start-stop-gc` but recognized `__libc_` sections as a workaround for glibc `libc.a`.

#### What if all metadata sections are discarded?

You may see this: `error: undefined symbol: __start_meta` (ld.lld) or ``undefined reference to `__start_meta'`` (GNU ld).

One approach is to use undefined weak symbols:
```c
__attribute__((weak)) extern const char __start_meta[], __stop_meta[];
```

Another is to ensure there is at least one live metadata section, by creating an empty section in the runtime.
In binutils 2.36, GNU as introduced the flag `R` to represent `SHF_GNU_RETAIN` on FreeBSD and Linux emulations.
I have added the support to LLVM integrated assembler and allowed the syntax on all ELF platforms.
```asm
.section meta,"aR",@progbits
```

With GCC>=11 or Clang>=13 (<https://reviews.llvm.org/D97447>), you can write:
```c
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wattributes"
__attribute__((retain,used,section("meta")))
static const char dummy[0];
#pragma GCC diagnostic pop
```

In a macro, you may use:

```c
_Pragma("GCC diagnostic push")
_Pragma("GCC diagnostic ignored \"-Wattributes\"")
...
_Pragma("GCC diagnostic pop")
``

`-Wattributes` ignores warnings for older compilers.
The `used` attribute, when attached to a function or variable definition, indicates that there may be references to the entity which are not apparent in the source code.
On COFF and Mach-O targets (Windows and Apple platforms), the `used` attribute prevents symbols from being removed by linker section GC.
On ELF targets, GNU ld/gold/ld.lld may remove the definition if it is not otherwise referenced.

The `retain` attributed was introduced in GCC 11 to set the `SHF_GNU_RETAIN` flag on ELF targets.

GCC 11 has an open issue that `__has_attribute(retain)` returning 1 does not guarantee `SHF_GNU_RETAIN` is set (<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99587>).


The typical solution before `SHF_GNU_RETAIN` is:
```c
asm(".pushsection .init_array,\"aw\",@init_array\n" \
    ".reloc ., R_AARCH64_NONE, meta\n"              \
    ".popsection\n")
```

This idea is that `SHT_INIT_ARRAY` sections are GC roots. An empty `SHT_INIT_ARRAY` does not change the output.
The artificial reference keeps `meta` live.

I added `.reloc` support for `R_ARM_NONE/R_AARCH64_NONE/R_386_NONE/R_X86_64_NONE/R_PPC_NONE/R_PPC64_NONE` in LLVM 9.0.0.

## COMDAT

See [COMDAT and section group](/blog/2021-07-25-comdat-and-section-group).

## SHF_LINK_ORDER

In [a generic-abi thread](https://groups.google.com/g/generic-abi/c/_CbBM6T6WeM), Cary Coutant initially suggested to use a new section flag `SHF_ASSOCIATED`.
HP-UX and Solaris folks objected to a new generic flag.
Cary Coutant then discussed with Jim Dehnert and noticed that the existing (rare) flag `SHF_LINK_ORDER` has semantics closer to the metadata GC semantics,
so he intended to replace the existing flag `SHF_LINK_ORDER`.
Solaris had used its own `SHF_ORDERED` extension before it migrated to the ELF simplification `SHF_LINK_ORDER`.
Solaris is still using `SHF_LINK_ORDER` so the flag cannot be repurposed.
People discussed whether `SHF_OS_NONCONFORMING` could be repurposed but did not take that route: the platform already knows whether a flag is unknown and knowing a flag is non-conforming does not help produce better output.
In the end the agreement was that `SHF_LINK_ORDER` gained additional metadata GC semantics.

The new semantics:

> This flag adds special ordering requirements for link editors. The requirements apply to the referenced section identified by the sh_link field of this section's header. If this section is combined with other sections in the output file, the section must appear in the same relative order with respect to those sections, as the referenced section appears with respect to sections the referenced section is combined with.
>
> A typical use of this flag is to build a table that references text or data sections in address order.
>
> In addition to adding ordering requirements, SHF_LINK_ORDER indicates that the section contains metadata describing the referenced section.  When performing unused section elimination, the link editor should ensure that both the section and the referenced section are retained or discarded together. Furthermore, relocations from this section into the referenced section should not be taken as evidence that the referenced section should be retained.

Actually, ARM EHABI has been using `SHF_LINK_ORDER` for index table sections `.ARM.exidx*`.
A `.ARM.exidx` section contains a sequence of 2-word pairs. The first word is 31-bit PC-relative offset to the start of the region. The idea is that if the entries are ordered by the start address, the end address of an entry is implicitly the start address of the next entry and does not need to be explicitly encoded.
For this reason the section uses `SHF_LINK_ORDER` for the ordering requirement.
The GC semantics are very similar to the metadata sections'.

So the updated `SHF_LINK_ORDER` wording can be seen as recognition for the current practice (even though the original discussion did not actually notice ARM EHABI).

In GNU as, before version 2.35, `SHF_LINK_ORDER` could be produced by ARM assembly directives, but not specified by user-customized sections.



## Implementation pitfalls

### Mixed unordered and ordered sections

If an output section consists of only non-`SHF_LINK_ORDER` sections, the rule is clear: input sections are ordered in their input order.
If an output section consists of only `SHF_LINK_ORDER` sections, the rule is also clear: input sections are ordered with respect to their linked-to sections.

What is unclear is how to handle an output section with mixed unordered and ordered sections.
Now, in a non-relocatable link, `SHF_LINK_ORDER` sections are ordered before non-`SHF_LINK_ORDER` sections in an output section (<https://sourceware.org/bugzilla/show_bug.cgi?id=26256>, [D77007](https://reviews.llvm.org/D77007)).

Before, the lld diagnostic `error: incompatible section flags for .rodata` and GNU ld's diagnostic caused a problem if the user wanted to place such input sections along with unordered sections, e.g. `.init.data : { ... KEEP(*(__patchable_function_entries)) ... }` (<https://github.com/ClangBuiltLinux/linux/issues/953>).

Mixed unordered and ordered sections within an input section description was still a problem.
This made it infeasible to add `SHF_LINK_ORDER` to an existing metadata section and expect new object files linkable with old object files which do not have the flag.
I asked how to resolve this upgrade issue and Ali Bahrami responded:

> The Solaris linker puts sections without SHF_LINK_ORDER at the end of the output section, in first-in-first-out order, and I don't believe that's considered to be an error.

So I went ahead and implemented a similar rule for ld.lld: [D84001](https://reviews.llvm.org/D84001) allows arbitrary mix and places `SHF_LINK_ORDER` sections before non-`SHF_LINK_ORDER` sections.

### If the linked-to section is discarded due to compiler optimizations

We decided that the integrated assembler allows `SHF_LINK_ORDER` with sh_link=0 and ld.lld can handle such sections as regular unordered sections (<https://reviews.llvm.org/D72904>).

### If the linked-to section is discarded due to `--gc-sections`

You will see `error: ... sh_link points to discarded section ...`.

A `SHF_LINK_ORDER` section has an assumption: it can only be referenced by its linked-to section.
Inlining and the discussed `__start_` rule can break this assumption.

### Others

* During `--icf={safe,all}`, `SHF_LINK_ORDER` sections are not eligible (conservative but working).
* In relocatable output, `SHF_LINK_ORDER` sections cannot be combined by name.
* When comparing two input sections with different linked-to output sections, use vaddr of output sections instead of section indexes. Peter Smith fixed this in <https://reviews.llvm.org/D79286>.

## Case study

### `-fpatchable-function-entry=`

A function section has a metadata section. No inlining.

`SHF_LINK_ORDER` is the perfect solution. A section group can be used, but that just adds size overhead.

### `clang -fprofile-generate` and `-fprofile-instr-generate`

A function needs `__llvm_prf_cnts`, `__llvm_prf_data` and in some cases `__llvm_prf_vals`. Inlining may happen.

A function references its `__llvm_prf_cnts` and may reference its `__llvm_prf_data` if value profiling applies.
The `__llvm_prf_data` references the text section, the associated `__llvm_prf_cnts` and the associated `__llvm_prf_vals`.

Because the `__llvm_prf_cnts` and the `__llvm_prf_data` may be referenced by more than one text section, `SHF_LINK_ORDER` is not a solution.
We need to place the `__llvm_prf_cnts`, the `__llvm_prf_data` and (if present) the `__llvm_prf_vals` in one section group so that they will be retained or discarded as a unit.
If the text section is already in a COMDAT group, we can reuse the group; otherwise we need to create a zero flag section group and optionally place the text section into the group.
LLVM from 13.0.0 onwards will use a zero flag section group.

Note: due to the `__start_` reference rule and the fact that the `__llvm_prf_data` references the text section, with GNU ld and gold all instrumented text sections cannot be discarded.
There can be a huge size bloat.
If you use GNU ld>=2.37, you can try `-z start-stop-gc`.

For Windows, the cnts section is named `.lprfc$M` and the data section is named `.lprfd$M`.
The garbage collection story is unfortunate.

If an `IMAGE_COMDAT_SELECT_ASSOCIATIVE` section defines an external symbol, MSVC link.exe may report a spurious duplicate symbol error (`error LNK2005`), even if the associative section would be discarded after handling the leader symbol.
lld-link doesn't have this limitation.
However, a portable implementation needs to work around MSVC link.exe.

For a COMDAT `.lprfd$M`, its symbol must be external (linkonce_odr), otherwise references to a non-prevailing symbol would cause an error.
Due to the limitation, `.lprfd$M` has to reside in its own COMDAT, no sharing with `.lprfc$M`.

Different COMDAT groups mean that the liveness of one `.lprfc$M` does not make its associative `.lprfd$M` live.
Since a `.lprfd$M` may be unreferenced, we have to conservatively assume all COMDAT `.lprfd$M` live.
Since `.lprfc$M` input sections parallel `.lprfd$M` input sections, we have to conservatively assume all COMDAT `.lprfc$M` live.
For an external symbol, we use a `/INCLUDE:` directive in `.drectve` to mark it as a GC root.
As a result, `.drectve` may have many `/INCLUDE:` directives, just to work around the link.exe limitation.

Note: for ELF we can use `R_*_NONE` to establish an artificial dependency edge between two sections.
I don't think PE-COFF provides a similar feature.

### `clang -fsanitize-coverage=`

### `clang -fexperimental-sanitize-metadata=`

`clang -fexperimental-sanitize-metadata=atomics` instruments functions and creates `!pcsections` metadata for functions and atomic instructions.
The address of an instrumented atomic instruction is recorded in a section named `sanmd_atomics`.
The `sanmd_atomics` section has the `SHF_LINK_ORDER` flag and links to the text section.

## Miscellaneous

Arm Compiler 5 splits up DWARF Version 3 debug information and puts these sections into comdat groups.
On "monolithic input section handling", Peter Smith commented that:

> We found that splitting up the debug into fragments works well as it permits the linker to ensure that all the references to local symbols are to sections within the same group, this makes it easy for the linker to remove all the debug when the group isn't selected.
>
> This approach did produce significantly more debug information than gcc did. For small microcontroller projects this wasn't a problem. For larger feature phone problems we had to put a lot of work into keeping the linker's memory usage down as many of our customers at the time were using 32-bit Windows machines with a default maximum virtual memory of 2Gb.

COMDAT sections have size overhead on extra section headers. Developers may be tempted to decrease the overhead with `SHF_LINK_ORDER`.
However, the approach does not work due to the ordering requirement. Considering the following fragments:

```text
header [a.o common]
- DW_TAG_compile_unit [a.o common]
-- DW_TAG_variable [a.o .data.foo]
-- DW_TAG_namespace [common]
--- DW_TAG_subprogram [a.o .text.bar]
--- DW_TAG_variable [a.o .data.baz]
footer [a.o common]
header [b.o common]
- DW_TAG_compile_unit [b.o common]
-- DW_TAG_variable [b.o .data.foo]
-- DW_TAG_namespace [common]
--- DW_TAG_subprogram [b.o .text.bar]
--- DW_TAG_variable [b.o .data.baz]
footer [b.o common]
```

`DW_TAG_*` tags associated with concrete sections can be represented with `SHF_LINK_ORDER` sections.
After linking the sections will be ordered before the common parts.

On Mach-O, ld64 define `section$start$__DATA$__data` and `section$end$__DATA$__data` which are similar to `__start_`/`__stop_`.
ld64's behavior is similar to `ld.lld -z start-stop-gc`.
