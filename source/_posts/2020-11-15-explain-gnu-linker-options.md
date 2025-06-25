layout: post
title: Explain GNU style linker options
author: MaskRay
tags: [binutils,linker]
---

Updated in 2025-02

(首先慶祝一下LLVM 2000 commits達成！)

## Compiler driver options

Before describing the linker options, let's introduce the concept of driver options.
The user-facing options of `gcc` and `clang` are called driver options.
Some driver options affect the options passed to the linker.
Many such options have the same name as the linker's, and they often have additional functions in addition to the options of the same name passed to the linker, such as:

<!-- more -->

* `-shared`: Don't set `-dynamic-linker`; don't link `crt1.o`
* `-static`: Don't set `-dynamic-linker`; use `crtbegint.o` instead of `crtbegin.o`, use `--start-group` link `-lgcc -lgcc_eh -lc` (they have (bad) circular dependency)

`-Wl,--foo,value,--bar=value` will pass the three options `--foo`, `value`, and `--bar=value` to the linker.
If there are a lot of link options, you can put each line in a text file `response.txt`, and then specify `-wl,@response.txt`.

Note that `-O2` will not pass `-O2` to the linker, but `-Wl,-O2` will.

* `-fno-pic,-fno-PIC` are synonymous and generate position-dependent code.
* `-fpie,-fPIE` are called small PIE and large PIE respectively. They introduce an optimization on the basis of PIC: the compiled .o can only be used for executable files. See `-Bsymbolic` below.
* `-fpic,-fPIC` are called small PIC and large PIC. They generate position-independent code respectively. There are differences in code generation between the two modes on 32-bit powerpc and sparc (architectures that are about to retire). There are no differences in most architectures.

## Input files

The linker accepts several types of input.
For symbols, the symbol table of each input file will affect symbol resolution.
For sections, only sections (called input sections) in regular object files will contribute to the sections of the output file (called output sections).

* .o (regular object files)
* .so (shared objects): only affects symbol resolution
* .a (archive files)

See [Symbol processing](/blog/2021-06-20-symbol-processing) for symbol resolution detail.

## Modes

The linker is in one of the following four modes.
The mode controls the output type (executable file/shared object/relocatable object).

* `-no-pie` (default): Generate a position-dependent executable (`ET_EXEC`). This mode has the most loose requirements: the source files can be compiled with `-fno-pic, -fpie/-fPIE, -fpic/-fPIC`.
* `-pie`: Generate a position-independent executable (`ET_DYN`). Source files need to be compiled with `-fpie/-fPIE, -fpic/-fPIC`
* `-shared`: Generate a position-independent shared object (`ET_DYN`). The most restrictive mode: source files need to be compiled with `-fpic/-fPIC`.
* `-r`: Generate a relocatable file. This is called a relocatable link and is special. It suppresses various linker synthesized sections and reserves relocations. See [Relocatable linking](/blog/2022-11-21-relocatable-linking).

Confusingly, the compiler driver provides several options with the same name: `-no-pie, -pie, -shared, -r`.
GCC 6 introduced the configure-time option `--enable-default-pie`: such builds enable `-fPIE` and `-pie` by default.
Now, many Linux distributions have enabled this option as the basic security hardening.

### Executable link (`-no-pie` and `-pie`)

A defined symbol is non-preemptible.
For a branch instruction using a PLT-generating relocation, the branch can be bound directly the the definition, avoiding a PLT.
For a code sequence involving a GOT-generating relocation, the code sequence may be optimized to use direct access. See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table) for detail.

A non-local `STV_DEFAULT/STV_PROTECTED` defined symbol is by default not exported to the dynamic symbol table.

#### `-no-pie`

`-no-pie` indicates that the link-time address equals the run-time address.
This property is leveraged by a linker: all relocations referencing a non-preemptible symbol can be resolved, including absolute GOT-generating (e.g. `R_AARCH64_LD64_GOT_LO12_NC`), PC-relative GOT-generating (e.g. `R_X86_64_REX_GOTPCRELX`), etc.
In the absence of a GOT optimization, the GOT entry for a non-preemptible symbol is a constant, avoiding a dynamic relocation.
The image base is an arch-specific non-zero value by default.

* Some architectures have different PLT code sequences (i386, ppc32 .glink).
* `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` can be further optimized
* ppc64 `.branch_lt` (long branch addresses) can be optimized

#### `-pie`

`-pie` is very similar to `-shared -Bsymbolic`, but it produces an executable file. The following behavior is close to `-no-pie` but different from `-shared`:

* Allow copy relocation and canonical plt.
* Allow relax general dynamic/local dynamic tls models and tls descriptors to initial exec/local exec.
* Will resolve undefined weak symbols to zeroes. ld.lld does not generate dynamic relocation. Whether GNU ld generates dynamic relocation has very complicated rules and is architecture dependent.

### Shared object link (`-shared`)

A non-local `STV_DEFAULT` definition is preemptible (interposable) by default, that is, the definition may be replaced by the definition in the executable file or another shared object at runtime.
The compiler and the linker cooperate by using GOT and PLT entries to reference such symbols.

A non-local `STV_DEFAULT/STV_PROTECTED` symbol is exported to the dynamic symbol table.
E.g. in the following program, `foo` is exported to the dynamic symbol table if linked with `-shared` but (by default) not if linked with `-no-pie` or `-pie`.
```c
void foo() {}
```

### PIC link (`-pie` and `-shared`)

A symbolic relocation (absolute relocation & width matches the word size) referencing a non-preemptible non-TLS symbol converts to a relative relocation.

See [Relative relocations and RELR](/blog/2021-10-31-relative-relocations-and-relr) for detail.

## Symbol related

### `-Bsymbolic`

The `-Bsymbolic` family options make non-local `STV_DEFAULT` definitions in a shared object non-preemptible. They are a no-op for executable output. See "Mode" above for an introduction of "preemptible".

* `-Bsymbolic` makes all definitions except those matched by `--dynamic-list/--export-dynamic-symbol-list/--export-dynamic-symbol` non-preemptible
* `-Bsymbolic-functions` is similar to `-Bsymbolic`, but only applies to `STT_FUNC` definitions
* `-Bsymbolic-non-weak-functions` is similar to `-Bsymbolic`, but only applies to non-`STB_WEAK` `STT_FUNC` definitions

See [ELF interposition and -Bsymbolic](/blog/2021-05-09-elf-interposition-and-bsymbolic) for detail.

### `--defsym`

Define a symbol.

Similar to ld64 `-alias`.

### `--exclude-libs`

If a matched archive (either regular or surrounded by `--whole-archive`/`--no-whole-archive`) defines a non-local symbol, don't export the symbol.

As an example, `clang++ -static-libstdc++ -Wl,--export-dynamic,--exclude-libs=libstdc++.a a.cc` does not export libstdc++ defined symbols.

### `--export-dynamic`

The option puts non-local `STV_DEFAULT/STV_PROTECTED` defined symbols to the dynamic symbol table in an executable output.
The option is a no-op when

* `-shared` is specified, because a shared object does this by default.
* `-no-pie` is specified without any input shared object, because the dynamic symbol table is not present.

Here are the rules (logical AND) that a symbol is exported to the dynamic symbol table:

* non-local `STV_DEFAULT/STV_PROTECTED` (this means it can be hidden by `--exclude-libs`)
* logical OR of the following:
  + undefined symbol
  + (`--export-dynamic` || `-shared`) && `! (unnamed_addr linkonce_odr GlobalValue || local_unnamed_addr linkonce_odr (constant GlobalVariable || Function))` (in LTO, certain `linkonce_odr` symbols can be hidden)
  + matched by `--dynamic-list/--export-dynamic-symbol-list/--export-dynamic-symbol`
  + defined or referenced by a shared object as `STV_DEFAULT`
  + `STV_PROTECTED` definition in a shared object preempted by copy relocation/canonical PLT when `--ignore-{data,function}-address-equality}` is specified
  + `-z ifunc-noplt` && has at least one relocation

If the executable file defines a symbol that is referenced by a link-time shared object, the linker exports the symbol so that the undefined symbol in the shared object can be bound to the definition in the executable file at runtime.
If the executable file defines a symbol that is also defined by a link-time shared object, the linker exports the symbol to enable symbol interposition at runtime.

In LLVM, certain unnamed_addr `GlobalValue`s are not affected by `--export-dynamic` for executable linking.
See `lld/test/ELF/lto/internalize-exportdyn.ll` and `lld/test/ELF/lto/unnamed-addr-comdat.ll`.

### `--export-dynamic-symbol=glob`, `--export-dynamic-symbol-list`, and `--dynamic-list`

These options have different semantics for an executable and a shared object.

* executable: put matched non-local defined symbols to the dynamic symbol table (`--export-dynamic` applies to all non-local defined symbols.)
* shared object: references to matched non-local `STV_DEFAULT` symbols shouldn't be bound to definitions within the shared object, even if they would otherwise be due to `-Bsymbolic`, `-Bsymbolic-functions`, or `--dynamic-list`

`--dynamic-list` additionally implies `-Bsymbolic`.

For the shared object case, I usually call the operation "make a symbol preemptible".

One may use `--export-dynamic-symbol=foo*` to match all non-local `STV_DEFAULT` symbols `foo*`. ld.lld before 11 uses an exact match instead of a glob.

`--export-dynamic-symbol-list` is implemented since [GNU ld 2.35](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=37a141bfed4dd3c33d77c15dfde00e4b4f5b24c7) and [ld.lld 14](https://reviews.llvm.org/D107317).

In the following example, we shall see a `GLOB_DAT` dynamic relocation iff `var` is preemptible.
```c
// a.c
int var;
int inc() { return ++var; }
```

```
# Preemptible by default in a shared object.
% clang -O2 -fpic -shared a.cc && readelf -Wr a.out | grep var
0000000000003fc8  0000000500000006 R_X86_64_GLOB_DAT      000000000000400c var + 0

# -Bsymbolic makes a definition non-preemptible.
% clang -O2 -fpic -shared -Bsymbolic a.cc && readelf -Wr a.out | grep var

# --export-dynamic-symbol makes a definition preemptible despite -Bsymbolic
% clang -O2 -fpic -shared -Wl,-Bsymbolic,--export-dynamic-symbol=var a.cc && readelf -Wr a.out | grep var
0000000000003fc8  0000000500000006 R_X86_64_GLOB_DAT      000000000000400c var + 0

# Without a symbolic intention option, --export-dynamic-symbol is a no-op for -shared.
% clang -O2 -fpic -shared -Wl,--export-dynamic-symbol=foo a.cc && readelf -Wr a.out | grep var
0000000000003fc8  0000000500000006 R_X86_64_GLOB_DAT      000000000000400c var + 0

# --dynamic-list implies -Bsymbolic.
% clang -O2 -fpic -shared -Wl,--dynamic-list=<(printf '{a;};') a.cc && readelf -Wr a.out | grep var

# A matched symbol is still preemptible.
% clang -O2 -fpic -shared -Wl,--dynamic-list=<(printf '{var;};') a.cc && readelf -Wr a.out | grep var
0000000000003fc8  0000000500000006 R_X86_64_GLOB_DAT      000000000000400c var + 0
```

If a symbol matched by `local:` in a version script is specified by a dynamic list, the version script takes precedence and the symbol will be made local.

### `--discard-none`, `--discard-locals`, and `--discard-all`

If `.symtab` is produced, a local symbol defined in a live section is preserved if:

```cpp
if ((--emit-relocs or -r) && referenced) || --discard-none
  return true
if --discard-all
  return false
if --discard-locals
  return is not .L
# No --discard-* is specified.
return not (.L in a SHF_MERGE section)
```

For RISC-V, newer GCC and Clang pass `-X` (`--discard-locals`) to the linker.

### `--no-undefined-version`

If a version script specifies an exact pattern which does not match a defined symbol, report an error.

Say we have the following version script, if `foo` is not a defined symbol, the linker will report an error.
For a glob pattern (e.g. `bar*`) matching no symbol, there is no error. This is a compromise.
```
v1 {
  foo;
  bar*;
};
```

GNU ld has supported `--no-undefined-version` since [2002-08](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=3194163592008a9e575d577921647ab91b09c77b), but `--undefined-version` was a late addition in 2022-10 (milestone: binutils 2.40).

### `--strip-all`

Do not create `.strtab` or `.symtab`.

### `-u symbol`

If an archive file defines the symbol specified by `-u`, then pull the relevant member (convert from archive file to object file, and then the file will be the same as normal .o).

For example: `ld -u foo ... a.a`. If `a.a` does not define the symbols referenced by previous object files, `a.a` will not be pulled.
If `-u foo` is specified, then the archive member with `foo` defined in `a.a` will be pulled.

Another usage of `-u` is to specify a GC root.

### `--version-script=script`

The version script has three purposes:

* Define versions
* Specify some patterns so that the matched, defined, unversioned symbols have the specified version
* local version: `local:` can make matched defined symbols `STB_LOCAL`

The binding of the unversioned symbol is `STB_LOCAL` and will not be exported to the dynamic symbol table

If a symbol matched by `local:` is specified by a dynamic list, the version script takes precedence and the symbol will be made local.

[All about symbol versioning](/blog/2020-11-26-all-about-symbol-versioning) describes symbol versioning in detail.

### `-y symbol`

Often used for debugging. Output where the specified symbol is referenced and defined.

### `-z muldefs`

Alias: `--allow-multiple-definition`

A symbol is allowed to be defined in multiple files. By default, the linker does not allow two non-local regular definitions (non-weak, non-common) with the same name.

### `-z unique-symbol`

Rename local symbols so that there are no duplicates.

Some Intel folks were working on function granular kernel address space layout randomization and wanted such a feature (<https://sourceware.org/bugzilla/show_bug.cgi?id=26391>).
GNU ld since 2.36 supports this option.
I closed the ld.lld [feature request](https://bugs.llvm.org/show_bug.cgi?id=50745).

I don't think this is a good design.

First, the stability problem.
Say, the old kernel has `foo.1 foo.2`.
If there is a new local `foo` symbol, the new kernel will have `foo.1 foo.2 foo.3`.
However, The new symbols don't necessarily correspond to local symbols of the same names in the old kernel.
Such disturbence is probably more likely with LTO or PGO.
For Clang LTO, the kernel Makefile currently specifies `-mllvm -import-instr-limit=5`.
If a function close to the boundary happens to cross the boundary, if inlined into other translation units, the stability issue may affect many translation units.

The implementation has to perform an iteration on all local symbols, which can affect link speed.

In addition, the `.[0-9]+` scheme has been used by C++ mangling.
The Itanium C++ ABI says "A <mangled-name> containing a period represents a vendor-specific version or portion of the entity named by the <encoding> prior to the first period. There is no restriction on the characters that may be used in the suffix following the period."
In GNU, this is used to represent function cloning.

```text
% c++filt <<< $'_ZL3foov\n_ZL3foov.1'
foo()
foo() [clone .1]
```

As an alternative, I suggest that the FGASLR developer uses the `STT_FILE` symbol:
```text
STT_FILE    a.c
STT_NOTYPE  foo
STT_FILE    b.c
STT_NOTYPE  foo
```

The ELF specification says:

> Conventionally, the symbol's name gives the name of the source file associated with the object file. A file symbol has STB_LOCAL binding, its section index is SHN_ABS, and it precedes the other STB_LOCAL symbols for the file, if it is present.

I mentioned my concern on a reply to [``[PATCH v9 02/15] livepatch: use `-z unique-symbol` if available to nuke pos-based search``](https://lore.kernel.org/all/20211223002209.1092165-3-alexandr.lobakin@intel.com/).

## Library related

### `--as-needed` and `--no-as-needed`

Normally each link-time shared object has a `DT_NEEDED` tag.
Such a shared object will be loaded by the dynamic loader.

`--as-needed` can avoid unneeded `DT_NEEDED` tags.
`--as-needed` and `--no-as-needed` are position-dependent options (informally called, but no more appropriate adjectives).
In ld.lld, a shared object is needed, if one of the following conditions is true:

* it is linked at least once in `--no-as-needed` mode (i.e. `--as-needed a.so --no-as-needed a.so` => needed)
* or it has a definition resolving a non-weak reference from a live section (not discarded by `--gc-sections`)

In gold, the rule is probably:

* it is linked at least once in `--no-as-needed` mode (i.e. `--as-needed a.so --no-as-needed a.so` => needed)
* or it has a definition resolving a non-weak reference

In GNU ld, the rules are quite complex. The basic looks like the following:

* it is linked at least once in `--no-as-needed` mode (i.e. `--as-needed a.so --no-as-needed a.so` => needed)
* or it has a definition resolving a non-weak reference by a previous input file (it works similar to archive selection)

In `ld.bfd ... a.so --as-needed b.so --no-as-needed`, [if `a.so` references a symbol defined by `b.so` but `a.so` does not need `b.so`, the final output will need `b.so`](https://discourse.llvm.org/t/as-needed-breaks-our-build-with-undefined-symbols/75505/6).
This is probably used as a workaround for underlinking problems.
When the missing dependency (`b.so`) by a shared object is seen, the output will get the `DT_NEEDED` entry to satisfy `b.so`'s requirement, even if itself doesn’t need the dependency.

### `-Bdynamic` and `-Bstatic`

These two options are position-dependent options, which affect `-lname` that appears on the command line later.

* `-Bdynamic` (default): Search for `libfoo.so` and `libfoo.a` in the directory list specified by `-l`
* `-Bstatic`: Search `libfoo.a` in the directory list specified by `-l`

Historically `-Bstatic` and `-static` are synonymous in GNU ld.
The compiler driver option `-static` is a different option. In addition to passing `-static` to ld, it also removes the default `--dynamic-linker`, which affects the linking of libgcc, libc, etc.

### `--no-dependent-libraries`

ld.lld specific. Ignore sections of type `SHT_LLVM_DEPENDENT_LIBRARIES` (conventionally named `.deplibs`) in object files.

This section contains a list of filenames. The filenames will be add by ld.lld as additional input files.

### `-soname=name`

Set the `DT_SONAME` dynamic tag in the dynamic table of the generated shared object.

The linker will record the shared objects at link time, and use a `DT_NEEDED` record in the dynamic table of the generated executable file/shared object to describe each shared object at link time.

* If the shared object contains `DT_SONAME`, this field provides the value of `DT_NEEDED`
* Otherwise, if the link is through `-l`, the value is the base file name
* Otherwise, the value is the path name (there is a difference between absolute/relative paths)

For example: `ld -shared -soname=a.so.1 a.o -o a.so; ld b.o ./a.so`, `a.out` has a `DT_NEEDED` tag of `a.so.1`.
If the first command does not contain `-soname`, `a.out` will have a `DT_NEEDED` tag of `./a.so`.

### `--start-group` and `--end-group`

If there is a mutual reference between `a.a` and `b.a`, and you are not sure which one will be pulled into the link first, you have to use this pair of options. An example is given below:

For an archive linking order: `main.o a.a b.a`, assuming that `main.o` refers to `b.a`, and `a.a` does not satisfy a previous undefined symbol, then the linking order will cause an error.
Can the link order be replaced by `main.o b.a a.a`? If `main.o` references `a.a` after the change, and `b.a` does not satisfy one of the previous undefined symbols, then the link sequence will also cause an error.

One solution is `main.o a.a b.a a.a`. In many cases, it is enough to repeat `a.a` once, but if only `a.a(a.o)` is loaded when linking the first a.a, only `b.a(b.o)` is loaded when linking b.a, and only is loaded when linking the second `a.a(c.o)` and `a.a(c.o)` needs another member in `b.a`, this link sequence will still cause undefined symbol error.

We can repeat `b.a` again, that is `main.o a.a b.a a.a b.a`, but a better solution is `main.o --start-group a.a b.a --end-group`, or `main.o -( a.a b.a -)`.

### `--start-lib` and `--end-lib`

See [Archives and --start-lib](/blog/2022-01-16-archives-and-start-lib).
If `a.a` contains `b.o c.o`, `ld ... --start-lib b.o c.o --end-lib` works like `ld ... a.a`.

### `--sysroot`

This is different from the `--sysroot` driver option.
In GCC/Clang, the driver option `--sysroot` does two things:

* Decide include/library search paths (e.g. `$sysroot/usr/include`, `$sysroot/lib64`)
* Pass `--sysroot` to ld.

In ld,

* `-l =foo` and `-l=foo` find `libfoo.so` or `libfoo.a` under the sysroot directory.
* `foo` in `INPUT` or `GROUP` finds `foo` under the sysroot directory.
* If a linker script is in the sysroot directory, when it opens an absolute path file (`INPUT` or `GROUP`), add sysroot before the absolute path.

### `-t` `--trace`

Print relocatable object files, shared objects, and extracted archive members.

### `--whole-archive` and `--no-whole-archive`

The .a after the `--whole-archive` option will be treated as .o without lazy semantics.
If `a.a` contains `b.o c.o`, then `ld --whole-archive a.a --no-whole-archive` has the same effect as `ld b.o c.o`.

### `--push-state` and `--pop-state`

GNU ld implemented the options in binutils 2.25.

`-Bstatic, --whole-archive, --as-needed`, etc. are all position-dependent options that represent the boolean state. `--push-state` can save the boolean state of these options, and `--pop-state` will restore it.

When inserting a new option in the link command line to change the state, you usually want to restore it. At this time, you can use `--push-state` and `--pop-state`.
For example, to make sure to link `libc++.a` and `libc++abi.a`, you can use `-wl,--push-state,-Bstatic -lc++ -lc++abi -wl,--pop-state`.

## Dependency related

See [Dependency related linker options](/blog/2021-06-13-dependency-related-linker-options) for details.

### `-z defs` and `-z undefs`

Whether to report an error for an unresolved undefined symbol from a regular object.
"unresolved" means that the symbol is not defined by a regular object file or a link-time shared object.
Executable links default to `-z defs/--no-undefined` (not allowed) and `-shared` links default to `-z undefs` (allowed).

Many build systems enable `-z defs`, requiring shared objects to specify all dependencies when linking (link what you use).

### `--allow-shlib-undefined` and `--no-allow-shlib-undefined`

Whether to report an error for an unresolved `STB_GLOBAL` undefined symbol from a shared object.
Executable links default to `--no-allow-shlib-undefined` (report errors) and `-shared` links default to `--allow-shlib-undefined` (do not report errors).

For the following code, an error will be reported when linking the executable file:
```c
// a.so
void f();
void g() {f();}

// exe
void g()
int main() {g();}
```

If you specify `--allow-shlib-undefined` when linking the executable, the link will succeed, but ld.so will report an error at runtime.
In glibc, the error is `symbol lookup error: ... undefined symbol: `.

GNU ld has a complex algorithm to find transitive closures. Only when shared objects of transitive closures cannot resolve an undefined symbol, an error will be reported.
gold and lld use a simplified rule: if all `DT_NEEDED` dependencies of a shared object are directly linked, an error is enabled; if some of the dependencies are not linked, then gold/lld cannot accurately determine whether an indirectly shared object can provide a definition, so they are conservative and do not report errors.

It is worth mentioning that `-z defs/-z undefs/--no-undefined` and `--[no-]allow-shlib-undefined` can be controlled by an option `--unresolved-symbols`.

### `--warn-backrefs`

See [Dependency related linker options#--warn-backrefs](/blog/2021-06-13-dependency-related-linker-options#warn-backrefs).

## Layout related

### `--no-rosegment`

By default ld.lld places read-only data sections (e.g. `.rodata`) and text sections (e.g. `.text`) into two `PT_LOAD` segments.

* R `PT_LOAD`
* RX `PT_LOAD`
* RW `PT_LOAD` (overlaps with `PT_GNU_RELRO`)
* RW `PT_LOAD`

Specify this option to combine the R `PT_LOAD` and the RX `PT_LOAD`.
The RX `PT_LOAD` segment is traditionally called the text segment and is the first segment.

ld.lld places rodata and data on both sides of text. This layout has the advantage that the distance between text and data is shorter, decreasing the relocation overflow pressure.

gold is the first linker which implements `--rosegment`.

### `--xosegment`

This option enables support for [execute-only memory](https://isopenbsdsecu.re/mitigations/execute_only/).

* AArch32 uses the `SHF_ARM_PURECODE` section flag to desginate sections with pure program instructions and no data.
* AArch64 uses the `SHF_AARCH64_PURECODE` section flag.

By default, LLD treats sections with the `SHF_ALLOC|SHF_EXECINSTR|SHF_AARCH64_PURECODE` flags as compatible with those having `SHF_ALLOC|SHF_EXECINSTR` flags, merging them into a single `PT_LOAD segment`.
When `--xosegment` ois specified, LLD separates these sections into distinct `PT_LOAD` segments: one for sections with `SHF_ALLOC|SHF_EXECINSTR|SHF_AARCH64_PURECODE` and another for sections with `SHF_ALLOC|SHF_EXECINSTR`.

### `-z noseparate-code`

This is GNU ld's classic layout allowing some file content to be mapped as more than one `PT_LOAD` segments, with one being executable and another one being non-executable.
In this layout, two adjacent `PT_LOAD` program headers may overlap in file offsets. This trick avoids padding before the start of the next program header.

In the absence of linker script fragments, there are typically just two `PT_LOAD` segments:

* RX `PT_LOAD`: encompassing both read-only sections (`SHF_ALLOC`) and executable sections (`SHF_ALLOC|SHF_EXECINSTR`).
* RW `PT_LOAD`
  + The prefix part is `PT_GNU_RELRO`. This part of mprotect becomes readonly after rtld processes dynamic relocations.
  + The part that is not `PT_GNU_RELRO`. This part is always writable at runtime.

The first `PT_LOAD` is often called the text segment. The term is somewhat inaccurate because the segment has read-only data as well.

This layout is used by default since ld.lld 10 for its size benefits.

Note: when a `SHT_NOBITS` section is followed by another section, the `SHT_NOBITS` section behaves as if it occupies the file offset range.
This is because ld.lld does not implement a file size optimization.
This optimization unused by almost all linked images because it's rare to add `SHF_ALLOC` sections after a `SHT_NOBITS SHF_ALLOC` section.

### `-z separate-code`

The option is introduced in binutils 2.31 and enabled by default on Linux/x86. GNU ld has such a layout:

* R `PT_LOAD`
* RX `PT_LOAD`
* R `PT_LOAD`
* RW `PT_LOAD`
  + `PT_GNU_RELRO` part
  + Non-`PT_GNU_RELRO` part

In this layout, two adjacent `PT_LOAD` program headers cannot overlap in file offsets. That is, a byte (RX `PT_LOAD`) in the file that is mapped to the executable section will not be mapped to an R `PT_LOAD` at the same time.
The idea is that since read-only memory cannot be executed so ROP gadgets there cannot be used.
However, this is pretty much a secure theatre as executable memory has plenty of ROP gadgets anyway.

Due to implementation complexity, the adopted layout is not so great in that there is another read-only `PT_LOAD` after the `RX` `PT_LOAD`.
A better layout is to merge this R with the first R ([PR23704](https://sourceware.org/bugzilla/show_bug.cgi?id=23704)).
Another issue is that when there is no RW `PT_LOAD`, the first few `non-`SHF_ALLOC` sections' content may be mapped to the RX memory.

I introduced this option in ld.lld 10. The semantics are similar to GNU ld but the layout is different: the two RW `PT_LOAD` are allowed to overlap, which means that the address of the second `PT_LOAD` does not need to be aligned, and max-page-size*2 bytes can be wasted at most.

GNU ld's `-z separate-code` is essentially split into two options in lld: `-z separate-code` and `--rosegment`.

### `-z separate-loadable-segments`

This is ld.lld's traditional layout: all `PT_LOAD` segments do not overlap (a byte will not be loaded into two memory mappings at the same time).
I added the option in [2019](https://reviews.llvm.org/D67481).

The implementation is that the address of each new `PT_LOAD` is aligned to max-page-size. lld presets 4 `PT_LOAD`(r,rx,rw(relro),rw(non-relro)). Three alignments in the output file may waste some bytes.
On aarch64 and powerpc, because the max-page-size specified by abi is larger (65536), up to 65536*3 bytes can be wasted.

### `-z relro`

Place RELRO sections in the `PT_GNU_RELRO` program header.

GNU ld uses one RW `PT_LOAD` program header with padding at the start. The first half of the `PT_LOAD` overlaps with `PT_GNU_RELRO`.
The padding is added so that the end of `PT_GNU_RELRO` is [aligned by max-page-size](https://sourceware.org/bugzilla/show_bug.cgi?id=28824). (See `ld.bfd --verbose` output.)
Prior to GNU ld 2.39, the end was aligned by common-page-size.
GNU ld's one RW `PT_LOAD` layout makes the alignment increase the file size. max-page-size can be large, such as 65536 for many systems, causing [wasted space](https://sourceware.org/bugzilla/show_bug.cgi?id=30612).

lld utilitizes two RW `PT_LOAD` program headers: one for RELRO sections and the other for non-RELRO sections.
Although this might appear unusual initially, it eliminates the need for alignment padding as seen in GNU ld's layout.
I implemented the current layout in 2019 (<https://reviews.llvm.org/D58892>).
The end of the `PT_GNU_RELRO` segment and the associated RW `PT_LOAD` segment is [padded to a common-page-size boundary](https://github.com/llvm/llvm-project/pull/66042).
This is achieved by adding a padding section `.relro_padding` like mold.

Before LLD 18, there is an issue that runtime_page_size < common-page-size does not work.

The layout used by mold is similar to that of lld.
In mold's case, the end of `PT_GNU_RELRO` is padded to max-page-size by appending a `SHT_NOBITS` `.relro_padding` section.
This approach ensures that the last page of `PT_GNU_RELRO` is protected, regardless of the system page size.
However, when the system page size is less than max-page-size, the map from the first `RW` `PT_LOAD` is larger than needed.

In my opinion, losing protection for the last page when the runtime page size is larger than common-page-size is not really an issue.
Double mapping a page of up to max-common-page for the protection could cause undesired VM waste.
Protecting `.got.plt` is the main purpose of `-z now`. Protecting a small portion of `.data.rel.ro` doesn't really make the program more secure, given that `.data` and `.bss` are so huge and full of attach targets.
If users are really anxious, they can set common-page-size to match their system page size.

GNU ld's internal linker scripts place RELRO sections between `DATA_SEGMENT_ALIGN` and `DATA_SEGMENT_RELRO_END` (built-in functions).
`DATA_SEGMENT_ALIGN` is where padding is added so that `DATA_SEGMENT_RELRO_END` aligns to a max-page-size boundary.
```
. = DATA_SEGMENT_ALIGN(CONSTANT(MAXPAGESIZE), CONSTANT(COMMONPAGESIZE));
. = DATA_SEGMENT_RELRO_END(0, .);
. = DATA_SEGMENT_END(.);
```

ld.lld emulates these built-in functions:

* `DATA_SEGMENT_ALIGN`: set the current location to `alignTo(script->getDot(), + align)`
* `DATA_RELRO_END`: set the current location to `alignTo(script->getDot(), MAXPAGESIZE)`. `.relro_padding` is placed immediately before `DATA_RELRO_END`.

### `-z lrodata-after-bss`

See [Relocation overflow and code models#x86-64 linker requirement](/blog/2023-05-14-relocation-overflow-and-code-models#x86-64-linker-requirement).

### `--execute-only`

This is a ld.lld specific option for AArch64. The option requires `--rosegment` and makes the `RX` `PT_LOAD` segment executable-only (`PF_X`).

## Relocation related

### `--apply-dynamic-relocs`

Some psABI use the RELA format (AArch64, PowerPC, RISC-V, x86-64, etc): relocations contain the addend field.
On such targets, `--apply-dynamic-relocs` requires the linker to set the initial value of the relocated location to the addend instead of 0.
If the executable file/shared objects uses compression, `--no-apply-dynamic-relocs` can improve compression.

`--apply-dynamic-relocs` is supported for all ports in ld.lld.
As of August 2023, [only the aarch64 port](https://sourceware.org/PR25891) of GNU ld supports `--apply-dynamic-relocs`.

### `--emit-relocs`

This option makes `-no-pie/-pie/-shared` links to keep input relocations, in a way similar to `-r`.
It can be used for binary analysis after linking. The only two uses I know are `config_relocatable` and bolt of linux kernel x86.

The output section order may be different with `--emit-relocs`.
`.rela.eh_frame` sections are kept. See <https://reviews.llvm.org/D44679> that the first `.rela.eh_frame` input section may cause `.eh_frame` to be placed before other read-only output sections.

GNU ld's powerpc uses the [transformed relocation types](/blog/2023-02-26-linker-notes-on-power-isa#emit-relocs).

### `--pack-dyn-relocs=value`

`relr` can enable `DT_RELR`, a more compact relative relocation (`R_*_RELATIVE`) encoding format.
Relative relocations are common in position independent executables.

### `-z rel` and `-z rela`

Each architecture has a prevailing relocation format.
ld.lld implements `-z rel` to use REL for dynamic relocations even on architecture using RELA as the prevailing format.
This option can save some space.

* COPY, GLOB_DAT and J[U]MP_SLOT always have 0 addend. A rtld implementation does not need to read the implicit addend. REL is strictly better.
* A RELATIVE has a non-zero addend. It can use an implicit addend as well. Alternative, such relocations can be packed compactly with the RELR relocation entry format.
* For other dynamic relocation types (e.g. symbolic relocation R_X86_64_64), a ld.so
  implementation needs to read the implicit addend. REL may have minor performance impact, because implicit addends force random access reads instead of being able to blast out a bunch of writes while chasing the relocation array.

### `-z report-relative-reloc`

Dump information about `R_*_RELATIVE` and `R_*_IRELATIVE` relocations.

### `-z text` and `-z notext`

`-z text` does not allow text relocations.
`-z notext` allows text relocations.

Starting from binutils 2.35, GNU ld on linux/x86 enables the configure-time option `--enable-textrel-warning=warning` by default, and a warning will be given if there are text relocations.

The wording of the concept of text relocations is inaccurate. The actual meaning is the general term for dynamic relocations acting on sections without the `SHF_WRITE` flag.
If the value of relocations in .o cannot be determined at link time, it needs to be converted to dynamic relocations and calculated by ld.so at runtime (type and .o are the same).
If the active section does not have the `SHF_WRITE` flag, ld.so will have to temporarily execute `mprotect` to change the permissions of the memory maps, modify, and restore the previous read-only permissions, which hinders page sharing.

Shared objects form text relocations more than executable files.
Executable files have canonical plt and copy relocations to avoid certain text relocations.

Different linkers allow different relocation types of text relocations on different architectures. GNU ld may allow quite a few relocation types supported by glibc ld.so.
On x86-64, the linker will allow `R_X86_64_64` and `R_X86_64_PC64`. However, most loaders don't support `R_X86_64_PC64`.

In the following assembler, `defined_in_so` is a symbol defined in a shared object. The scenario of each text relocation is given in the comments.

```asm
.globl global
global:
local:
  .quad local              # (-pie or -shared) R_X86_64_RELATIVE
  .quad global             # (-pie) R_X86_64_RELATIVE or (-shared) R_X86_64_64
  .quad defined_in_so      # (-shared) R_X86_64_64
  .quad defined_in_so - .  # (-shared) R_X86_64_PC64
```

In `-no-pie` or `-pie` mode, the linker will make different choices according to the symbol type of `defined_in_so`:

* `STT_FUNC`: generate canonical plt
* `STT_OBJECT`: Generate copy relocation
* `STT_NOTYPE`: gnu ld will generate copy relocation. lld will generate text relocation

## Section related

### `--gc-sections`

Specify `-ffunction-sections` or `-fdata-sections` at compile time to have an effect.
The linker will do liveness analysis to remove unused sections from the output.

See [Linker garbage collection](/blog/2021-02-28-linker-garbage-collection) for detail.

### `-z start-stop-gc` and `-z nostart-stop-gc`

`-z start-stop-gc` means a `__start_foo` or `__stop_foo` reference from a live section does not retain all `foo` input sections.

`-z nostart-stop-gc` means a `__start_foo` or `__stop_foo` reference from a live section retains all `foo` input sections.

See [Metadata sections, COMDAT and SHF_LINK_ORDER](/blog/2021-01-31-metadata-sections-comdat-and-shf-link-order) for detail.

### `--icf=all` and `--icf=safe`

Enable identical code folding.
The name originated from MSVC linker `/OPT:ICF` where "ICF" stands for "identical COMDAT folding". gold named it after "identical code folding".


This name is slightly misleading:

* The feature operates on sections instead of functions.
* The feature apply to readonly data as well.

We define identical sections as they have identical content and their outgoing relocation sets cannot be distinguished:
they need to have the same number of relocations, with the same relative locations, with the referenced symbols indistinguishable.
This is a recursive definition: if `.text.a` and `.text.b` reference different symbols at the same location, they can still be indistinguishable if the referenced symbols satisfy the identical code/rodata requirement.

Among a group of identical sections, the linker may conservatively suppress folding for some.
`--keep-unique=<symbol>` makes the section defining `<symbol>` unique.
In ld.lld, a readonly section is by default foldable (gold does not fold readonly data). However, a readonly section defining a `.dynsym` symbol is not.

For the rest sections, in a set of identical sections, the linker picks one representative and drops the rest, then redirect references to the representative.

gold implements `--icf=safe` based on relocation.

`ld.lld --icf=safe` uses a special section `.llvm_addrsig` (LLVM address significance table, type `SHT_LLVM_ADDRSIG`) produced by Clang `-faddrsig`.
As of 2023-01, `-faddrsig` is the default on most Linux targets but disabled for Android, Gentoo, and `-fintegrated-as`.
If the section is absent, ld.lld is conservative and assumes every section defining a symbol in the table is address significant.

`SHT_LLVM_ADDRSIG` encodes symbol indexes as ULEB128. `objcopy`, `ld -r`, and other binary manipulation tools may alter the symbol table.
An interesting property is that `objcopy` and `ld -r` sets `sh_link=0` for an known section type `SHT_LLVM_ADDRSIG`.
ld.lld uses `sh_link!=0` to check the validity and reports a warning in case of `sh_link==0`.

I am somewhat sad that the design tradeoff leans toward the code size but not the generality, and the current state that a number of Linux distributions default to `-faddrsig` in Clang Driver while others default to `-fno-addrsig`.
It might be better if we used `R_*_NONE` relocations (and use REL instead of RELA to decrease size bloat) to encode symbol indexes.
Then that perhaps means we should default to `-fno-addrsig` and let users opt in the feature.

lld's Mach-O port [chose](https://discourse.llvm.org/t/problems-with-mach-o-address-significance-table-generation/63392) a relocation-based representation for `__DATA,__llvm_addrsig`.

`ld.lld --icf=all` ignores `.llvm_addrsig`.

For a `-shared` link of
```c
int foo() { return 1; }
int bar() { return 1; }
```

`foo` and `bar` are in `.dynsym`.
`ld.lld --icf=safe` assumes that symbols in `.dynsym` are address significant and two symbols cannot share the same address, so ld.lld conservatively suppresses merging `.text.foo` and `.text.bar`.

`gold --icf=safe` does merge `.text.foo` and `.text.bar`. Such a choice will be unsafe if a program uses a map and expects that `map[dlsym(h, "foo")]` and `map[dlsym(h, "bar")]` resolve to different objects.

In LLVMCodeGen, a global value with a `{,local_}unnamed_addr` attribute does not go into `.llvm_addrsig`.

`--icf=all` gives up C++ language guarantee about pointer equality.
Some think this is fair as some portion of the guarantee is sabotaged anyway (`-fvisibility-inlines-hidden`).
See [ELF interposition and -Bsymbolic](/blog/2021-05-09-elf-interposition-and-bsymbolic).

In <https://reviews.llvm.org/D141310>, an opt-in Clang diagnostic `-Wcompare-function-pointers` is proposed to catch some problems which will cause `--icf=all` to fail.

ICF can make debugging more difficult as the debugger might not be able to distinguish between the folded instances.

* Debug information associated to folded functions is essentially redirected.
* Setting a breakpoint on a function also affects folded functions.

ICF can alter function names in stack traces and make profiling inaccurate.

The debug information regression could be alleviated if you enable `DW_AT_LLVM_stmt_sequence` and use the caller to disambiguate the address in a folded function.

https://github.com/llvm/llvm-project/pull/139493#issuecomment-2896493771

When we have

* Section A1, with a relocation on a symbol S1, where S1 is at offset K in section B1.
* Section A2, with a relocation on a symbol S2, where S2 is at offset K in section B2.

When the relocation type is `R_AARCH64_ADR_GOT_PAGE`, and sections B1 and B2 are merged while keeping symbols S1 and S2 separate, sections A1 and A2 can currently be merged, which leads to correctness issues.

### `--symbol-ordering-file=<file>`

Specify a text file with one defined symbol per line.
Within an input section description (e.g. `*(.text .text.*)`), sort the matched input sections: if symbol A is before symbol B in the ordering file, place the section defining A before the section defining B.

If a symbol is not defined or the section in which it is located is discarded, the linker will output a warning unless `--no-warn-symbol-ordering` is specified.
However, the default `--no-warn-symbol-ordering` seems to often get in the way.

`--symbol-ordering-file=` is primarily used for two goals: performance or compression.

If one function frequently calls another, and the input sections where the two functions are located in the linked image are close, the probability that they will fall on the same page will increase.
By considering references among functions and placing corelated functions together, the page working set can be reduced, and the TLB thrashing can be reduced. See *profile guided code positioning* by Karl Pettis and Robert C. Hansen.

Mobile applications often prioritize compressed code size.
For cold functions, their compressed size matters much more than their performance.
To improve compressed size, similar functions can be grouped together to enhance compression algorithms (like the Lempel-Ziv family).

This option is unique to ld.lld. gold has a `--section-ordering-file`, sorted by section name.
In practice, text and data sections mostly have different names. However, `clang -fno-unique-section-names` ([GCC feature request](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=95095)) can create sections of the same that defeat `--section-ordering-file`.

```sh
cat > a.s <<e
.section .rodata.0,"a",@progbits; .byte 0x0
.section .rodata.1,"a",@progbits; .byte 0x1
.section .rodata.2,"a",@progbits; .byte 0x2
.section .rodata.3,"a",@progbits; .byte 0x3
.section .rodata.4,"a",@progbits; .byte 0x4
e
cat > a.txt <<e
.rodata.3
.rodata.[2]
.rodata.1
e
as a.s -o a.o
gold --section-ordering-file=a.txt a.o -o a
```

```
% readelf -x .rodata a

Hex dump of section '.rodata':
  0x004000b0 00040302 01                         .....
```

GNU ld 2.43 introduced `--section-ordering-file` with different semantics.
The section ordering script must specify output sections already defined in the linker script.
The specified extra mapping will be prepended to the output section.
```sh
cat > b.txt <<e
.rodata : { *(.rodata.3) *(.rodata.[2]) *(.rodata.1) }
e
ld.bfd --section-ordering-file=b.txt a.o -o a
```

### `--call-graph-profile-sort`

When `--call-graph-profile-sort` (default) is in effect, ld.lld inspects `SHT_LLVM_CALL_GRAPH_PROFILE` sections (call graph profile) in input relocatable object files.
A `SHT_LLVM_CALL_GRAPH_PROFILE` section consists of (from_symbol, to_symbol, weight) tuples.
LLD utilitizes the information to compute a call graph with input sections as nodes and (from_section, to_section, weight) as edges, then sorts sections within an input section description.
The sorting algorithm is based on _Optimizing Function Placement for Large-Scale Data-Center Applications_.

LLD sorts input sections by decreasing density, where density is computed as the weight divided by the size.
Initially, each input section is placed in a cluster by itself.
When processing each input section, its cluster is appended to the cluster containing its most likely predecessor in the call graph.
A merge can be blocked if any of the following conditions are satisfied:

* The edge is unlikely (the edge weight is too small considering the sum of input edge weights).
* The total size of the two clusters is larger than a threshold.
* The merged density would make the predecessor cluster's density much smaller.

Finally, the clusters are sorted by decreasing density.

If `--symbol-ordering-file=` is specified, `--symbol-ordering-file=` specified sections are placed first.
The call graph profile is still used for other sections (lld >= 20).

takes precedence over `--call-graph-profile-sort`.

When both `--call-graph-profile-sort` and `--print-symbol-order=` are specified, ld.lld will dump the symbol order in the specified file.
The file can be used with `--symbol-ordering-file=`.

### `--bp-compression-sort=` and `--bp-startup-sort=`

Both options instruct the linker to optimize section layout with the following goals:

* `--bp-compression-sort=[data|function|both]`: Improve Lempel-Ziv compression by grouping similar sections together, resulting in a smaller compressed app size.
* `--bp-startup-sort=function --irpgo-profile=<file>`: Utilize a temporal profile file to reduce page faults during program startup.

The linker determines the section order by considering three groups:

* Function sections ordered according to the temporal profile (`--irpgo-profile=`), prioritizing early-accessed and frequently accessed functions.
* Function sections. Sections containing similar functions are placed together, maximizing compression opportunities.
* Data sections. Similar data sections are placed together.

Within each group, the sections are ordered using the Balanced Partitioning algorithm.

The linker constructs a bipartite graph with two sets of vertices: sections and utility vertices.

* For profile-guided function sections:
  + The number of utility vertices is determined by the symbol order within the profile file.
  + If `--bp-compression-sort-startup-functions` is specified, extra utility vertices are allocated to prioritize nearby function similarity.
* For sections ordered for compression: Utility vertices are determined by analyzing k-mers of the section content and relocations.

The call graph profile is disabled during this optimization.

When `--symbol-ordering-file=` is specified, sections described in that file are placed earlier.

### `-z nosectionheader`

GNU ld 2.41 introduced the option to omit the section header table.

## Analysis related

### `--cref`

Output the cross reference table. For each non-local symbol, output the defined file and the list of files with references.

### `-m` and `-map=<file>`

Output the link map, you can view the address of the output sections, the file offset, and the included input sections.

## Warning related

### `--fatal-warnings`

Turn warnings into errors. The difference between warning and error is that besides whether it contains `warning` or `error` string, the more important point is that error prevents the output of the link result.

### `--noinhibit-exec`

Turn some errors into warnings. Be careful not to specify `--fatal-warnings` to upgrade the degraded warnings to errors again:)

## Randomness related

### `--shuffle-sections=<seed>`

Shuffle input sections to uncover bugs that rely on a certain order of sections.

### `--randomize-section-padding=<seed>`

Randomly insert padding between input sections and at the start of each segment using given seed.

Imagine a change that unintentionally reduces the memory alignment of a frequently executed function.
While the original program might not have guaranteed alignment for this function, the change could exacerbate the issue.
Using `--randomize-section-padding` can help uncover such subtle performance degradations by introducing variability in memory layout.

## Others

### `--build-id=value`

Generate `.note.gnu.build-id` to give the output an identifier. The identifier is generated by hashing the whole output.

SHA-1 is the most common choice. The linker will fill in the content of `.note.gnu.build-id` with zeros, hash each byte and fill the result back to `.note.gnu.build-id`.
Some linkers use tree-style hashes for parallelism.

### `--compress-debug-sections=[zlib|zstd]`

Use zlib or zstd to compress `.debug_*` sections of the output file and mark `SHF_COMPRESSED`.
See [Compressed debug sections](/blog/2022-01-23-compressed-debug-sections).

### `--hash-style=style`

The ELF specification requires a hash table `DT_HASH` for dynamic symbol lookup. `--hash-style=sysv` generates the table.

`DT_GNU_HASH` is better than `DT_HASH` in terms of space consumption and performance.
mips uses the alternative `DT_MIPS_XHASH` (a good example of mips abi suffering from its own wisdom).
I personally think `DT_MIPS_XHASH` is solving a wrong problem. In fact, there is a way to use `DT_GNU_HASH`, but people in the mips community may not want to worry about it another time.

See [glibc and DT_GNU_HASH](/blog/2022-08-21-glibc-and-dt-gnu-hash) for a story about "Easy Anti-Cheat".

### `--no-ld-generated-unwind-info`

See [PR12570 .plt has no associated .eh_frame/.debug_frame](https://sourceware.org/bugzilla/show_bug.cgi?id=12570).

When the pc is in the plt entry, if the linker does not synthesize `.eh_frame` information, unwinding from the current PC will not get frames.
On i386 and x86-64, in the lazy binding state, the first call of a plt entry will execute the push instruction. After the esp/rsp is changed, if the plt entry does not have the unwind information provided by `.eh_frame`, the unwinder may not be able to unwind correctly, which affects the accuracy of profilers.

```asm
jmp *got(%rip)
pushq $0x0
jmpq .plt
```

However, I think this feature is obsoleted and irrelevant nowadays.
To recognize the PLT name, a profiler needs to do:

* Parse the `.plt` section to know the region of PLT entries
* Parse `.rel[a].plt` to get `R_*_JUMP_SLOT` dynamic relocations and their referenced symbol names.
* If the current PC is within the PLT region, parse nearly instructions and find the GOT load. The associated `R_*_JUMP_SLOT` identifies the symbol name.
* Concatenate the symbol name and `@plt` to form `foo@plt`

Note: `foo@plt` is a convention used by some tools, but it is not a name in the symbol table.

gdb has heuristics to identify this situation.

This problem will not affect the c++ exception. The plt entry is a tail call, and the `_Unwind_RaiseException` called by `__cxa_throw` will penetrate the tail calls of the ld.so resolver and plt entry.
The pc will be restored to the next instruction of the caller of the plt entry.

```cpp
// b.cc - b.so
void ext() { throw 3; }

// a.cc - exe
#include <stdio.h>

void ext();
void foo() {
  try {
    ext(); // PLT entry
  } catch (int x) {
    printf("%d\n", x);
  }
}

int main() {
  foo();
}
```

### `-O`

Enable size optimizations.
The optimization level is different from the compiler driver option `-O`.
`-O` does not imply `--lto-O`: there is no effect on LTO code generation.

In ld.lld, `-O1` is the default.

`-O0` disables constant merge of `SHF_MERGE`.

`-O2` enables some computation heavy size optimization:

* enable string suffix merge of `SHF_MERGE|SHF_STRINGS`. This is very slow and not parallel.
* `--compress-debug-sections=zlib` uses zlib compression with higher compression ratio.
* Since 14.0.0, deduplicate local symbol names in `.strtab`. I may remove this completely once ld.lld supports parallel `.symtab` write .

In GNU ld, non-zero `-O` can make `.hash` and `.gnu.hash` smaller.

For a symbol assignment referencing a `SHF_MERGE` section, it is considered to refer to the constant data element.
After duplicate elimination, the symbol value is adjusted to refer to the data element in the output section.

### `-plugin file`

GNU ld and gold support this option to load GCC LTO plugin (`liblto_plugin.so`) or LLVM LTO plugin (`LLVMgold.so`).
`clang -flto={full,thin}` passes `-plugin path/to/LLVMgold.so` unless `-fuse-ld=lld`.

`binutils-gdb/include/plugin-api.h` defines the plugin API.

Despite the name of `LLVMgold.so` containing gold, the file can be used by GNU binutils (ld, gold, nm, ar) and mold.

### `--verbose`

GNU ld dumps a linker script (either internal or external) with this option.
gold, ld.lld, and mold are not linker script driven. There is no linker script output.

## Address related

### `-Ttext-segment`

The text segment is traditionally the first segment. Users who specify `-Ttext-segment` may actually want to specify the image base.
The option has strange semantics (likely a bug) when `-z separate-code` is used together: <https://sourceware.org/bugzilla/show_bug.cgi?id=25207>.

ld.lld provides `--image-base` to set the image base.
GNU ld's ELF port has implemented `

GNU ld's PE/COFF port has supported `--image-base` for a long time and implemented the option for ELF in the binutils 2.44 release.

This option appears to be used mainly for `mmap` `MAP_FIXED` usage to avoid conflict with ASLR.
The better alternative is to avoid setting a fixed address. qemu `linux-user/elfload.c:probe_guest_base` may give some insight.

## Target-specific

### `--cmse-implib`, `--out-implib=out.lib`

See [Linker notes on AArch32](/blog/2023-04-23-linker-notes-on-aarch32)

# 中文版

解析GNU風味的linker options

## 編譯器driver options

在描述鏈接器選項前先介紹一下driver options。通常使用`gcc`或`clang`，指定的都是driver options。一些driver options會影響傳遞給鏈接器的選項。
有些driver options和鏈接器重名，它們往往在傳遞給鏈接器同名選項之外還有額外功效，比如：

<!-- more -->

* `-shared`: 不設置`-dynamic-linker`，不鏈接`crt1.o`
* `-static`: 不設置`-dynamic-linker`，使用`crtbeginT.o`而非`crtbegin.o`，使用`--start-group`鏈接`-lgcc -lgcc_eh -lc`(它們有(不好的)循環依賴)

`-Wl,--foo,value,--bar=value`會傳遞`--foo`、`value`、`--bar=value`三個選項給鏈接器。
如果有大量鏈接選項，可以每行一行放在一個文本文件`response.txt`裏，然後指定`-Wl,@response.txt`。

注意，`-O2`不會傳遞`-O2`給鏈接器，`-Wl,-O2`則會。

* `-fno-pic,-fno-PIC`是同義的，生成position-dependent code
* `-fpie,-fPIE`分別叫做small PIE、large PIE，在PIC基礎上引入了一個優化：編譯的.o只能用於可執行檔。參見下文的`-Bsymbolic`。
* `-fpic,-fPIC`分別叫做small PIC、large PIC，position-independent code。在32-bit PowerPC和Sparc上(即將退出歷史舞臺的架構)兩種模式有代碼生成差異。大多數架構沒有差異。

## 輸入

鏈接器接受幾類輸入。對於符號，每個輸入文件的符號表都會影響符號解析；對於sections，只有regular object files裏的sections(稱爲input sections)會拼接得到輸出文件的output sections。

* .o (regular object files)
* .so (shared objects): 只影響符號解析
* .a (archive files)

符號解析細節參見參見[Symbol processing](/blog/2021-06-20-symbol-processing)

## 模式

以下四種鏈接模式四選一，控制輸出文件的類型(可執行檔/shared object/relocatable object)：

* `-no-pie` (default): 生成position-dependent executable (`ET_EXEC`)。要求最寬鬆，源文件可用`-fno-pic,-fpie,-fpic`編譯
* `-pie`: 生成position-independent executable (`ET_DYN`)。源文件須要用`-fpie,-fpic`編譯
* `-shared`: 生成position-independent shared object (`ET_DYN`)。最嚴格，源文件須要用`-fpic`編譯
* `-r`: relocatable link，不生成linker synthesized sections，且保留relocations

`-pie`可以和`-shared`都是position-independent的鏈接模式。`-pie`也可以和`-no-pie`都是可執行檔的鏈接模式。
`-pie`和`-shared -Bsymbolic`很相似，但它畢竟是可執行檔，以下行爲和`-no-pie`貼近而與`-shared`不同：

* 允許copy relocation和canonical PLT
* 允許relax General Dynamic/Local Dynamic TLS models和TLS descriptors到Initial Exec/Local Exec
* 會鏈接時解析undefined weak，(ld.lld行爲)不生成dynamic relocation。GNU ld是否生成dynamic relocation有非常複雜的規則，且和架構相關

容易產生混淆的是，編譯器driver提供了幾個同名選項：`-no-pie,-pie,-shared,-r`。
GCC 6引入了configure-time選項`--enable-default-pie`：啓用該選項的GCC預設`-pie`和`-fPIE`。現在，很多Linux發行版都啓用了該選項作爲基礎的security hardening。

The linker is in one of the following four modes.
The mode controls the output type (executable file/shared object/relocatable object).

* `-no-pie` (default): Generate a position-dependent executable (`ET_EXEC`). This mode has the most loose requirements: the source files can be compiled with `-fno-pic, -fpie/-fPIE, -fpic/-fPIC`.
* `-pie`: Generate a position-independent executable (`ET_DYN`). Source files need to be compiled with `-fpie/-fPIE, -fpic/-fPIC`
* `-shared`: Generate a position-independent shared object (`ET_DYN`). The most restrictive mode: source files need to be compiled with `-fpic/-fPIC`.
* `-r`: Generate a relocatable file. This is called a relocatable link and is special. It suppresses various linker synthesized sections and reserves relocations.

Confusingly, the compiler driver provides several options with the same name: `-no-pie, -pie, -shared, -r`.
GCC 6 introduced the configure-time option `--enable-default-pie`: such builds enable `-fPIE` and `-pie` by default.
Now, many Linux distributions have enabled this option as the basic security hardening.

### 可執行檔(`-no-pie`和`-pie`)

A defined symbol is non-preemptible.
For a branch instruction using a PLT-generating relocation, the branch can be bound directly the the definition, avoiding a PLT.
For a code sequence involving a GOT-generating relocation, the code sequence may be optimized to use direct access. See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table) for detail.

A non-local `STV_DEFAULT/STV_PROTECTED` defined symbol is by default not exported to the dynamic symbol table.

#### `-no-pie`

`-no-pie` indicates that the link-time address equals the run-time address.
This property is leveraged by a linker: all relocations referencing a non-preemptible symbol can be resolved, including absolute GOT-generating (e.g. `R_AARCH64_LD64_GOT_LO12_NC`), PC-relative GOT-generating (e.g. `R_X86_64_REX_GOTPCRELX`), etc.
In the absence of a GOT optimization, the GOT entry for a non-preemptible symbol is a constant, avoiding a dynamic relocation.
The image base is an arch-specific non-zero value by default.

* Some architectures have different PLT code sequences (i386, ppc32 .glink).
* `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` can be further optimized
* ppc64 `.branch_lt` (long branch addresses) can be optimized

#### `-pie`

`-pie` is very similar to `-shared -Bsymbolic`, but it produces an executable file. The following behavior is close to `-no-pie` but different from `-shared`:

* Allow copy relocation and canonical plt.
* Allow relax general dynamic/local dynamic tls models and tls descriptors to initial exec/local exec.
* Will resolve undefined weak symbols to zeroes. ld.lld does not generate dynamic relocation. Whether GNU ld generates dynamic relocation has very complicated rules and is architecture dependent.

### Shared object link (`-shared`)

A non-local `STV_DEFAULT` definition is preemptible (interposable) by default, that is, the definition may be replaced by the definition in the executable file or another shared object at runtime.
The compiler and the linker cooperate by using GOT and PLT entries to reference such symbols.

A non-local `STV_DEFAULT/STV_PROTECTED` symbol is exported to the dynamic symbol table.
E.g. in the following program, `foo` is exported to the dynamic symbol table if linked with `-shared` but (by default) not if linked with `-no-pie` or `-pie`.
```c
void foo() {}
```

### PIC link (`-pie` and `-shared`)

A symbolic relocation (absolute relocation & width matches the word size) referencing a non-preemptible non-TLS symbol converts to a relative relocation.

See [Relative relocations and RELR](/blog/2021-10-31-relative-relocations-and-relr) for detail.

### Archive member selection

.a文件具有archive member selection的特殊語義。每個成員都是惰性的。
如果鏈接器發現.a中的某個archive member定義了某個之前被引用但尚未定義的符號，則會從archive中pull這個member。
該member會在概念上成爲一個regular object file，其符號表被用於符號解析，且貢獻input sections，之後的處理方式就和.o沒有任何差異了。

若該archive不能滿足之前的某個undefined符號，GNU ld和gold會跳過該archive，詳見`--warn-backrefs`。

Thin archive的鏈接語義和regular archive相同。

`--start-group`可以改變archive member selection語義。
`--whole-archive`可以取消archive member selection，還原object file語義。

## 模式

## 符號相關

### `-Bsymbolic`

In an ELF shared object, a defined non-local `STV_DEFAULT` symbol is preemptible (interposable) by default, that is, the definition may be replaced by the definition in the executable file or another shared object at runtime.
A definition in an executable file is guaranteed to be non-preemptible (non-interposable).

The linker provides several mechanisms to make non-local `STV_DEFAULT` definitions in a shared object non-preemptible, similar to `-no-pie, -pie`.

* `-Bsymbolic` makes all definitions except those matched by `--dynamic-list/--export-dynamic-symbol-list/--export-dynamic-symbol`) non-preemptible
* `-Bsymbolic-functions` is similar to `-Bsymbolic`, but only applies to `STT_FUNC` definitions

See [ELF interposition and -Bsymbolic](/blog/2021-05-09-elf-interposition-and-bsymbolic) for detail.

### `--exclude-libs`

If a matched archive defines a non-local symbol, don't export this symbol.

### `--export-dynamic`

Shared objects預設導出所有non-local `STV_DEFAULT/STV_PROTECTED`定義符號到dynamic symbol table。可執行檔可用`--export-dynamic`模擬shared objects行爲。

下面描述可執行檔/shared object裏一個符號被導出的規則(logical AND)：

* non-local `STV_DEFAULT/STV_PROTECTED` (this means it can be hid by `--exclude-libs`)
* logical OR of the following:
  + undefined
  + (`--export-dynamic` || `-shared`) && ! (unnamed_addr linkonce_odr GlobalVariable || local_unnamed_addr linkonce_odr constant GlobalVariable)
  + matched by `--dynamic-list/--export-dynamic-symbol-list/--export-dynamic-symbol`
  + defined or referenced by a shared object as `STV_DEFAULT`
  + `STV_PROTECTED` definition in a shared object preempted by copy relocation/canonical PLT when `--ignore-{data,function}-address-equality}` is specified
  + `-z ifunc-noplt` && has at least one relocation

如果可執行檔定義了在某個鏈接時shared object引用了一個符號，那麼鏈接器需要導出該符號，使得運行時該shared object的undefined符號可以綁定到可執行檔中的定義。

### `--export-dynamic-symbol=glob`, `--export-dynamic-symbol-list`, and `--dynamic-list`

These options have different semantics for an executable and a shared object.

* executable: put matched non-local defined symbols to the dynamic symbol table (`--export-dynamic` applies to all non-local defined symbols.)
* shared object: rerences to matched non-local `STV_DEFAULT` symbols shouldn't be bound to definitions within the shared object even if they would otherwise be due to `-Bsymbolic`, `-Bsymbolic-functions`, or `--dynamic-list`

`--dynamic-list` additionally implies `-Bsymbolic`.

For the shared object case, I usually call this "make a symbol preemptible": even if a symbolic intention option (`-Bsymbolic`, `-Bsymbolic-functions`, or `--dynamic-list` is in action, a matched symbol is NOT bound locally.

One may use `--export-dynamic-symbol=foo*` to match all non-local `STV_DEFAULT` symbols `foo*`. ld.lld before 11 uses an exact match instead of a glob.

`--export-dynamic-symbol-list` is implemented since [GNU ld 2.35](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=37a141bfed4dd3c33d77c15dfde00e4b4f5b24c7) and [ld.lld 14](https://reviews.llvm.org/D107317).

### `--discard-none`, `--discard-locals`, and `--discard-all`

如果輸出`.symtab`，一個live section裏定義的local符號被保留的條件是：

```
if ((--emit-reloc or -r) && referenced) || --discard-none
  return true
if --discard-all
  return false
if --discard-locals
  return is not .L
# No --discard-* is specified.
return not (.L in a SHF_MERGE section)
```

### `--no-undefined-version`

If a version script specifies an exact pattern which does not match a defined symbol, report an error.

Say we have the following version script, if `foo` is not a defined symbol, the linker will report an error.
For a glob pattern (e.g. `bar*`) matching no symbol, there is no error. This is a compromise.
```
v1 {
  foo;
  bar*;
};
```

GNU ld has supported `--no-undefined-version` since [2002-08](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=3194163592008a9e575d577921647ab91b09c77b), but `--undefined-version` was a late addition in 2022-10 (milestone: binutils 2.40).

### `--strip-all`

不要創建`.strtab`和`.symtab`。

### `-u symbol`

若某個archive file定義了`-u`指定的符號則pull(由archive file轉換爲object file，之後該文件就和一般的.o相同)。

比如：`ld -u foo ... a.a`。若`a.a`不定義被之前object files引用的符號，`a.a`不會被pull。
如果指定了`-u foo`，那麼`a.a`中定義了`foo`的archive member會被pull。

`-u`的另一個作用是指定一個GC root。

### `--version-script=script`

Version script有三個用途：

* 定義versions
* 指定一些模式，使得匹配的、定義的、unversioned的符號具有指定的version
* Local version：`local:`可以改變匹配的、定義的、unversioned的符號的binding爲`STB_LOCAL`，不會導出到dynamic symbol table

[Symbol versioning](/blog/2020-11-26-symbol-versioning)描述了具體的symbol versioning機制。

### `-y symbol`

常用於調試。輸出指定符號在哪裏被引用、哪裏被定義。

### `-z muldefs`

允許重複定義的符號。鏈接器預設不允許兩個同名的non-local regular definitions(非weak、非common)。

### `-z unique-symbol`

Rename local symbols so that there are no duplicates.

Some Intel folks were working on function granular kernel address space layout randomization and wanted such a feature (<https://sourceware.org/bugzilla/show_bug.cgi?id=26391>).
GNU ld since 2.36 supports this option.
I closed the ld.lld [feature request](https://bugs.llvm.org/show_bug.cgi?id=50745).

I don't think this is a good design.

First, the stability problem.
Say, the old kernel has `foo.1 foo.2`.
If there is a new local `foo` symbol, the new kernel will have `foo.1 foo.2 foo.3`.
However, The new symbols don't necessarily correspond to local symbols of the same names in the old kernel.
Such disturbence is probably more likely with LTO or PGO.
For Clang LTO, the kernel Makefile currently specifies `-mllvm -import-instr-limit=5`.
If a function close to the boundary happens to cross the boundary, if inlined into other translation units, the stability issue may affect many translation units.

The implementation has to perform an iteration on all local symbols, which can affect link speed.

In addition, the `.[0-9]+` scheme has been used by C++ mangling.
The Itanium C++ ABI says "A <mangled-name> containing a period represents a vendor-specific version or portion of the entity named by the <encoding> prior to the first period. There is no restriction on the characters that may be used in the suffix following the period."
In GNU, this is used to represent function cloning.

```text
% c++filt <<< $'_ZL3foov\n_ZL3foov.1'
foo()
foo() [clone .1]
```

As an alternative, I suggest that the FGASLR developer uses the `STT_FILE` symbol:
```text
STT_FILE    a.c
STT_NOTYPE  foo
STT_FILE    b.c
STT_NOTYPE  foo
```

The ELF specification says:

> Conventionally, the symbol's name gives the name of the source file associated with the object file. A file symbol has STB_LOCAL binding, its section index is SHN_ABS, and it precedes the other STB_LOCAL symbols for the file, if it is present.

I mentioned my concern on a reply to [``[PATCH v9 02/15] livepatch: use `-z unique-symbol` if available to nuke pos-based search``](https://lore.kernel.org/all/20211223002209.1092165-3-alexandr.lobakin@intel.com/).

## Library相關

### `--as-needed` and `--no-as-needed`

防止一些沒有用到的鏈接時shared objects留下`DT_NEEDED`。

`--as-needed`和`--no-as-needed`是position-dependent選項(非正式叫法，但沒找到更貼切的形容詞)，影響後面命令行出現的shared objects。一個shared object is needed，如果下面條件之一成立：

* 在命令行中至少一次出現在`--no-as-needed`模式下
* 定義了一個被.o live section non-weak引用的符號。也就是說，weak定義仍可能被認爲是unneeded。--gc-sections丟棄的section的引用不算

### `-Bdynamic` and `-Bstatic`

這兩個選項是position-dependent選項，影響後面命令行出現的`-lname`。

* `-Bdynamic` (default)：在`-L`指定的目錄列表中查找`libfoo.so`和`libfoo.a`
* `-Bstatic`：在`-L`指定的目錄列表中查找`libfoo.a`

注意，歷史上GNU ld裏`-Bstatic`和`-static`同義。編譯器driver的`-static`是個不同的選項，除了傳遞`-static`給ld外，還會去除預設的`--dynamic-linker`，影響libgcc libc等的鏈接。

### `--no-dependent-libraries`

忽略object files裏的`.deplibs` section。

### `-soname=name`

設置生成的shared object的dynamic table中的`DT_SONAME`。

鏈接器會記錄鏈接時shared objects，在生成的可執行檔/shared object的dynamic table中用一條`DT_NEEDED`記錄描述每一個鏈接時shared object。

* 若該shared object含有`DT_SONAME`，該字段提供`DT_NEEDED的值
* 否則，若通過`-l`鏈接，值爲去除目錄後的文件名
* 否則值爲路徑名(絕對/相對路徑有差異)

比如：`ld -shared -soname=a.so.1 a.o -o a.so; ld b.o ./a.so`，`a.out`的`DT_NEEDED`爲`a.so.1`。如果第一個命令不含`-soname`，則`a.out`的`DT_NEEDED`爲`./a.so`。

### `--start-group` and `--end-group`

如果`A.a`和`B.a`有相互引用，且不能確定哪一個會被先pull into the link，得使用這對選項。下面給出一個例子：

對於一個archive鏈接順序：`main.o A.a B.a`，假設`main.o`引用了`B.a`，而`A.a`沒有滿足之前的某個undefined符號，那麼該鏈接順序會導致錯誤。
鏈接順序換成`main.o B.a A.a`行不行呢？如果`main.o`變更後引用了`A.a`，而`B.a`沒有滿足之前的某個undefined符號，那麼該鏈接順序也會導致錯誤。

一種解決方案是`main.o A.a B.a A.a`。很多情況下重複一次就夠了，但是假如鏈接第一個A.a時僅加載了`A.a(a.o)`，鏈接B.b時僅加載了`B.a(b.o)`，鏈接第二個A.a時僅加載了`A.a(c.o)`且`A.a(c.o)`需要`B.a`中的另一個member，該鏈接順序仍會導致undefined symbol錯誤。

我們可以再重複一次`B.a`，即`main.o A.a B.a A.a B.a`，但更好的解決方案是`main.o --start-group A.a B.a --end-group`。

### `--start-lib` and `--end-lib`

gold發明的很有用的功能，可以代替thin archive。使regular object files有類似archive files的語義(按需加載)。

下文的`--whole-archive`用於.a，而`--start-lib`則用於`.o`：

`ld ... --start-lib b.o c.o --end-lib`作用類似`ld ... a.a`，如果`a.a`包含`b.o c.o`。

我提交了一個GNU ld的feature request：<https://sourceware.org/bugzilla/show_bug.cgi?id=24600>

### `--sysroot`

和GCC/Clang driver的`--sysroot`不同。如果一個linker script在sysroot目錄下，它打開絕對路徑文件(`INPUT` or `GROUP`)時，在絕對路徑前加上sysroot。

### `--whole-archive` and `--no-whole-archive`

`--whole-archive`選項後的.a會當作.o一樣處理，沒有惰性語義。
如果`a.a`包含`b.o c.o`，那麼`ld --whole-archive a.a --no-whole-archive`和`ld b.o c.o`作用相同。

### `--push-state` and `--pop-state`

`-Bstatic, --whole-archive, --as-needed`等都是表示boolean狀態的position-dependent選項。`--push-state`可以保存這些選項的boolean狀態，`--pop-state`則會還原。

在鏈接命令行插入新選項裏變更狀態時，通常希望能還原，這個時候就可以用`--push-state`和`--pop-state`。
比如確保鏈接`libc++.a`和`libc++abi.a`可以用`-Wl,--push-state,-Bstatic -lc++ -lc++abi -Wl,--pop-state`。

## 依賴關係相關

### `-z defs` and `-z undefs`

遇到來自regular objects的不能解析的undefined符號(不能在鏈接時綁定到可執行檔或一個鏈接時shared object中的定義)，是否報錯。可執行檔預設爲`-z defs/--no-undefined`(不允許)，而shared objects預設爲`-z undefs`(允許)。

很多構建系統會啓用`-z defs`，要求shared objects在鏈接時指定所有依賴(link what you use)。

### `--allow-shlib-undefined` and `--no-allow-shlib-undefined`

遇到來自shared objects的不能解析的undefined符號，是否報錯。可執行檔預設爲`--no-allow-shlib-undefined`(不允許)，而shared objects預設爲`--allow-shlib-undefined`(允許)。

對於如下代碼，鏈接可執行檔時會報錯：
```c
// a.so
void f();
void g() { f(); }

// exe
void g()
int main() { g(); }
```

如果啓用`--allow-shlib-undefined`，鏈接會成功，但ld.so會在運行時報錯，在glibc中爲：`symbol lookup error: ... undefined symbol: `。

GNU ld有個複雜的算法查找transitive closure，只有transitive closure的shared objects都無法解析一個undefined符號時纔會報錯。
gold和ld.lld使用一個簡化的規則：如果一個shared object的所有`DT_NEEDED`依賴都被直接鏈接了，則啓用報錯；如果部分依賴沒有被鏈接，那麼gold/ld.lld無法準確判斷是否一個未被直接鏈接的shared object能提供定義，就保守地不報錯。

值得一提的是，`-z defs/-z undefs/--no-undefined`和`--[no-]allow-shlib-undefined`可以被一個選項`--unresolved-symbols`控制。

### `--warn-backrefs`

ld.lld特有，參見<http://lld.llvm.org/ELF/warn_backrefs.html>。

## Layout相關

### `--no-rosegment`

ld.lld採用兩個RW `PT_LOAD`的設計：

* R `PT_LOAD`
* RX `PT_LOAD`
* RW `PT_LOAD`(和`PT_GNU_RELRO`重疊)
* RW `PT_LOAD`

指定該選項可以合併R `PT_LOAD`和RX `PT_LOAD`。

### `-z separate-loadable-segments`

ld.lld傳統佈局：所有`PT_LOAD` segments都沒有重疊(一個字節不會被同時加載到兩個memory mappings)。

實現方式是每個新`PT_LOAD`的地址對齊到max-page-size。ld.lld預設有4個`PT_LOAD`(R,RX,RW(RELRO),RW(non-RELRO))，在輸出文件裏三次對齊都可能浪費一些字節。
在AArch64和PowerPC上因爲ABI指定的max-page-size較大(65536)，最多可浪費65536*3字節。

### `-z separate-code`

binutils 2.31引入，在Linux/x86上爲預設。GNU ld採用：

* R `PT_LOAD`
* RX `PT_LOAD`
* R `PT_LOAD`
* RW `PT_LOAD`
  + 前綴部分爲`PT_GNU_RELRO`
  + 非`PT_GNU_RELRO`的部分

`separate-code`的含義是文件中一個被映射到可執行段的字節(RX `PT_LOAD`)不會被同時映射到一個R `PT_LOAD`。
注意RX後的R是不憂的，理想情況是把這個R和第一個R合併，但似乎在GNU ld裏實現會很困難。

我在ld.lld 10引入該選項，語義和GNU ld類似但佈局不同(沒有必要模仿兩個R的非優佈局)：兩個RW `PT_LOAD`允許重疊，也就是說第二個`PT_LOAD`的地址不用對齊，最多可浪費max-page-size*2字節。

### `-z noseparate-code`

經典佈局，允許可執行段和其他`PT_LOAD`重疊。GNU ld通常用：

* RX `PT_LOAD`
* RW `PT_LOAD`
  + 前綴部分爲`PT_GNU_RELRO`。這部分在ld.so解析完dynamic relocations後mprotect成readonly
  + 非`PT_GNU_RELRO`的部分。這部分在運行時始終可寫

第一個`PT_LOAD`常被籠統的稱爲text segment，實際上不準確：非執行部分的rodata也在裏面。

ld.lld 10中預設使用這種佈局，不需要對齊任何`PT_LOAD`。

## Relocation相關

### `--apply-dynamic-relocs`

對於psABI採用RELA的architectures(AArch64,PowerPC,RISC-V,x86-64,etc)，因爲dynamic relocations包含addend字段，鏈接器在被relocate的地址填上0，而不是addend值。
如果可執行檔/shared objects使用壓縮，能稍稍利於壓縮。

`--apply-dynamic-relocs` is supported for all ports in ld.lld.
As of August 2023, [only the aarch64 port](https://sourceware.org/PR25891) of GNU ld supports `--apply-dynamic-relocs`.

### `--emit-relocs`

可用於`-no-pie/-pie/-shared`獲得類似`-r`的效果：保留輸入的relocations。可用於鏈接後的二進制分析，我知道的唯二用途是Linux kernel x86的`CONFIG_RELOCATABLE`和BOLT。

### `--pack-dyn-relocs=value`

`relr`可以啓用`DT_RELR`，一種更加緊湊的relative relocation (`R_*_RELATIVE`)編碼方式。Relative relocations常見於`-pie`鏈接的可執行檔。

### `-z text` and `-z notext`

`-z text`不允許text relocations。
`-z notext`允許text relocations。

binutils 2.35起，Linux/x86上的GNU ld預設啓用configure-time選項`--enable-textrel-check={warning,error}`，若有text relocations會給出warning/error。

Text relocations這個概念的用詞不準確，實際含義是作用在readonly sections上的dynamic relocations的總稱。
.o中的relocations如果不能在鏈接時確定值，就需要轉換成dynamic relocations在運行時由ld.so計算(type和.o中相同)。
如果作用的section沒有`SHF_WRITE`標誌，ld.so就得臨時執行`mprotect`變更memory maps的權限、修改、再還原之前的只讀權限，這樣就妨礙了page sharing。

Shared objects形成text relocations的情況比可執行檔多。
可執行檔有canonical PLT和copy relocations可以避免某些text relocations。

不同鏈接器在不同架構上允許的text relocations的relocation types不同。GNU ld會允許一些glibc ld.so支持的types。
在x86-64上，鏈接器都會允許`R_X86_64_64`和`R_X86_64_PC64`。

下面的彙編程序裏`defined_in_so`是定義在某個shared object的符號。註釋裏給出每種text relocation的場景。

```asm
.globl global
global:
local:
  .quad local              # (-pie or -shared) R_X86_64_RELATIVE
  .quad global             # (-pie) R_X86_64_RELATIVE or (-shared) R_X86_64_64
  .quad defined_in_so      # (-shared) R_X86_64_64
  .quad defined_in_so - .  # (-shared) R_X86_64_PC64
```

在`-no-pie`或`-pie`模式下，根據`defined_in_so`的符號類型，鏈接器會作出不同選擇：

* `STT_FUNC`: 產生canonical PLT
* `STT_OBJECT`: 產生copy relocation
* `STT_NOTYPE`：GNU ld會產生copy relocation。ld.lld會產生text relocation

## Section相關

### `--gc-sections`

非常常見的選項。編譯時指定`-ffunction-sections`或`-fdata-sections`纔有效果。鏈接器會做liveness analysis從輸出中去除沒有用的sections。

GC roots:

* `--entry/--init/--fini/-u`指定的所有定義符號所在的sections
* Linker script表達式被引用的定義符號所在的sections
* `.dynsym`中的所有定義符號所在的sections
* 類型爲`SHT_PREINIT_ARRAY/SHT_INIT_ARRAY/SHT_FINI_ARRAY`
* 名稱爲`.ctors/.dtors/.init/.fini/.jcr`
* 不在section group中的`SHT_NOTE`(這個section group規則是爲了Fedora watermark)
* 被`.eh_frame`引用的personality routines和language-specific data area

### `--icf=all` `--icf=safe`

啓用Identical Code Folding。這個名稱其實不準確：(1) 適用於readonly data；(2) 合併的單位是section，而不是函數。

對於一組相同的sections，選擇一個作爲代表，丟棄其餘的sections，然後把relocation重定向到代表section。

gold實現了基於relocation的`--icf=safe`；ld.lld實現了基於LLVM address significance table的`--icf=safe`。

### `--symbol-ordering-file=<file>`

指定一個文本文件，每行一個定義的符號。如果符號A在符號B前面，那麼在每一個input section description進行排序，A所在的section排在B所在的section前面。

如果一個符號未定義，或者所在的section被丟棄，鏈接器會輸出一個warning，除非指定了`--no-warn-symbol-ordering`。

如果一個函數頻繁調用另一個，在linked image中如果讓兩個函數所在的input sections接近，可以增大它們落在同一個page的概率，減小page working set及減少TLB thrashing。參見Karl Pettis and Robert C. Hansen的 *Profile Guided Code Positioning*

這個選項是ld.lld特有的。gold有一個`--section-ordering-file`，根據section name排序。實踐中要求text/data sections具有不同的名字(不可使用`clang -funique-section-names`)。
而基於符號名排序則可以使用`-funique-section-names`。

## 分析相關

### `--cref`

輸出cross reference table。對於每一個non-local符號，輸出定義的文件和被引用的文件列表。

### `-M` and `-Map=<file>`

輸出link map，可以查看output sections的地址、文件偏移、包含的input sections。

## Warning相關

### `--fatal-warnings`

把warnings轉成errors。Warning和error的差別除了是否包含`warning`或`error`字串外更重要的一點是，error會阻止輸出鏈接結果。

### `--noinhibit-exec`

把部分errors轉成warnings。注意不要指定`--fatal-warnings`把降級的warnings再升級爲errors:)

## 其他

### `--build-id=value`

生成`.note.gnu.build-id`，標識一個鏈接結果。一般用SHA-1。鏈接器會給`.note.gnu.build-id`的區域填零，散列每個字節後把結果填回`.note.gnu.build-id`。
每個鏈接器用的計算方式各有不同。

### `--compress-debug-sections=[zlib|zstd]`

用zlib壓縮輸出文件的`.debug_*` sections，並標記`SHF_COMPRESSED`。`SHF_COMPRESSED`是合併入ELF specification的最後一個feature，之後ELF specification就處於不被維護的狀態……

### `--hash-style`

`--hash-style=sysv`指定ELF specification定義的`DT_HASH`，一個用於加速符號解析的hash table。
`DT_GNU_HASH`在空間佔用和效率都優於`DT_HASH`。
指的一提的是Mips有個`DT_MIPS_XHASH`(Mips ABI設計聰明反被聰明誤的好例子)，我個人覺得在解決一個錯誤的問題。實際上有辦法用`DT_GNU_HASH`，但可能Mips社區的人覺得東西塞進去了就不想多管了。

### `--no-ld-generated-unwind-info`

參見[PR12570 .plt has no associated .eh_frame/.debug_frame](https://sourceware.org/bugzilla/show_bug.cgi?id=12570)。

PC在PLT entry中時，如果鏈接器不合成`.eh_frame`信息，unwinder可能會無法正確unwind。
在i386和x86-64上，lazy binding狀態下，一個PLT entry的首次調用會執行push指令。在ESP/RSP改變後，如果PLT entry沒有`.eh_frame`提供的unwind信息，unwinder可能會無法正確unwind，影響profiler精度。

```asm
jmp *got(%rip)
pushq $0x0
jmpq .plt
```

However, I think this feature is obsoleted and irrelevant nowadays.
To recognize the PLT name, a profiler needs to do:

* Parse the `.plt` section to know the region of PLT entries
* Parse `.rel[a].plt` to get `R_*_JUMP_SLOT` dynamic relocations and their referenced symbol names.
* If the current PC is within the PLT region, parse nearly instructions and find the GOT load. The associated `R_*_JUMP_SLOT` identifies the symbol name.
* Concatenate the symbol name and `@plt` to form `foo@plt`

Note: `foo@plt` is a convention used by some tools, but it is not a name in the symbol table.

GDB有heuristics可以識別這種情況。

這個問題不會影響C++ exception。PLT entry是tail call，`__cxa_throw`調用的`_Unwind_RaiseException`會穿透ld.so resolver和PLT entry的tail calls。
PC會還原爲PLT entry的caller的下一條指令。

```cpp
// b.cc - b.so
void ext() { throw 3; }

// a.cc - exe
#include <stdio.h>

void ext();
void foo() {
  try {
    ext(); // PLT entry
  } catch (int x) {
    printf("%d\n", x);
  }
}

int main() {
  foo();
}
```

### `-O`

優化等級，和編譯器driver選項`-O`不同。

在ld.lld中，`-O0`禁用`SHF_MERGE`的常量合併；`-O2`啓用`SHF_MERGE|SHF_STRINGS`的string suffix merge，`--compress-debug-sections=zlib`使用較高壓縮比的zlib壓縮。

### `-plugin file`

GNU ld和gold支持這個選項加載GCC LTO插件(`liblto_plugin.so`)或LLVM LTO插件(`LLVMgold.so`)

插件的API接口由`binutils-gdb/include/plugin-api.h`定義。

注意，`LLVMgold.so`的名稱含gold，但也能用於GNU binutils (ld, gold, nm, ar)和mold。
