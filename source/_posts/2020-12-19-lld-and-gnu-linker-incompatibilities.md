layout: post
title: LLD and GNU linker incompatibilities
author: MaskRay
tags: [binutils,linker,llvm]
---

Updated in 2024-06.

Subtitle: Is ld.lld a drop-in replacement for GNU ld?

The motivation for this article was someone challenging the "drop-in replacement" claim on LLD's website (the discussion was about Linux-like ELF toolchain):

> LLD is a linker from the LLVM project that is a drop-in replacement for system linkers and runs much faster than them. It also provides features that are useful for toolchain developers.

99.9% pieces of software work with ld.lld without a change. Some linker script applications may need an adaption (such adaption is oftentimes due to brittle assumptions: asking too much from GNU ld's behavior which should be fixed anyway). So I defended for this claim.

<!-- more -->

Piotr Kubaj said that this is a probably more of a marketing term than a technical term, the term tries to lure existing users into thinking "it's the same you know, but better!". I think that this is fair in some senses: for many applications ld.lld has achieved much faster speed and much lower memory usage than GNU ld. A more important thing is that ld.lld adds a third choice to the spectrum. It brings competitive pressure to both sides, gives incentive for improvement, and makes for more standardized future features/extensions. One reason that I am subscribed to the binutils mailing list is I want to participate in its design processes (I am proud to say that I have managed to find some early issues of various new things).

Anyway, I thought documenting the compatibility problems between the ELF ports of ld.lld and GNU ld is useful, not only to others but also to my future self, hence this article. I will try to describe GNU gold behaviors as well.

So here is the long list. Please keep in mind that many compatibility issues do not really matter and a user may never run into such an issue.
Many of them just serve as educational purposes and my personal reference. There some some user perceivable differences but quite a lot are WONTFIX on both GNU ld and ld.lld. ld.lld, as a newer linker, has less legacy compatibility burden and can make good default choices in some cases and say no to some unneeded features/behaviors. A large number of features are duplicated in GNU ld's various ports. It is also common that one thing behaves this way in port A and another way in port B.

* GNU ld reports `gc-sections requires either an entry or an undefined symbol` in a `-r --gc-section` link. ld.lld doesn't error (<https://reviews.llvm.org/D84131#2162411>). I am unsure whether such a diagnostic will be useful (an uncommon use case where the GC roots are more than the explict linker options).
* The default image base for `-no-pie` links is different. For example, on x86-64, GNU ld defaults to 0x400000 while ld.lld defaults to 0x200000.
* GNU ld synthesizes a `STT_FILE` symbol when copying non-`STT_SECTION` `STB_LOCAL` symbols. ld.lld doesn't.
  + The `STT_FILE` symbol name is the input filename. For compiler driver specified startup files like `crti.o` and `crtn.o`, their absolute paths will end up in the linked image. This breaks local determinism (toolchain paths are leaked) for some users.
  + I filed <https://bugs.llvm.org/show_bug.cgi?id=48023> and <https://sourceware.org/bugzilla/show_bug.cgi?id=26822>. From binutils 2.36 onwards, the base name will be used.
* Default library paths.
  + GNU ld has default library paths.
  + ld.lld doesn't. This is intentional so <https://reviews.llvm.org/D70048> (NetBSD) cannot be accepted.
* GNU ld supports grouped short options. This can sometimes cause surprising behaviors with misspelled or unimplemented options, e.g. `-no-pie` means `-n -o -pie` because GNU ld as of 2.35 has not implemented `-no-pie`. Nick Clifton committed `Update the BFD linker so that it deprecates grouped short options.` to deprecated the GNU ld feature. ld.lld never supports grouped short options.
* Mixed SHF_LINK_ORDER and non-SHF_LINK_ORDER input sections in an output section.
  + ld.lld performs sorting within an input section description and allows arbitrary mixes.
  + GNU ld does not allow mixed sections <https://sourceware.org/bugzilla/show_bug.cgi?id=26256> (H.J. Lu has a patch)
* ld.lld defaults to `-z relro` by default. This is probably not a good default but it is difficult to change now. I have a comment <https://bugs.llvm.org/show_bug.cgi?id=48549>. GNU ld warns for `-z relro` and `-z norelro` for non Linux/FreeBSD BFD emulations (e.g. `-m aarch64elf`).
* Different archive member extraction semantics. See <http://lld.llvm.org/ELF/warn_backrefs.html> for details.
* ld.lld `--warn-backrefs` warns for `def.a ref.o def.so` if `def.a` cannot satisfy previous unresolved symbols. ld.lld resolves the definition to `def.a` while GNU linkers resolve the definition to `def.so`.
* GNU ld `-static` has traditionally been a synonym to `-Bstatic`. Recently on x86 it has been changed to behave a bit similar to `gold -static`, which disallows linking against shared objects. ld.lld `-static` is still a synonym to `-Bstatic`.
* GNU linkers have a default `--dynamic-linker`. ld.lld doesn't.
* GNU ld discards empty sections more aggressively. ld.lld does not discard an output section if it contains an empty input section.
* GNU ld has architecture-specific rules for relocations referencing undefined weak symbols. I don't think the GNU ld behaviors can be summarized (even by maintainers!). ld.lld's are consistent.
* The conditions to create `.interp` are different. I believe GNU ld's is quite difficult to describe.
* `--no-allow-shlib-undefined` and `--rpath-link`
  + GNU ld traces all shared objects (transitive `DT_NEEDED` dependencies) and emulates the behavior of a dynamic loader to warn more cases.
  + gold and ld.lld implement a simplified version. They warn for shared objects whose `DT_NEEDED` dependencies are all seen as input files.
  + ld.lld accepts and ignores `--rpath-link`
* `--fatal-warnings`
  + GNU ld still reports `warning: ...`.
  + ld.lld switches to `error: ...`.
* `--no-relax`
  + GNU ld: disable `R_X86_64_[REX_]GOTPCRELX`
  + ld.lld: no-op before 14.0.0 (<https://reviews.llvm.org/D113615>)
* GNU ld's internal linker scripts place `.ctors` into `.init_array`. gold enables `--ctors-in-init-array` by default which does the same thing. ld.lld doesn't implement the functionality.
* ld.lld places `.rodata` (among other `SHF_ALLOC` and non-`SHF_WRITE`-non-`SHF_EXECINSTR` sections) before `.text` (among other `SHF_ALLOC` and `SHF_EXECINSTR` sections).
* `.symtab`/`.shstrtab`/`.strtab` in a linker script.
  + Ignored by GNU ld, therefore `--orphan-handling=` does not warn/error.
  + Respected by ld.lld
* GNU ld recognizes some sections like `.text` and `.tbss` as special (`bfd_elf_special_section`). `.text`, if present as an output section, doesn't have the `SHF_WRITE` flag even if we add a `SHF_WRITE` input section into the output section.
* Whether `ADDR(.foo)` in a linker script can retain an empty output section.
  + GNU ld: no. Symbol assignments relative to such empty sections may have strange `st_shndx`.
  + ld.lld: yes.
* GNU ld does not produce `.rela.eh_frame` in `-r` or `--emit-relocs` mode. gold and ld.lld produce `.rela.eh_frame`.
* GNU ld applies tail merging to `.dynstr` and `.strtab`, but ld.lld doesn't.
* GNU ld applies tail merging to other `SHF_MERGE|SHF_STRINGS` sections by default. ld.lld performs tail merging only with `-O2`.
* If an undefined symbol is referenced by both `R_X86_64_JUMP_SLOT` (lazy) and `R_X86_64_GLOB_DAT` (non-lazy)
  + GNU ld generates `.plt.got` with `R_X86_64_GLOB_DAT` relocations. `R_X86_64_JUMP_SLOT` can thus be omitted to decrease the number of dynamic relocations.
  + ld.lld does not implement this saving. This naturally requires more than one pass scanning relocations which ld.lld doesn't do at present. <https://bugs.llvm.org/show_bug.cgi?id=32938>
* GNU ld relaxes `R_X86_64_GOTPCREL` relocations with some forms (e.g. `movq foo@GOTPCREL(%rip), %reg -> leaq foo(%rip), %reg`). ld.lld never relaxes`R_X86_64_GOTPCREL` relocations.
* GNU linkers give `.gnu.linkonce*` sections COMDAT section semantics. ld.lld simply ignores such sections. <https://bugs.llvm.org/show_bug.cgi?id=31586> tracks when the hack can be removed.
* GNU ld adds `PT_PHDR` and `PT_INTERP` together. A shared object usually does not have the two program headers. In ld.lld, `PT_PHDR` is always added unless the address assignment makes is unsuitable to place program headers at all.
* The conditions to create the dynamic symbol table `.dynsym`.
  + ld.lld: there is an input shared object, `-pie`/`-shared`, or `--export-dynamic`.
  + GNU ld's is quite complex. `--export-dynamic` is not special, though.
* `--export-dynamic-symbol`
  + gold's implies `-u`.
  + GNU ld (from 2.35 onwards) and ld.lld's do not imply `-u`.
* In GNU ld, a defined `foo@v` can suppress the extraction of an archive member defining `foo@@v1`. ld.lld treats them two separate symbols and thus the archive member extraction still happens. This can hardly matter. See [All about symbol versioning](https://maskray.me/blog/2020-11-26-all-about-symbol-versioning) for details.
* Default program headers.
  + With traditional `-z noseparate-code`, GNU ld defaults to a `RX/R/RW` program header layout. With `-z separate-code` (default on Linux/x86 from binutils 2.31 onwards), GNU ld defaults to a `R/RX/R/RW` program header layout.
  + ld.lld defaults to `R/RX/RW(RELRO)/RW(non-RELRO)`. With `--rosegment`, ld.lld uses `RX/RW(RELRO)/RW(non-RELRO)`.
  + Placing all R before RX is preferable because it can save one program header and reduce alignment costs.
  + ld.lld's split of RW saves one maxpagesize alignment and can make the linked image smaller.
  + This breaks some assumptions that the (so-called) "text segment" precedes the (so-called) "data segment".
  + For example, certain programs expect `.text` is the first section of the text segment and specify `-Ttext=0` to place the `PF_R|PF_X` program header at p_vaddr=0. This is a brittle assumption and should be avoided. If `PT_PHDR` is needed, `--image-base=0` is a replacement. If `PT_PHDR` is not needed, `.text 0 : { *(.text .text.*) }` is a replacement.
  + If `-Ttext=` or `--section-start` specifies an output section address below this base, the result is likely unintended. ld.lld since 21 will report an error. <https://github.com/llvm/llvm-project/pull/140187/>
* ld.lld places `.iplt` into the output section of the same name. GNU ld and gold just use `.plt`. ld.lld's approach has the benefit that the tricky feature stands out and symbolizers don't have to deal with mixed PLT entries.
* GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in `-pie` mode. glibc `csu/libc-start.c` needs it when statically linked, but not in the static pie mode. ld.lld does not distinguish `-no-pie`, `-pie` and `-shared`. <https://bugs.llvm.org/show_bug.cgi?id=48674>
* ld.lld uses `--no-apply-dynamic-relocs` by default. GNU ld and gold fill in the GOT entries with link-time values. GNU ld only supports `--no-apply-dynamic-relocs` for aarch64 <https://sourceware.org/bugzilla/show_bug.cgi?id=25891>.
* When relaxing `R_X86_64_REX_GOTPCRELX`, GNU ld suppresses the relaxation if it would cause relocation overflow. ld.lld does not perform the check.
* GNU ld and gold allow `--exclude-libs=b` to hide `b.a`. ld.lld requires `--exclude=libs=b.a`.
* ld.lld cannot GC non-group non-SHF_LINK_ORDER `.gcc_except_table*` sections. GNU ld can GC such sections. For Clang>=13, `clang -fbinutils-version=2.36` can set `SHF_LINK_ORDER` on `.gcc_except_table*` to allow GC.
* In GNU ld, a definition referenced by an unneeded (`--as-needed`) shared object is not exported into `.dynsym` [PR26551](https://sourceware.org/bugzilla/show_bug.cgi?id=26551). gold and ld.lld export such a definition.
* ppc64: GNU ld defines `.TOC.` to the output section of `.got` plus 0x8000. ld.lld chooses the input section `.got` plus 0x8000. All architectures I know define their GOT relocations relative to the input section `.got`.
* When a copy relocated symbol has aliases (e.g. `environ`, `_environ`, `__environ`), ld.lld copies all aliases, while GNU ld only copies some aliases.

### `--as-needed`

In GNU ld and gold, a shared object gets a `DT_NEEDED` entry if:

* it is linked at least once in --no-as-needed mode (i.e. `--as-needed a.so --no-as-needed a.so` => needed)
* or it has a definition resolving a non-weak reference

In GNU ld, an as-needed shared object is like an archive. It should come after the references.

In ld.lld,

* it is linked at least once in `--no-as-needed` mode (i.e. `--as-needed a.so --no-as-needed a.so` => needed)
* or it has a definition resolving a non-weak reference from a live section (not discarded by `--gc-sections`)

```asm
# RUN: split-file %s %t
# RUN: cc -c %t/a.s -o %t/a.o
# RUN: cc -shared -Wl,--soname=b.so %t/b.s -o %t/b.so
## DT_NEEDED entry.
# RUN: ld.bfd --gc-sections %t/a.o --as-needed %t/b.so -o %t.out
## No DT_NEEDED entry.
# RUN: ld.lld --gc-sections %t/a.o --as-needed %t/b.so -o %t.out

#--- a.s
.globl _start
_start:

.section .text.a,"ax"
call foo

#--- b.s
.weak foo
foo:
 ret
```

### Semantics of --wrap

GNU ld hand ld.lld have slightly different `--wrap` semantics. I use "slightly" because in most use cases users will not observe a difference.

In GNU ld, `--wrap` only applies to undefined symbols.
In ld.lld, `--wrap` happens after all other symbol resolution steps. The implementation is to mangle the symbol table of each object file (`foo -> __wrap_foo; __real_foo -> foo`) so that all relocations to `foo` or `__real_foo` will be redirected.

The ld.lld semantics have the advantage that non-LTO, LTO and relocatable link behaviors are consistent. I filed <https://sourceware.org/bugzilla/show_bug.cgi?id=26358> for GNU ld.

```asm
# GNU ld: call bar
# ld.lld: call __wrap_bar
  call bar
.globl bar
bar:
```

If there are `__real_foo` references but `foo` does not exist, GNU linkers can redirect `__real_foo` references to `foo`.
ld.lld doesn't do anything. This difference doesn't matter in practice.

```
# REQUIRES: x86
# RUN: rm -rf %t && split-file %s %t
# RUN: llvm-mc -filetype=obj -triple=x86_64 %t/a.s -o %t/a.o
# RUN: llvm-mc -filetype=obj -triple=x86_64 %t/b.s -o %t/b.o
# RUN: rm -f %tb.a && llvm-ar r %tb.a %t/b.o
# RUN: ld.lld %t/a.o %tb.a -o %t/lld --wrap pthread_create -z undefs
# RUN: ld.bfd %t/a.o %tb.a -o %t/bfd --wrap pthread_create -z undefs
# RUN: gold %t/a.o %tb.a -o %t/bfd --wrap pthread_create --unresolved-symbols=ignore-in-object-files

#--- a.s
.globl _start
_start:
.cfi_startproc
  call pthread_create
.cfi_endproc

#--- b.s
.global pthread_create
pthread_create:
.cfi_startproc
  ret
.cfi_endproc
```

### Relocation referencing a local relative to a discarded input section

* How to resolve a relocation referencing a `STT_SECTION` symbol associated to a discarded `.debug_*` input section.
  + GNU ld and gold have logic resolving the relocation to the prevailing section symbol.
  + ld.lld does not have the logic. ld.lld 11 defines some tombstone values.

> A symbol table entry with STB_LOCAL binding that is defined relative to one of a group's sections, and that is contained in a symbol table section that is not part of the group, must be discarded if the group members are discarded. References to this symbol table entry from outside the group are not allowed.

ld.bfd/gold/lld error if the section containing the relocation is SHF_ALLOC. .debug* do not have the SHF_ALLOC flag and those relocations are allowed.

lld resolves such relocations to 0. ld.bfd and gold, however, have some CB_PRETEND/PRETEND logic to resolve relocations to the definitions in the prevailing comdat groups. The code is hacky and may not suit lld.

<https://bugs.llvm.org/show_bug.cgi?id=42030>

### Canonical PLT entry for ifunc

How to handle a direct access relocation referencing a `STT_GNU_IFUNC`?

c.f. [GNU indirect function](2021-01-18-gnu-indirect-function).

### `__rela_iplt_start`

GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in `-pie` mode.
ld.lld defines `__rela_iplt_start` regardless of `-no-pie`, `-pie` or `-shared`.

Static pie and static no-pie relocation processing is very different in glibc.

* Static no-pie uses special code to process a magic array delimitered by `__rela_iplt_start/__rela_iplt_end`.
* Static pie uses self-relocation to take care of `R_*_IRELATIVE`. The above magic array code is executed as well. If `__rela_iplt_start`/`__rela_iplt_end` are defined (like what ld.lld does), we will get `0 < __rela_iplt_start < __rela_iplt_end` in `csu/libc-start.c`. `ARCH_SETUP_IREL` will crash when resolving the first relocation which has been processed.

nsz has a glibc patch that moves the self-relocation later so everything is set up for ifunc resolvers.

## Text relocations

* In GNU ld, `-z notext`/`-z text`/unspecified are a tri-state. For `-z notext`/unspecified, the dynamic tags `DT_TEXTREL` and `DF_TEXTREL` are added on demand. If unspecified and GNU ld is configured with `--enable-textrel-check=warning`, a warning will be issued.
* ld.lld has two states and adds `DT_TEXTREL` and `DF_TEXTREL` if `-z notext` is specified.
* GNU ld supports more relocation types as text relocations.

LLD 16.0.0 has changed a behavior related to absolute relocations in `.eh_frame`. See [Misc](#misc).

```cpp
echo 'void bar(); int main() { try { bar(); } catch (...) {} }' > a.cc
echo 'void bar() {}' > b.cc
clang++ -m32 -fno-pic -no-pie -fuse-ld=lld -z notext a.cc b.cc
```

## Short-range absolute relocations

```sh
cat > a.s <<e
.globl _start; _start:
.data; .short _start
e
llvm-mc -filetype=obj -triple=i686 a.s -o x86_32.o
llvm-mc -filetype=obj -triple=x86_64 a.s -o x86_64.o
```

For the short-range absolute relocations (e.g. `R_X86_64_16` and `R_X86_64_32`), GNU ld's x86_64 port rejects them regardless of whether `-z text` or `-z notext` is used.
Simiarly, ld.lld rejects them:
```
% ld.bfd -pie x86_64.o && readelf -Wr a.out
ld.bfd: x86_64.o: relocation R_X86_64_16 against symbol `_start' can not be used when making a PIE object; recompile with -fPIE
ld.bfd: failed to set dynamic section sizes: bad value
% ld.lld -pie x86_64.o
ld.lld: error: relocation R_X86_64_16 cannot be used against symbol '_start'; recompile with -fPIC
>>> defined in x86_64.o
>>> referenced by x86_64.o:(.data+0x0)
```

The link-time address of `_start` may be representable with an `R_X86_64_16`. However, with `-pie`, the semantics require that the run-time address of `_start` is representable, which is not the case.
Therefore linkers report an error.
This behavior is implemented in many other ports of GNU ld.

(To make the `R_X86_64_16` relocation accepted, the relocation should reference an absolute symbol (`SHN_ABS`) instead.)

GNU ld's i386 port somehow accepts such a short-range relocation in PIC links.
It is possibly an unimplemented check that gets abused. x86-32 is more or less considered legacy, so a linker change may be deemed unnecessary now.
```
% ld.bfd -m elf_i386 -pie x86_32.o && readelf -Wr a.out

There are no relocations in this file.
```

ld.lld's behavior with the default `-z text` is consistent with other ports, as it rejects the short-range relocations.
However, when `-z notext` is specified for x86-32 links, ld.lld converts the short-range relocations to dynamic relocations, specifically text relocations.
I believe that the conversion of `R_386_8` and `R_386_16` is unintended. Only `R_386_32`, `R_386_TLS_LE`, and `R_386_TLS_LE_32` should be supported as text relocations.
ld.lld's behavior is because we generally support fewer relocation types as text relocations than GNU ld.
x86-32 linkers traditionally have many hacks to work around legacy code. Some may be no longer relevant.
```
% ld.lld -pie x86_32.o
ld.lld: error: relocation R_386_16 cannot be used against symbol '_start'; recompile with -fPIC
>>> defined in x86_32.o
>>> referenced by x86_32.o:(.rodata+0x0)
% ld.lld -pie x86_32.o -z notext && readelf -Wr a.out

Relocation section '.rel.dyn' at offset 0x170 contains 2 entries:
 Offset     Info    Type                Sym. Value  Symbol's Name
00000180  00000014 R_386_16
000031f4  00000014 R_386_16
```

## Linker scripts

* GNU ld has an internal linker script if neither `-T` (`--script`) nor `-dT` (`--default-script`) is specified. gold and ld.lld do not have an internal linker script.
  + I closed <https://bugs.llvm.org/show_bug.cgi?id=51309> as reporting an internal linker script would significantly increase complexity without justifiable benefit.
  + Some built-in processing cannot be serialized into a linker script.
* If neither `FILEHDR` nor `PHDRS` is not specified, and the minimum address of `SHF_ALLOC` sections is set so that adding headers will need to add an extra page, ld.lld will not place ELF header and program headers in a `PT_LOAD` program header.
* Some linker script commands are unimplemented in ld.lld, e.g. `BLOCK()` as a compatibility alias for `ALIGN()`. `BLOCK` is documented in GNU ld as a compatibility alias and it is not widely used, so there is no reason to keep the kludge in ld.lld.
* Some syntax is not recognized by ld.lld, e.g. ld.lld recognizes `*(EXCLUDE_FILE(a.o) .text)` but not `EXCLUDE_FILE(a.o) *(.text)` (<https://bugs.llvm.org/show_bug.cgi?id=45764>)
  + To me the unrecognized syntax is misleading.
  + If we support one way doing something, and the thing has several alternative syntax, we may not consider the alternative syntax just for the sake of completeness.
* [Different orphan section placement](/blog/2024-06-02-understanding-orphan-sections). GNU ld has very complex rules and certain section names have special semantics. ld.lld adopted some of its core ideas but made a lot of simplication:
  + output sections are given ranks
  + output sections are placed after symbol assignments
  At some point we should document it. <https://bugs.llvm.org/show_bug.cgi?id=42327>
* For an error detected when processing a linker script, ld.lld may report it multiple times (e.g. `ASSERT` failure). GNU ld has such issues, too, but probably much rarer.
* `SORT` commands
  + GNU ld: <https://sourceware.org/binutils/docs/ld/Input-Section-Basics.html#Input-Section-Basics> mentions the feature but its behavior is strange/unintuitive. I created [SORT and multiple patterns in an input section description](https://sourceware.org/pipermail/binutils/2020-November/114083.html).
  + ld.lld performs sorting within an input section description. <https://reviews.llvm.org/D91127>
* In ld.lld, `AT(lma)` forces creation of a new `PT_LOAD` program header. GNU ld can reuse the previous `PT_LOAD` program header if LMA addresses are contiguous. `lma-offset.s`
* In ld.lld, non-`SHF_ALLOC` sections always get 0 `sh_addr`. In GNU ld you can have non-zero `sh_addr` but `STT_SECTION` relocations referencing such sections are not really meaningful.
* Dot assignment (e.g. `. = 4;`) in an output section description.
  + GNU ld: dot advances to 4 relative to the start. If you consider `.` on the right hand side and `ABSOLUTE(.)`, I don't think the behaviors are consistent.
  + ld.lld: move dot to address 0x4, which will usually trigger an `unable to move location counter backward` error. <https://bugs.llvm.org/show_bug.cgi?id=41169>

### Symbol assignments within an output section

See an example for GNU ld and lld's behaviors.
```
// a.s
nop

// a.x
SECTIONS {
  . = 0x1204;
  .text : {
    *(.text)
    /* dot is 0x1205 */
    a1 = ALIGN(32); /* = ALIGN(ABSOLUTE(.), 32) */
    a2 = 32;
    a2_1 = a2;
    a3 = ABSOLUTE(0x1225);
    a3_1 = a3;
    a4 = ABSOLUTE(0x1225) + 1;
    a4_1 = a4;
    a5 = ADDR(.text);
  }
  /DISCARD/ : { *(.dynsym) *(.gnu.hash) *(.hash) *(.dynstr) }
}
```

For (`bar = 32;`) in an output section description, GNU ld defines `bar` at 32 relative to the start of the output section.
Similarly, `. = 32` advances the location counter to 4 relative to the start of the output section.
The behaviors are different for symbol assignments outside of an output section and `ABSOLUTE` has special but unclear semantics.
When you think of binary operators, the behaviors are more confusing.

ld.lld simply doesn't implement the special but unclear semantics of relative offsets within an output section.

```
% as a.s -o a.o

# I choose -pie because st_shndx can be different in GNU ld's -no-pie mode. I think both ld.lld -no-pie and ld.lld -pie should be consistent with ld.bfd -pie, instead of ld.bfd -no-pie.
% ld.bfd -pie a.o -T a.x
% readelf -W -s a.out
...
     2: 0000000000001220     0 NOTYPE  GLOBAL DEFAULT    1 a1
     3: 0000000000001225     0 NOTYPE  GLOBAL DEFAULT  ABS a3
     4: 0000000000002429     0 NOTYPE  GLOBAL DEFAULT    1 a3_1
     5: 000000000000242a     0 NOTYPE  GLOBAL DEFAULT    1 a4_1
     6: 0000000000001204     0 NOTYPE  GLOBAL DEFAULT    1 a5
     7: 0000000000001224     0 NOTYPE  GLOBAL DEFAULT    1 a2
     8: 0000000000001224     0 NOTYPE  GLOBAL DEFAULT    1 a2_1
     9: 0000000000001226     0 NOTYPE  GLOBAL DEFAULT  ABS a4


% ld.lld -pie a.o -T a.x  # lld makes many symbols absolute
% readelf -W -s a.out
...
     2: 0000000000000020     0 NOTYPE  GLOBAL DEFAULT  ABS a2
     3: 0000000000001225     0 NOTYPE  GLOBAL DEFAULT  ABS a3
     4: 0000000000001226     0 NOTYPE  GLOBAL DEFAULT  ABS a4
     5: 0000000000001220     0 NOTYPE  GLOBAL DEFAULT  ABS a1
     6: 0000000000000020     0 NOTYPE  GLOBAL DEFAULT  ABS a2_1
     7: 0000000000001225     0 NOTYPE  GLOBAL DEFAULT  ABS a3_1
     8: 0000000000001226     0 NOTYPE  GLOBAL DEFAULT  ABS a4_1
     9: 0000000000001204     0 NOTYPE  GLOBAL DEFAULT    1 a5
```

For portability, I only recommend:

* `sym = .;` define a symbol at the current location
* `. += 4;` advance the location counter. This is preferred over `. = . + 4;`

### Output section fill

When the [fill pattern](https://sourceware.org/binutils/docs/ld/Output-Section-Fill.html) is a hexadecimal literal, e.g. 0x90, the pattern is automatically replicated to a 32-bit pattern 0x90909090 (`ld/ldexp.c:exp_get_fill`).
However, this behavior does not apply to a decimal literal or an expression.

```
.text : { *(.text) } =0x90        # set the fill pattern to 0x90909090
.text : { *(.text) } =0x90909090  # set the fill pattern to 0x90909090
.text : { *(.text) } =144         # set the fill pattern to 0x00000090
.text : { *(.text) } =0x90+0      # set the fill pattern to 0x00000090
```

This behavior makes me puzzled ([PR30865](https://sourceware.org/bugzilla/show_bug.cgi?id=30865)). ld.lld treats `=0x90` the same as `=144`.

### `PROVIDE`

In GNU ld, the right hand side of a PROVIDE assignment is processed only when the symbol is referenced (`ld/ldexp.c:exp_fold_tree_1`).
Therefore, if a symbol is only referenced by the right hand side of a PROVIDE assignment, it is not considered referenced and `PROVIDE` will not define it.

For simplicity, [ld.lld does not implement this special rule](https://github.com/llvm/llvm-project/issues/74771).
```sh
as /dev/null -o a.o
cat > a.t <<e
SECTIONS {
  PROVIDE(f3 = 0x1000);
  PROVIDE(f2 = f3);
  PROVIDE(f1 = f2);
  PROVIDE(foo = f1);
}
e
ld.bfd a.o -T a.t -o a.bfd
ld.lld a.o -T a.t -o a.lld
```
```
% readelf -s a.bfd

Symbol table '.symtab' contains 1 entry:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
% readelf -s a.lld

Symbol table '.symtab' contains 4 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000001000     0 NOTYPE  GLOBAL DEFAULT  ABS f3
     2: 0000000000001000     0 NOTYPE  GLOBAL DEFAULT  ABS f2
     3: 0000000000001000     0 NOTYPE  GLOBAL DEFAULT  ABS f1
```

GNU ld performs many iterations, but some ports (e.g. x86) do not allow too many iterations.
```
% cat chain.t
PROVIDE(f7 = 0x1000);
PROVIDE(f6 = f7);
PROVIDE(f5 = f6);
PROVIDE(f4 = f5);
PROVIDE(f3 = f4);
PROVIDE(f2 = f3);
PROVIDE(f1 = f2);
PROVIDE(newsym = f1);
% ld.bfd a.o -T chain.t
ld.bfd:chain.t:2: undefined symbol `f7' referenced in expression
```

## `.note.GNU-stack`

Linkers create the `PT_GNU_STACK` program header to indicate whether the program stack should be executable.

By default, GNU ld and gold set the `PF_X` bit of the `PT_GNU_STACK` program header if one relocatable object file does not contain `.note.GNU-stack`.
This behavior can be disabled with the configure option `--enable-default-execstack=no`.

ld.lld ignores `.note.GNU-stack` and defaults to `-z noexecstack`.

* Arch Linux feature request: [[binutils] Set --enable-default-execstack=no configure option](https://bugs.archlinux.org/task/75601)

## `.gnu.warning`

If an input section named `.gnu.warning` is included in the output, GNU linkers will issue a warning with the message extracted from the section's content.
```sh
echo '.globl _start; _start: .section .gnu.warning; .asciz "hello"' > a.s
gcc -c a.s
ld.bfd a.o  # a.o: warning: hello
```

If a section from a relocatable object file or shared object is named `.gnu.warning.$symbol`, GNU linkers will record the symbol name.
If the symbol is referenced by a relocatable object file, they will issue a warning.
```sh
echo '.globl _start; _start: .section .gnu.warning._start; .asciz "hello"' > a.s
echo 'call _start; call _start' > b.s
gcc -c a.s b.s
gcc -shared a.o -o a.so
```
```
% ld.bfd a.o b.o
ld.bfd: warning: hello
% gold a.o b.o
b.o(.text+0x1): warning: hello
b.o(.text+0x6): warning: hello
```

GNU ld will likely scan the symbol table and report a warning for each referenced symbol with the warning bit set.
gold moves the check to relocation scanning and reports a warning for each reference. However, this approach introduces some per-relocation overhead.

ld.lld doesn't implement the feature. It is unclear whether the feature is useful. <https://github.com/llvm/llvm-project/issues/41353>

For GCC and Clang, I think `__attribute__((deprecated(...)))` (`-Wdeprecated-declarations`) is a quite good alternative.

## `.comment`

ld.lld records linker version information in the section `.comment` for non-relocatable links.

GNU ld since 2023-03 records the information using the linker script directive `LINKER_VERSION`.
External linker script users can specify `--enable-linker-version`.

## Misc

I'll also mention some ld.lld release notes which can demonstrate some GNU incompatibility in previous versions.
(For example, if one thing is supported in version N, then the implication is that it is unsupported in previous versions.
Well, it could be that it worked in older versions but regressed at some version. However, I don't know the existence of such things.)

LLD 16.0.0

* Older versions may produce a dynamic relocation in `.eh_frame` when `-z notext` is specified. <https://reviews.llvm.org/D143136> switched to a canonical PLT entry.

LLD 12.0.0

* `-r --gc-sections` is supported.
* The archive member extraction semantics of COMMON symbols is by default (`--fortran-common`) compatible with GNU ld. You may want to read [Semantics of a common definition in an archive](https://sourceware.org/pipermail/binutils/2020-August/112878.html) for details. This is unfortunate.
* `.rel[a].plt` and `.rel[a].dyn` get the `SHF_INFO_LINK` flag. <https://reviews.llvm.org/D89828>

LLD 11.0.0

* ld.lld can discard unused symbols with `--discard-all`/`--discard-locals` when `-r` or `--emit-relocs` is specified. <https://reviews.llvm.org/D77807>
* `--emit-relocs --strip-debug` can be used. <https://reviews.llvm.org/D74375>
* ``SHT_GNU_verneed`` in shared objects are parsed, and versioned undefined symbols in shared objects are respected. Previously non-default version symbols could cause spurious `--no-allow-shlib-undefined` errors. <https://reviews.llvm.org/D80059>
* `DF_1_PIE` is set for position-independent executables. <https://reviews.llvm.org/D80872>
* Better compatibility related to output section alignments and LMA regions. [D75286](https://reviews.llvm.org/D75286) [D74297](https://reviews.llvm.org/D74297) [D75724](https://reviews.llvm.org/D75725) [D81986](https://reviews.llvm.org/D81986)
* `-r` allows `SHT_X86_64_UNWIND` to be merged into `SHT_PROGBITS`. This allows clang/GCC produced object files to be mixed together. <https://reviews.llvm.org/D85785>
* In a input section description, the filename can be specified in double quotes. `archive:file` syntax is added. <https://reviews.llvm.org/D72517> <https://reviews.llvm.org/D75100>
* Linker script specified empty `(.init|.preinit|.fini)_array` are allowed with RELRO. <https://reviews.llvm.org/D76915>

LLD 10.0.0

* ld.lld supports `\` (treating the next character like a non-meta character) and `[!...]` (negation) in glob patterns. <https://reviews.llvm.org/D66613>

LLD 9.0.0

* The `DF_STATIC_TLS` flag is set for i386 and x86-64 when initial-exec TLS models are used.
* Many configurations of the Linux kernel's arm32_7, arm64, powerpc64le and x86_64 ports can be linked by ld.lld.

LLD 8.0.0

* `SHT_NOTE` sections get very high ranks (they usually precede other sections). <https://reviews.llvm.org/D55800>

In the LLD 7.0.0 era, <https://reviews.llvm.org/D44264> was my first meaningful (albeit trivial) patch to ld.lld. Next I made contribution to `--warn-backrefs`. Then I started to fix tricky issues like copy relocations of a versioned symbol, duplicate `--wrap`, and section ranks. I have learned a lot from these code reviews. In the 8.0.0, 9.0.0 and 10.0.0 era, I have fixed a number of tricky issues and improved a dozen of other things and am confident to say that other than MIPS ;-) and certain other ISA specific things I am familiar with every corner of the code base. These are still challenges such as integration of RISC-V style linker relaxation and post-link optimization, improvement to some aspects of the linker script, but otherwise ld.lld is a stable and finished part of the toolchain.

A few random notes:

* Symbol resolution can take 10%~20% time. Parallelization can theoretically improve the process but it is hard to overstate the challenge (if you additionally take into account determinism).
* Be wary of feature creep. I have learned a lot from ELF design discussions on generic-abi and from Solaris "linker aliens" in particular. I am sorry to say so but some development on ld.lld indeed belongs to such categories. Sometimes it is difficult to draw a line between unsupported legacy and legacy we have to support.
* ld.lld's adoption is now so large that sometimes a decision (like a default value for an option) cannot make everyone happy.
