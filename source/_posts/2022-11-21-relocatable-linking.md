---
layout: post
title: Relocatable linking
author: MaskRay
tags: [linker,llvm]
---

Updated in 2025-02.

In GNU ld, `-r` produces a relocatable object file. This is known as relocatable linking or partial linking.
This mode suppresses many passes done for an executable or shared object output (in `-no-pie/-pie/-shared` modes).
`-r`, `-no-pie`, `-pie`, and `-shared` specify 4 different modes. The 4 options are mutually exclusive.

The relocatable output can be used for analysis and binary manipulation.
Then, the output can be used to link the final executable or shared object.

```sh
clang -pie a.o b.o

# ==>

clang -r a.o b.o -o r.o
clang -pie r.o
```

Let's go through various linker passes and see how relocatable linking changes the operation.

<!-- more -->

## Linker passes

### Find and scan input files

Only relocatable object files, archive files, and linker scripts are allowed as input.
To support LTO, another kind of files may be allowed as well.
Other files lead to an error. In another word, `-r` implies `-static` (`-Bstatic`).

```
% ld -r a.o b.so
ld.lld: error: attempted static link of dynamic object b.so
```

COMDAT resolution is performed and may discard some sections and hence symbols defined relative to them.

When FatLTO is enabled, the `.llvm.lto` section should be removed for a non-LTO relocatable link, because the default concatenated `.llvm.lto` sections cannot reflect the linking semantics.

### Symbol resolution

The linker does not do special treatment for symbol versioning. `@` and `@@` in symbol names are left as is.

The linker does not define reserved symbols (e.g. `__ehdr_start, _GLOBAL_OFFSET_TABLE_`).
The linker does not define encapsulation symbols (`__start_$section, _stop_$section`) for C identifier name sections.

When LTO is used, LTO assumes all non-local symbols may be used and avoids eliminating them.

### Create synthetic sections

The linker does not create synthesized sections (`.interp, .gnu.hash, .rela.dyn, .got, .plt, .comment`, etc).

### Map input sections and synthetic sections into output sections

GNU ld suppresses the internal linker script. There is no default mapping like `.text.*` to `.text`, `.data.*` to `.data`, etc.

One may write a linker script to map sections.

In GNU ld, `link_info.resolve_section_groups` is off by default for relocatable linking.
This means that:

* Sections with the `SHF_GROUP` flag are handled like sections matched by the `--unique=pattern` option. They are processed like orphan sections and ignored by input section descriptions.
* Section groups' (usually named `.group`) content is updated as the section indexes are updated. Section groups can be discarded with `/DISCARD/ : { *(.group) }`.

`-r --force-group-allocation` changes the behavior:

* Section groups are discarded.
* Input sections with the `SHF_GROUP` flag are matched like normal sections.

In the following example, `a.o` contains two section groups.

```sh
cat > a.cc <<eof
extern int x;
inline int inline0() { return x; }
inline int inline1() { return x+1; }
int use() { return inline0() + inline1(); }
eof
echo 'SECTIONS { .text1 : { *(.text .text.*) }}' > a.t
clang -c a.cc
ld.bfd -r -T a.t a.o -o bfd.ro
ld.bfd -r -T --force-group-allocation a.t a.o -o bfd.force.ro
ld.lld -r -T a.t a.o -o lld.ro
ld.lld -r -T --force-group-allocation a.t a.o -o lld.force.ro  # https://github.com/llvm/llvm-project/pull/94704 (milestone: lld 19)
```

`.text._Z7inline0v` is treated as an orphan in `bfd.ro`. It is treated as an orphan in `lld.ro` and a normal section in the other three output files.

In `lld.ro`, both section groups contain `.text1`, which violates the ELF specification.
`llvm-readelf -g lld.ro` reports:

```

COMDAT group section [    3] `.group' [_Z7inline0v] contains 1 sections:
   [Index]    Name
   [    1]   .text1

COMDAT group section [    4] `.group' [_Z7inline1v] contains 1 sections:
   [Index]    Name
llvm-readelf: warning: 'lld.ro': section with index 1, included in the group section with index 3, was also found in the group section with index 4
   [    1]   .text1
```

`SHF_LINK_ORDER` sections of the same name, linking to different input sections, cannot be combined. (<https://reviews.llvm.org/D68094>)

### Scan relocations

This pass is skipped. The linker does not determine whether GOT/PLT/TLSDESC/TLSGD/etc are needed.

### Layout and assign addresses

Output sections just get zero addresses.

### Write headers

The ELF header and the section header table are created as usual.
Program headers are suppressed.

### Symbol table

Symbols are generally concatenated. However, `STT_SECTION` symbols need to be coalesced.

Unlike an executable or shared object link, `STV_HIDDEN` visibility symbols in a relocatable object are not converted to `STB_LOCAL` binding.

### Copy section contents to output and resolve relocations

The linker transforms relocations relative to input sections to relocations relative to output sections.
This is similar to `--emit-relocs`. Addends may be changed and implicit addends may result in section content changes.

Some sections may be discarded.
For a non-`SHF_ALLOC` section, ld.lld applies tombstone values for relocations referencing a symbol defined relative to a discarded section.

Note, only a limited set of relocations could be resolved in relocatable files.
Specifically, this applies to relocations following the (S-P+A) formula where S refers to a non-ifunc local symbol in the same section as P.
This behavior mirrors [assembler fixup resolution](/blog/2025-03-16-relocation-generation-in-assemblers):

> Fixup resolution depends on the fixup type:
>
> - PC-relative fixups that describe the symbol itself (the relocation operation looks like S - P + A) resolve to a constant if sym_a is a non-ifunc local symbol defined in the current section.
> - relocation_specifier(S + A) style fixups resolve when S refers to an absolute symbol.
> - Other fixups, including TLS and GOT related ones, remain unresolved.

Many scenarios prevent relocation resolution:

- Cross-section symbol references
- Symbols with STB_WEAK or STB_GLOBAL binding (due to potential symbol interposition in shared libraries and linker script symbol assignments)
- TLS or GOT-related relocations, and relocations potentially requiring PLT entries
- Relocations potentially requiring range-extension thunks or linker relaxation

Linkers implement `-r` typically just retain all relocations.

COMMON symbols are not allocated.

### Shared passes

For major passes mentioned above, relocatable linking seems to skip most of them.
However, there are a large number of minor features that relocatable linking shares with regular linking modes (`-no-pie, -pie, -shared`): `--wrap, --compress-debug-sections, --build-id, -Map`, etc.

A linker implementation may use a separate file to describe passes for relocatable linking, but it can easily miss the miscellaneous features.

## Garbage collection

The output combines sections and loses granularity than using all input files, therefore garbage collection for the final link will be less effective.

`-r --gc-sections` can be used together. I added the support to ld.lld in <https://reviews.llvm.org/D84131>.
A relocatable link does not set a default entry symbol (usually `_start`). One shall set up GC roots (`--entry` and `-u`) to use `-r --gc-sections`.

## Output differences due to a relocatable link step

The following two links may produce different output.
```sh
clang -pie a.o b.o -lz -ltinfo

clang -r a.o b.o -o r.o
clang -pie r.o -lz -ltinfo
```

Say both relocatable object files define `.rodata._ZL7indent8` and `.rodata._ZL7indent16`. (The section names are due to `-fdata-sections`.)
In the first link, the 4 input sections are combined in this order: `a.o:.rodata._ZL7indent8, a.o:.rodata._ZL7indent16, b.o:.rodata._ZL7indent8, b.o:.rodata._ZL7indent16`.
In the second link, the relocatable link combines `a.o:.rodata._ZL7indent8` and `b.o:.rodata._ZL7indent8` (and the same with `.rodata._ZL7indent8`), so the final order appears shuffled.

This grouping effect may affect the output section size due to alignment padding.

## Application

### Alternative to a static library

The total numbers of sections and symbols are smaller, making the relocatable smaller than the total size of input files.
This makes relocatable linking an alternative to a static library at the expense of less effective garbage collectin.

### glibc

glibc provides many crt1 files: `Scrt1.o, rcrt1.o, crt1.o, grcrt1.o, gcrt1.o`. These files share a lot of common code but have customization.
glibc uses relocatable linking to create these files, e.g.

* `Scrt1.o` is the relocatable link output of `csu/start.os`, `csu/abi-note.o`, and `csu/init.o`.
* `crt1.o` is the relocatable link output of `csu/start.o`, `csu/abi-note.o`, and `csu/init.o`.

`elf/Makefile` performs the following steps to build `elf/ld.so`:

* Create `elf/libc_pic.a` from libc `.os` files
* Create `elf/dl-allobjs.os` from a relocatable link of rtld `.os` files
* Create link map `elf/librtld.map` from a relocatable link of `elf/dl-allobjs.os`, `elf/libc_pic.a`, and `-lgcc`
* Get a list of extracted archive members (`elf/librtld.mk`) from `elf/librtld.map` and create `elf/rtld-libc.a`
* Create `elf/librtld.os` from a relocatable link of `elf/dl-allobjs.os` and `elf/rtld-libc.a`
* Create `elf/ld.so` from a `-shared` link of `elf/librtld.os` with the version script `ld.map`

Here relocatable links are used for its symbol resolution: figure out what libc components need to go into `elf/ld.so`.
The version script `ld.map` is used to localize the libc symbols.

### grub

`grub-core/genmod.sh.in` uses relocatable linking to generate modules (e.g. `grub-core/reboot.module`).
`grub-core/gensyminfo.sh.in` dumps symbols into `grub-core/syminfo.lst` which is then used to generate a module dependency list.

### Linux kernel

`tools/objtool/objtool-in.o` is a relocatable output linked into `tools/objtool/objtool`.
I do not know why this step is needed.

`vmlinux.o` is a relocatable output for analysis (see `scripts/Makefile.vmlinux_o`).

* used by modules
* objtool uses `vmlinux.o` to perform "noinstr" (no-instrumentation) validation.

```sh
ld.lld -m elf_x86_64 -z noexecstack -r -o vmlinux.o  --whole-archive vmlinux.a --no-whole-archive --start-group  --end-group
llvm-objcopy  -j .modinfo -O binary vmlinux.o modules.builtin.modinfo
tr '\0' '\n' < modules.builtin.modinfo | sed -n 's/^[[:alnum:]:_]*\.file=//p' | tr ' ' '\n' | uniq | sed -e 's:^:kernel/:' -e 's/$/.ko/' > modules.builti
```

Kernel modules (`.ko` files) are created by `ld -r` and `modpost`.

Unsurprisingly, the ClangBuiltLinux project's contributors have identified some interesting corner cases with ld.lld.
I think most issues were reported in 2019 and fixed by me.

### Sanitizer internal symbolizer

Sanitizers can symbolize with an external tool `llvm-symbolizer` or an internal symbolizer.

The internal symbolizer uses a script [`compiler-rt/lib/sanitizer_common/symbolizer/scripts/build_symbolizer.sh`](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/sanitizer_common/symbolizer/scripts/build_symbolizer.sh).

* Build a portion of LLVM with `-flto`.
* Invoke `llvm-link` to link LLVM bitcode files.
* Internalize most symbols with `opt -internalize -internalize-public-api-list=$list`. Only `__sanitizer_symbolize_*` are kept as global symbols.
* Compile the bitcode into `symbolizer.o`.
* Inject `symbolizer.o` into `libclang_rt*san*.a` archives.

The bitcode compiles with `llvm-link` are an alternative to relocatable linking.

The built runtime libraries will be used by user programs. `clang -fsanitize=address` links `libclang_rt.asan.a` in `--whole-archive` mode, so the output will link in `symbolizer.o` with internal symbolizer functionality.
The output may use LLVM as well and the linked LLVM may be of a different version. The internalization in `symbolizer.o` avoids symbol collision.

### Localize symbols

The previous section mentions a trick with `llvm-link`. We can achieve the same effect with relocatable linking.
After relocatable linking, use `objcopy --keep-global-symbol` to keep a few entry symbols and localize the rest symbols.
The output can be distributed as a single `.o` file or as part of an archive file.

Both the `llvm-link` approach and the relocatable linking approach need extra care dealing with COMDAT groups.
See [COMDAT and section group#GRP_COMDAT](/blog/2021-07-25-comdat-and-section-group#grp_comdat) for why all COMDAT group signatures should be kept global to avoid pitfalls.

Here are the steps:

- Compile with hidden symbols by default (`-fvisibility=hidden` for definitions, or `#pragma GCC visibility("hidden")` for both definitions and references). Mark necessary symbols as exported
- Run `ld -r --force-group-allocation`. Optionally specify `--unique=*` to keep section granularity.
- Process the output with `objcopy --localize-hidden` with possibly other options.

### Find more projects using relocatable linking

Many projects have complex build systems and it's difficult to know whether `-r` is used.
I use a ld wrapper `/usr/local/bin/ld` to retrieve linker command lines
```sh
#!/bin/zsh
for i in "$@"; do
  if [[ $i == -r ]]; then
    echo "$@" >> /tmp/link.log
    break
  fi
done
LD_PRELOAD=path/to/libmimalloc.so ~/Stable/bin/ld.lld --threads=8 "$@"
```

## Mach-O

Apple ld also supports `-r`, which produces an object file of type `MH_OBJECT`.
The feature is sometimes called "single-object prelink".

You can specify `-exported_symbol` or `-exported_symbols_list` to make symbols private.

```
printf 'void foo(){} void bar(){}' | clang -c -xc - -o a.o
ld -arch arm64 -r -exported_symbol bar a.o
ld -arch arm64 -r -exported_symbols_list a.txt a.o
```
