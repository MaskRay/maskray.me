layout: post
title: Linker garbage collection
author: MaskRay
tags: [binutils,llvm,linker]
---

A program may have a lot of unused code and data.
There can be many reasons:

<!-- more -->

* There is indeed unused code and data in all code paths.
* The code and data may be unused in some code paths but not all. This is common for libraries where not all usage can exercise all code paths.
* The compiler generates duplicates definitions and expects the linker to deduplicate them, e.g. inline functions, implicit template instantiations, vtables, `type_info` objects. One copy is needed and the others can/must be discarded.

The compiler can detect some unused code and data for point 1 (e.g. `-Wunused`).
However, due to the limitation of seeing just one translation unit, a lot of cases (e.g. unused non-internal linkage definitions) cannot be detected.
The linker, which has visibility to all translation units, is the best place to do this.
Many linkers provide features to discard unused code and data:

* Archive member selection. If an archive member cannot satisfy an undefined reference, it may not be pulled into the link. This avoids all section contribution from the member.
* COMDAT section deduplication
* Section based garbage collection

This article focuses on the last point.
In GNU linkers, the feature is called "garbage collection" (`ld --gc-sections`).
On macOS, ld64 calls it "dead stripping" (`ld -dead_strip`).
On Windows, the documentation says "eliminates functions and data that are never referenced" (`link.exe /OPT:REF`).

Section is the right granule for the feature. In many object file formats, "section" is an inseparable unit.
The garbage collection feature starts with a list of sections which should be retained (GC roots).
For each retained section, the linker marks referenced sections via relocations and associated sections (ELF section group and PE `IMAGE_COMDAT_SELECT_ASSOCIATIVE`).
In the end, unmarked sections are discarded.
If a symbol is defined relative to a discarded section, the symbol is discarded as well.

The following pseudocode describes the process:

```cpp
set visited;
stack work_list;
for each section
  if section is a GC root {
    work_list.push_back(section);
    visited.insert(section);
  }
while (work_list.size()) {
  section sec = work_list.back();
  work_list.pop_back();
  for each section referenced by sec
    if (visited.insert(section))
      work_list.push_back(section);
}
```

A section may reference other sections in several ways.
The most common one is via relocations.
Section `A` contains relocations and a relocation may reference a symbol defined in another section `B`. We say that `A` references `B`.
Section `A` may have dependent sections. The dependent sections may need to be discarded with `A` as a whole.

## ELF

Compile source files with `-ffunction-sections` and `-fdata-sections` to leverage section based GC.

Without the options, the compiler tends to produce monolithic text and data sections.
The large sections make the GC coarse-grained and they will likely be retained as a whole.
`ld --gc-sections` can still discard some sections, but the effect is likely very poor.

Here is an assembly file interleaved with comments describing a GC root and section references via relocations.

```text
# _start is a GC root.
.text
.globl _start
_start:
  # The relocation causes .text.foo to be marked.
  call foo

.section .text.foo,"ax",@progbits
.globl foo
foo:
  # The relocation causes .text.bar to be marked.
  call bar

.section .text.bar,"ax",@progbits
.globl bar
bar:
  ret
```

### `-fno-unique-section-names`

`-ffunction-sections` causes section names like `.text.name`.
`-fdata-sections` causes section names like `.data.name`, `.rodata.name`, `.data.rel.local.name`.
While the names are nice for debugging purposes, there are several issues:

* Input section descriptions in a linker script need to take care of the optional suffix: `*(.data .data.*)`.
* The unique section names inrease the size of the string table.
* Section for optimization purposes unfortunately picked ambiguous names: `.text.exit.*`, `.text.hot.*`, `.text.unlikely.*`, etc. They may collide with some function names.

For the first point, `$` in PE/COFF is a great alternative.

For the second point, Clang provides `-fno-unique-section-names` to save the string table space: stem names such as `.text`, `.data`, `.rodata` are used.
`-fno-unique-section-names` needs to create multiple sections with the same name.
This requires a new assembler syntax: `.section .text,"a",@progbits,unique,1`.
LLVM implemented the syntax first. Then H.J. Lu implemented the syntax for binutils 2.35.

### GNU ld

The following sections are GC roots.

* Sections which define a symbol specified by `-u`, `--entry`, `--init`, or `--fini`
* Sections which define a symbol exported to `.dynsym`
  + The symbol has a visibility of `STV_DEFAULT` or `STV_PROTECTED` and is not localized
  + One of `-shared`, `--export-dynamic`, and `--gc-keep-exported` is specified
  + (Other conditions I am unaware of)
* Sections which have the `SHF_GNU_RETAIN` flag
* `SHT_PREINIT_ARRAY/SHT_INIT_ARRAY/SHT_FINI_ARRAY`
* Non-`SHF_ALLOC` non-`SHF_LINK_ORDER` `SHT_NOTE`
* Linker created sections (`SEC_LINKER_CREATED`)
* Sections which are matched by an input section description with the `KEEP` keyword
* (Other conditions I am unaware of)

GNU ld implements an interesting heuristic: if there is at least one retained non-`SHT_NOTE` `SHF_ALLOC` section in a relocatable object file, GNU ld will mark more sections.
Otherwise, most sections (e.g. `.debug_*`) will be discarded.
(See `bfd/elflink.c:_bfd_elf_gc_mark_extra_sections`)
This heuristic can discard quite a few debug sections when no non-`SHT_NOTE` `SHF_ALLOC` section is retained.

The following sections are discarded as well:

* Non-`SHF_ALLOC` non-`SHF_LINK_ORDER` sections which are not in a section group
  + Except `.debug_line.*` which is associated to a discarded text section
* Group sections which contain just non-`SHF_ALLOC` sections

GNU ld does not support garbage collecting `SHF_MERGE` sections ([PR26622](https://sourceware.org/bugzilla/show_bug.cgi?id=26622)).
If the only references to an as-needed shared object are from discarded sections, the `DT_NEEDED` entry can be omitted. GNU ld does not implement this ([PR24836](https://sourceware.org/bugzilla/show_bug.cgi?id=24836)).

GNU ld does not make RHS of a symbol assignment retain a section ([PR31158](https://sourceware.org/bugzilla/show_bug.cgi?id=31158)).

You can print discarded sections with `--print-gc-sections`.

### ld.lld

The following sections are GC roots.

* Sections which define a symbol specified by `-u`, `--entry`, `--init`, or `--fini`
* Sections which define a symbol exported to `.dynsym`
* Sections which define a symbol referenced by a linker script
* Sections which have the `SHF_GNU_RETAIN` flag
* `SHT_PREINIT_ARRAY/SHT_INIT_ARRAY/SHT_FINI_ARRAY`
* `SHT_NOTE` not within a section group (this rule is for Fedora watermark)
* `.ctors/.dtors/.init/.fini/.jcr`
* Personality routines or language-specific data area referenced by `.eh_frame`. ld.lld handles `--gc-sections` before `.eh_frame` deduplication, so this may retain <!-- more --> sections than needed.
* Sections which are matched by an input section description with the `KEEP` keyword

Non-`SHF_ALLOC` non-`SHF_LINK_ORDER` non-`SHF_REL[A]` non-`SHF_GROUP` sections are retained, albeit not GC roots.

If a live section contains a relocation referencing a non-weak symbol defined in a shared object, the shared object is considered needed (this will lead to a `DT_NEEDED` tag).

When `-z nostart-stop-gc` is in effect, if a live section contains a relocation referencing a symbol `__start_$secname` or `__stop_$secname`, and `$secname` has a C identifier name, all input sections `$secname` will be retained.
See [Metadata sections, COMDAT and SHF_LINK_ORDER](/blog/2021-01-31-metadata-sections-comdat-and-shf-link-order) for detail.

### Debug sections

Linkers do not discard debug sections. This follows the "smart format, dumb linker" philosophy.
DWARF sections are large. Optimizing them would take a significant amount of link time.

There are, however, instances where discarded text sections can lead to some placeholder values (sometimes referred to as tombstone values) in DWARF.

* `.debug_ranges` & `.debug_loc`: 1 (ld.lld<11: 0+addend; GNU ld uses 1 for .debug_ranges)
* `.debug_*`: 0 (ld.lld<11: 0+addend; GNU ld uses 0; future ld.lld: 0xffffffff or 0xffffffffffffffff)

### `SHF_GNU_RETAIN`

In binutils 2.36, GNU as introduced the flag `R` to represent `SHF_GNU_RETAIN` on FreeBSD and Linux emulations.
This is an architecture-independent way to mark a section as a GC root.
I have added the support to LLVM integrated assembler and ld.lld and allowed the syntax on all ELF platforms.
```asm
.section meta,"aR",@progbits
```

With GCC>=11 or Clang>=13 (<https://reviews.llvm.org/D97447>), you can write:
```c
__attribute__((retain,used,section("meta")))
static const char dummy[0];
```

The `used` attribute, when attached to a function or variable definition, indicates that there may be references to the entity which are not apparent in the source code.
On COFF and Mach-O targets (Windows and Apple platforms), the used attribute prevents symbols from being removed by linker section GC.
On ELF targets, GNU ld/gold/ld.lld may remove the definition if it is not otherwise referenced.

The `retain` attributed was introduced in GCC 11 to set the `SHF_GNU_RETAIN` flag on ELF targets.

The typical solution before `SHF_GNU_RETAIN` is:
```c
asm(".pushsection .init_array,\"aw\",%init_array\n" \
    ".reloc ., BFD_RELOC_NONE, meta\n"              \
    ".popsection\n")
```

(`BFD_RELOC_NONE` requires binutils>=2.26.)

This idea is that `SHT_INIT_ARRAY` sections are GC roots. An empty `SHT_INIT_ARRAY` does not change the output.
The `.reloc` directive produces a relocation. `BFD_RELOC_NONE` is special notation which transforms to the architecture dependent `R_*_NONE` relocation type.
The relocation serves as an artificial reference: it does not modify the location but can keep the section defining `meta` live.

I added `.reloc` support for `R_ARM_NONE/R_AARCH64_NONE/R_386_NONE/R_X86_64_NONE/R_PPC_NONE/R_PPC64_NONE` in LLVM 9.0.0 and `BFD_RELOC_NONE` support in LLVM 13.0.0.
If you target older Clang, you can write:

```c
asm(".pushsection .init_array,\"aw\",%init_array\n" \
    ".reloc ., R_AARCH64_NONE, meta\n"              \
    ".popsection\n")
```

See [Metadata sections, COMDAT and SHF_LINK_ORDER](/blog/2021-01-31-metadata-sections-comdat-and-shf-link-order) for metadata section usage.

### `SHF_MERGE` sections

This section flag applies to some data sections with constant contents (integer literals, string literals, etc).
If the section contains data elements of uniform size and the addresses are not significant, duplicate elements can be replaced by one representative element.
The references from other sections can be bound to the representative element.

ld.lld supports garbage collection of unused pieces. GNU ld has not implemented the feature ([PR26622](https://sourceware.org/bugzilla/show_bug.cgi?id=26622)).

### Suppressed undefined symbol errors

ELF linkers do not scan relocations in discarded input sections.
When `--gc-sections` is enabled, certain undefined symbol errors are suppressed.
```
% cat a.c
void bar();
void foo() { bar(); }
int main() { }
% clang -ffunction-sections -Wl,--gc-sections a.c # no error
% clang -ffunction-sections a.c
ld.lld: error: undefined symbol: bar
>>> referenced by a.c
>>>               /tmp/a-cda298.o:(foo)
```

## PE/COFF

PE has a concept "Grouped Sections": if the section name includes `$`, the linker discards `$` and the following characters.
Thus, an object section named `.text$name` actually contributes to the `.text` section in the image.

`.CRT$XCU` is the global initializer section (similar to ELF `SHT_INIT_ARRAY`).
A `.CRT$XCU` in an `IMAGE_COMDAT_SELECT_ASSOCIATIVE` section referencing an `IMAGE_SCN_LNK_COMDAT` text section can be garbage collected.

### lld-link

* Sections which define a symbol specified by `/include:`
* Sections which define a symbol specified by `llvm.used` (in bitcode, the `/include:` workaround happens too late)
* Sections which define a symbol specified by `/export:`
* Sections which define the delay load helper if `/delayload` is specified

### Misc

Unfortunately additional GC roots require linker options.
For a local symbol, there is no good way to retain the definition because `/include:` cannot be used.
Using a unique name COMDAT with its name referenced by a `/include:` directive is a possible workaround.

`__attribute__((used))` prevents the symbol from beging removed by linker section GC.
Due to the above limitation, in LLVM this attribute does not work with local symbols.

## Mach-O

Modern Mach-O is derived from an a.out variant and keeps some severe limitations.
There are some techniques to lift the limitations, but you can see inelegance in the result.
For example, you cannot have more than 255 sections.
Thankfully, there is an interesting feature `.subsections_via_symbols`.
While other object file formats introduce more sections and define a separate symbol for each section, `.subsections_via_symbols` uses a monolithic section and lets symbols split the section into logically independent pieces.
This is a really great invention working around the limitation.

```asm
.section __TEXT,__text,regular,pure_instructions
.globl _a
_a:
  ...

.globl _b
_b:
  ...
```

Because `.subsections_via_symbols` is always enabled, `-ffunction-sections` and `-fdata-sections` are no-op.

When the GNU attribute `__attribute__((used))` is specified on a declaration, the associated symbol gets a `.no_dead_strip` directive to keep it live.
In object files, this bit is `N_NO_DEAD_STRIP`.

The section attribute `no_dead_strip` can be specified on a section to keep it live.
In object files, this bit is `S_NO_DEAD_STRIP`.

The section attribute `live_support` can be specified on a section.
The section is live if any of its referenced section is live.
In object files, this bit is `S_ATTR_LIVE_SUPPORT`.
This attribute is used by `__TEXT,__eh_frame` and can be seen as a generalized `SHF_LINK_ORDER`.

## LLVM IR

The global variable `llvm.used` is an appending linkage array which contains a list of `GlobalValue`s (most are `GlobalObject`s).
If a symbol appears in the list, the compiler, assembler and linker cannot discard it.

The global variable `llvm.compiler.used` is an appending linkage array which contains a list of `GlobalValue`s (most are `GlobalObject`s).
This is similar to `llvm.used`, except that the linker can discard the symbol.

The GNU attribute `__attribute__((used))` lowers to `llvm.compiler.used` on ELF and `llvm.compiler.used` on COFF/Mach-O/wasm.

## Link-time optimization

Code generation options can be categorized into IR generation options (e.g. `-g`) and object file generation options (e.g. `-ffunction-sections -fdata-sections`).
Without LTO, you usually do not see a line between the two because both of them take actions when producing an object file.
With LTO, the distinction becomes clearer: IR generation options have effects on `.c` -> `.ll`/`.bc` (`.o` is often used for bitcode files as well) and object file generation options have effects on `.ll`/`.bc` -> `.o`.
In LLVM, `-ffunction-sections` and `-fdata-sections` are object file generating options and do not affect IR generation.
ld.lld and `LLVMgold.so` enable function sections and data sections automatically.

In LLVM LTO, there is a dead stripping feature which is enabled by default (`-mllvm -compute-dead=true`).
It computes the list of dead symbols (`computeDeadSymbolsWithConstProp`), i.e. not reachable from a retained symbol.
For ThinLTO, this can decrease compile time because of fewer function imports, and improvement on internalization because of fewer imports/exports.
In `thinBackend`, such dead symbols are dropped.

A natural question is why `--gc-sections` is still useful with LTO.

For regular LTO, very few sections generated by the compiler are discarded by `--gc-sections`.
They are not discarded because:

* Some constant references in LLVM IR can be discarded by target specific optimizations (e.g. `memcpy(..., &constant, sizeof(constant));`).
* Due to phase ordering issues some definitions are not discarded by the optimizer.

For ThinLTO there are more sections discarded by `--gc-sections`:

* ThinLTO can cause a definition to be imported to other modules. The original definition may be unneeded after imports.
* The definition may survive after intra-module optimization. After imports, a round of (inter-module) IR optimizations after `computeDeadSymbolsWithConstProp` may make the definition unneeded.
* Symbol resolution is conservative.

Regarding symbol resolution, symbol resolution happens before LTO and LTO happens before `--gc-sections`.
The symbol resolution process may be conservative: it may communicate to LTO that some symbols are referenced by regular object files while in the GC stage the references turn out to not exist because of discarded sections with more precise GC roots.

## Linux kernel

Since v4.10, the `CONFIG_LD_DEAD_CODE_DATA_ELIMINATION` configuration option is provided to leverage `ld --gc-sections`.

## Mach-O

* Sections which define a symbol specified by `-u`, `-e`, or `-exported_symbol`
* Sections which define a symbol with the `N_NO_DEAD_STRIP` attribute or the `ReferencedDynamically` attribute
* Sections with the `S_ATTR_NO_DEAD_STRIP` flag

## Windows `link.exe /OPT:REF`

In `link.exe`, garbage collection only works with COMDAT sections.
This is not a major issue in practice because with function sections every text section is in a COMDAT, either `IMAGE_COMDAT_SELECT_NODUPLICATES` or `IMAGE_COMDAT_SELECT_ANY`.
