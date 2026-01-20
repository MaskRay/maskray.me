---
layout: post
title: Why isn't ld.lld faster?
author: MaskRay
tags: [linker,llvm]
---

Updated in 2022-11.

LLD is the LLVM linker. Its ELF port is typically installed as `ld.lld`. This article makes an in-depth analysis of ld.lld's performance.
The topic has been in my mind for a while.
Recently Rui Ueyama released [mold](https://github.com/rui314/mold) 1.0 and people wonder why with multi-threading its ELF port is faster than ld.lld.
So I finally completed the article.

First of all, I am very glad that Rui Ueyama started mold.
Our world has a plethora of compilers, but not many people learn or write linkers.
As its design documentation says, there are many drastically different designs which haven't been explored.
In my view, mold is innovative in that it introduced parallel symbol table initialization, symbol resolution, and relocation scan which to my knowledge hadn't been implemented before, and showed us amazing results.
The innovation gives existing and future linkers incentive to optimize further.

<!-- more -->

TODO At some point I will move large portion of "how to implement an ELF linker" to a separate article.

## Generic ELF linker passes

Before jumping into the analysis, let me introduce the generic passes to be referenced by subsequent chapters.

Here are the passes a typical linker has.

* Parse command line options
* Find and scan input files (.o, .so, .a), interleaved with symbol resolution
* Global transforms (section based garbage collection, identical code folding, etc)
* Create synthetic sections
* Map input sections and synthetic (linker-generated) sections into output sections
* Scan relocations
* Finalize synthetic sections
* Layout and assign addresses (thunks, sections with varying sizes, symbol assignments)
* Write headers
* Copy section contents to output and resolve relocations

## ld.lld

In the past people advocated "ld.lld focuses on speed".
This impression was mainly driven by the comparison with GNU ld and gold.
People say that ld.lld is at least 2x faster than gold, sometimes 4x or more.

As I have observed the codebase and code review activity, the focus is probably more on parity on important features, ideal semantics, code readability, portability and generality, but less on performance and specifically multi-threading performance.
In some cases, simpler code is preferred over little performance improvement.

ld.lld has parallelism in a few places like writing sections and computing the content for a few synthetic sections, but some critical paths do not have parallelism.
If you experiment with `--threads=N`, you may find that the speed-up is not large, as shown by the Chrome example below.

### Linking Chrome

In Chrome, `chrome` is a large executable. In the default build configuration (no debug information), (circa Dec 2021) there are 8905295 sections (not counting the NULL section at index zero), 3325621 discarded COMDAT sections, 1372104 input sections (mostly `SHT_PROGBITS`, not counting `SHT_RELA/SHT_SYMTAB` or discarded COMDAT sections).

```text
# Input relocations for chrome (no debug information)
R_X86_64_PLT32: 167558
R_X86_64_64: 381017
R_X86_64_32: 39455
R_X86_64_GOTPCREL: 28125
R_X86_64_REX_GOTPCRELX: 3
```

Below I will compare lld 13.0.1 with mold 1.0.1.
mold's build system uses mimalloc by default. For a fair comparison, I link mimalloc into lld (`-DCMAKE_EXE_LINKER_FLAGS=-Wl,--push-state,$HOME/Dev/mimalloc/out/release/libmimalloc.a,--pop-state`).
I build both lld and mold with `-O3 -fno-exceptions -fno-pic -no-pie`.
The default build of Chrome uses `--gdb-index`, which is ignored by mold but takes some time for ld.lld. We should remove it for a fair comparison.

```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-20 "{/tmp/link-with-lld,/tmp/link-with-mold}" --threads=1"
Benchmark 1: numactl -C 20-20 /tmp/link-with-lld --threads=1
  Time (mean ± σ):      8.314 s ±  0.007 s    [User: 6.118 s, System: 2.190 s]
  Range (min … max):    8.304 s …  8.326 s    16 runs

Benchmark 2: numactl -C 20-20 /tmp/link-with-mold --threads=1
  Time (mean ± σ):     12.684 s ±  0.009 s    [User: 0.063 s, System: 0.004 s]
  Range (min … max):   12.670 s … 12.706 s    16 runs

Summary
  'numactl -C 20-20 /tmp/link-with-lld --threads=1' ran
    1.53 ± 0.00 times faster than 'numactl -C 20-20 /tmp/link-with-mold --threads=1'

% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-21 "{/tmp/link-with-lld,/tmp/link-with-mold}" --threads=2"
Benchmark 1: numactl -C 20-21 /tmp/link-with-lld --threads=2
  Time (mean ± σ):      7.551 s ±  0.099 s    [User: 6.249 s, System: 2.292 s]
  Range (min … max):    7.369 s …  7.728 s    16 runs

Benchmark 2: numactl -C 20-21 /tmp/link-with-mold --threads=2
  Time (mean ± σ):      6.519 s ±  0.019 s    [User: 0.064 s, System: 0.006 s]
  Range (min … max):    6.502 s …  6.578 s    16 runs

Summary
  'numactl -C 20-21 /tmp/link-with-mold --threads=2' ran
    1.16 ± 0.02 times faster than 'numactl -C 20-21 /tmp/link-with-lld --threads=2'

% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-23 "{/tmp/link-with-lld,/tmp/link-with-mold}" --threads=4"
Benchmark 1: numactl -C 20-23 /tmp/link-with-lld --threads=4
  Time (mean ± σ):      6.981 s ±  0.036 s    [User: 6.441 s, System: 2.393 s]
  Range (min … max):    6.899 s …  7.046 s    16 runs

Benchmark 2: numactl -C 20-23 /tmp/link-with-mold --threads=4
  Time (mean ± σ):      3.511 s ±  0.006 s    [User: 0.066 s, System: 0.006 s]
  Range (min … max):    3.501 s …  3.521 s    16 runs

Summary
  'numactl -C 20-23 /tmp/link-with-mold --threads=4' ran
    1.99 ± 0.01 times faster than 'numactl -C 20-23 /tmp/link-with-lld --threads=4'

% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "{/tmp/link-with-lld,/tmp/link-with-mold}" --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/link-with-lld --threads=8
  Time (mean ± σ):      6.635 s ±  0.036 s    [User: 6.573 s, System: 2.436 s]
  Range (min … max):    6.569 s …  6.684 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/link-with-mold --threads=8
  Time (mean ± σ):      2.034 s ±  0.008 s    [User: 0.066 s, System: 0.006 s]
  Range (min … max):    2.019 s …  2.043 s    16 runs

Summary
  'numactl -C 20-27 /tmp/link-with-mold --threads=8' ran
    3.26 ± 0.02 times faster than 'numactl -C 20-27 /tmp/link-with-lld --threads=8'
```

With `--threads=1`, ld.lld is faster than mold.
This may be due to mold's cost on splitting symbol processing passes.
However, ld.lld has very few parallel passes and leverage little from multi-threading, so mold get faster with more threads.

Let's examine the time spent on each pass of ld.lld.
I measured the following with `ld.lld --threads={1,8} --time-trace` and `jq -r '.traceEvents[] | select(.name|startswith("Total")) | "\(.dur/1000000) \(.name)"'`.
The unindented lines indicate the `--threads=1` run while the indented lines indicate what `--threads=8` can parallelize.

```text
8.143089 Total ExecuteLinker
8.055836 Total Link
3.172682 Total Parse input files
2.96519 Total Write output file
  --threads=8: 1.40027 Total Write output file
0.409991 Total Scan relocations
0.24452 Total Assign sections
0.176667 Total Finalize synthetic sections
0.167634 Total Split sections
  --threads=8: 0.033665 Total Split sections
0.149345 Total Add local symbols
0.142135 Total Process symbol versions
0.132731 Total LTO
0.119289 Total Finalize .eh_frame
0.101507 Total Add symbols to symtabs
0.100241 Total Merge/finalize input sections
0.086993 Total Load input files
0.064616 Total Finalize address dependent content
0.041352 Total Redirect symbols
0.03537 Total markLive
0.028474 Total Aggregate sections
0.026615 Total Strip sections
0.022518 Total Sort sections
0.021364 Total Combine EH sections
0.014919 Total Demote shared symbols
0.014668 Total Replace common symbols
0.000454 Total Locate library
0.000264 Total Read linker script
7.9e-05 Total Create output files
0 Total Resolve SHF_LINK_ORDER
0 Total Diagnose orphan sections
```

You can map these metrics to the generic linker passes.

Note that with 8 threads, "Parse input files" took 48% time. This is significant.
mold has demonstrated by parallelizing this pass we can get a noticeable speed-up.

### Linking other executables

In Chrome, symbol table initialization and symbol resolution appear to take a significant portion of time.

In another large executable with no debug information, there are 8381112 sections, 1222292 discarded COMDAT sections, and 3269857 input sections.
I measured lower "Parse input files" contribution.

* 32% Parse input files
* 15% Write output
* 14% Scan relocations
* 12% `--gc-sections`
*  5% Load input files
*  2% Merge/finalize input sections

In `-DCMAKE_BUILD_TYPE=RelWithDebInfo` build of Clang, (circa Dec 2021) there are 689777 sections, 140402 discarded COMDAT sections, 229044 input sections.
```text
# Input relocations for no-pie RelWithDebInfo clang
R_X86_64_64: 1253965
R_X86_64_32: 1203807
R_X86_64_GOTPCREL: 19597
R_X86_64_PLT32: 33927
R_X86_64_REX_GOTPCRELX: 1
```

I measured the following:

* 66.54% Total Merge/finalize input sections
* 15.96% Total Write output file
* 15.86% Total Write sections
* 5.04% Total Parse input files
* 4.28% Total Split sections
* 3.07% Total Scan relocations
* 1.93% Total Add local symbols
* 0.81% Total Finalize synthetic sections
* 0.69% Total Assign sections
* 0.43% Total Finalize `.eh_frame`
* 0.22% Total Finalize address dependent content
* 0.22% Total Add symbols to symtabs

Next, I will describe performance significant options and rudimentary performance analysis.

## Performance significant options

Different programs have different link characteristics.
For the same program, we can get vastly different results with different compiler and linker options.
The percentage numbers in this article are rough, just indicating the few applications that I tested. YMMV.

The most significant compiler options for link time are:

* `-O`: the input size matters
* `-g`: leads to large portion of time spent on writing `.debug_*` sections, diluting the significance of the performance of other passess
* `-gz` (compile action): zlib-compressed debug sections require uncompression on the linker side
* `-gz` (link action): passes `--compress-debug-sections=zlib` to the linker
* `-ffunction-sections`: often used with `-fdata-sections`. `-fno-unique-section-names` may help.

The most significant linker options for link time are:

* `--compress-debug-sections=zlib`: Compress output `.debug_*` sections with zlib
* (ld.lld specific) `-O0` (`-Wl,-O0` in compiler drivers) disables `SHF_MERGE` duplicate elimination. This can tremendously decrease link time for a program with large `.debug_str`, but you will get a huge `.debug_str` as the penalty.
* `-z nocombreloc`: Don't sort dynamic relocations in `.rela.dyn`. This is a large bottleneck when you have millions of relocations in a mostly statically linked executable. Well, this is the cost of PIE.
* `--icf={safe,all}`: The section-based deduplication requires extensive computation on contents and relocations.
* `--gc-sections`: Discard unreferenced input sections
* `--build-id`: Compute a message digest from the file content

Now let me make an in-depth analysis of various passes in ld.lld.

## Find and load input files

For each input file, ld.lld calls `open` and `mmap`, then checks its file type by scanning the initial bytes (magic): relocatable object file, shared object, archive, lazy object file, LLVM bitcode file, or a linker script.
The process is sequential and synchronous. In the absence of debug information, this pass may take 5% time.

lld-link does asynchronous file load which can speed up a bit.
Parallelizing this pass can improve the performance a little, but it may cost the readability a bit.

## Parse input and perform symbol resolution

### Relocatable object files

A relocatable object file contributes symbols, sections, and relocations to the link process.
ld.lld reads sections and makes in-memory representations of sections, then reads the symbol table.
In a symbol table, all symbols with `STB_LOCAL` binding precede the weak and global symbols.
The local part is file-local while the non-local part requires resolution with other files.

The non-local part consists of `STB_GLOBAL`, `STB_WEAK`, and (GCC specific) `STB_GNU_UNIQUE` symbols.
An entry may be defined (`Defined`) or undefined (`Undefined`).
ld.lld inserts each entry to the global symbol table and resolves the entry.

```cpp
template <class ELFT> void ObjFile<ELFT>::initializeSymbols() {
  ArrayRef<Elf_Sym> eSyms = this->getELFSyms<ELFT>();
  // firstGlobal == sh_info of SHT_SYMTAB, the number of local symbols.
  for (size_t i = 0; i != firstGlobal; ++i) {
    const Elf_Sym &eSym = eSyms[i];
    InputSectionBase *sec = this->sections[secIdx];
    if (eSym.st_shndx == SHN_UNDEF || sec == &InputSection::discarded)
      new (this->symbols[i]) Undefined(...);
    else
      new (this->symbols[i]) Defined(...);
  }

  for (size_t i = firstGlobal, end = eSyms.size(); i != end; ++i)
    if (!this->symbols[i])
      this->symbols[i] = symtab->insert(eSyms[i].getName());

  for (size_t i = firstGlobal, end = eSyms.size(); i != end; ++i) {
    const Elf_Sym &eSym = eSyms[i];
    if (eSym.st_shndx == SHN_UNDEF) {
      undefines.push_back(i);
      continue;
    }
    Resolve this->symbols[i] with eSyms[i];
  }
  // Introduced by https://reviews.llvm.org/D95985 to stabilize symbol resolution behaviors.
  for (unsigned i : undefineds) {
    const Elf_Sym &eSym = eSyms[i];
    this->symbols[i]->resolve(Undefined(...));
  }
}
```

### Shared objects

A shared object contributes just symbols to the link process.
For a shared object, ld.lld reads its dynamic symbol table (`SHT_DYNSYM`) which consists of undefined symbols (represented identically to an undefined in a relocatable object file) and defined symbols (represented as a `Shared`).

```cpp
template <class ELFT> void SharedFile::parse() {
  ArrayRef<Elf_Sym> syms = this->getGlobalELFSyms<ELFT>();
  for (size_t i = 0, e = syms.size(); i != e; ++i) {
    const Elf_Sym &sym = syms[i];
    StringRef name = sym.getName(this->stringTable);
    uint16_t idx = versyms[i] & ~VERSYM_HIDDEN; // SHT_GNU_versym index
    if (sym.isUndefined()) {
      if (idx != VER_NDX_LOCAL && idx != VER_NDX_GLOBAL)
        name = saver.save((name + "@" + verneedName).toStringRef();
      Symbol *s = symtab->insert(name);
      s->resolve(Undefined(this, name, ...));
      continue;
    }
    if (!(versyms[i] & VERSYM_HIDDEN)) {
      // Handle the unversioned definition.
      Symbol *s = symtab->insert(name);
      s->resolve(SharedSymbol(*this, name, ...));
    }
    if (idx == VER_NDX_GLOBAL)
      continue;
    // Handle the versioned definition.
    if (idx >= verdefs.size() || idx == VER_NDX_LOCAL)
      error(...);
    else {
      name = saver.save((name + "@" + verdefName).toStringRef());
      Symbol *s = symtab->insert(name);
      s->resolve(SharedSymbol(*this, name, ...));
    }
  }
}
```

### Archives

This is an ancient trick to decrease linker work: initially every archive member is in a lazy state, if an archive member provides a definition resolving an undefined symbol, it will be extracted.
Extraction may trigger recursive extraction within (default) the archive or (`--start-group`) other archives.

### Symbol resolution

TODO: move some part to "how to implement an ELF linker"

The resolution between two .o defined/undefined symbols is associative.
In the absence of weak symbols and duplicate definition errors, the interaction is also commutative.

The resolution between a .o and a .so is commutative and associative.

The resolution between two .so is associative.
If two .so don't define the same symbol, the resolution is also commutative.

For an archive symbol, its interaction with any other file kind is neither commutative nor associative.

When linking a large program with little debug information, `ld.lld --time-trace` shows some symbol resolution passes which take varying time from 0.x~3%.
Anyhow, an empty global symbol table iteration takes 0.x% time.
In the presence of `--dynamic-list` and `--version-script`, the upper bound can go higher because wildcard pattern matching can be slow.
Some passes are related to ideal symbol resolution semantics which are not needed by 99.9% software in the wild: demote shared symbols/archives, parse symbol versions, redirect symbols (for edge-case `--wrap` and unversioned symbol elimination).

I believe mold handles fewer cases and will take similar performance hits if the tricky cases are handled.

### What can be parallelized

"Parse input and perform symbol resolution" is typically the largest bottleneck in a link and can be parallelized in three places.

* initialization of sections (embarrassingly parallel)
* COMDAT group resolution
* initialization of local symbols (embarrassingly parallel)
* initialization of non-local symbols
* symbol resolution

#### Parallel initialization of sections

In my estimate this takes 40% time of "Parse input files".

ld.lld does not parse sections from archive members and lazy object files.
Eagerly parsing sections will make the linker more prepared for parallelism with a hit to the single-threading performance.
If many archive members end up to be extracted, this may end up to be a good trade-off for multi-threading.
For the `chrome` executable in Chrome, the utilitization ratio is very high (79%).

Tips: `ld.lld --print-archive-stats=-` writes the utilitization ratio of archives to `-` (stdout).

#### Parallel COMDAT group resolution

See [COMDAT and section group](https://maskray.me/blog/2021-07-25-comdat-and-section-group).

#### Parallel initialization of local symbols

Initialization of local symbols is embarrassingly parallel.
As of 13.0.0, ld.lld called `make<Defined>` or `make<Undefined>` for each symbol entry where `make` is backed up by the non-thread-safe `BumpPtrAllocator`.
I recently changed the code to batch allocate `SymbolUnion`.
Is this part ready to be parallelized? Not yet.

To support linker scripts, a defined symbol may be relative to either an output section or an input section.
The symbol representation has `struct Defined : Symbol { ... SectionBase *section; }`, which means we need to have the section representation first.
To diagnose relocations referencing a local symbol defined in a discarded section (due to COMDAT rules), a local symbol may be represented as `Undefined`.
This needs the section pointer and COMDAT group resolution to decide whether a local symbol should be `Defined` or `Undefined`.

For relocatable object files, ld.lld initialize sections before symbols.
To know the sections we need to eagerly parse archives and lazy object files.

If we do parallel initialization, the `section` member may need to be fixed after COMDAT group resolution.
The symbol kind may change from `Defined` to `Undefined` as well.
Note that the symbol kind may not need to be explicit through C++ class inheritance, though ld.lld currently does this and is different to change at this point due to the numerous references.

ld.lld may need an option to choose whether to eagerly parse archive and lazy object files.

#### Parallel initialization of non-local symbols

If both `a.o` and `b.o` have a non-local symbol `foo`, they need to refer to the same entry in the global symbol table.
Conceptually an object file needs a data structure like `vector<Symbol *> symbols;`.
In the database world, this is a hash join or sort merge join problem.
Every linker implementation uses a hash join.

ld.lld represents the global symbol table with `llvm::DenseMap<llvm::CachedHashStringRef, int> symMap; std::vector<Symbol *> symVector;`.
For each symbol table entry, ld.lld reserves a symbol in the global symbol table if it does not exist yet.
If we had a concurrent hash table, the insertion could obviously be parallelized.
In April 2021, there was a llvm-dev thread "Concurrent Hashmap?" but we haven't taken any action.

One trivial concurrent hash table which may be suitable for lld is shards plus spin locks.

According to my estimate, `symtab->insert` takes 35% time of "Parse input files".

mold uses a concurrent hash table from Intel oneTBB.

#### Parallel symbol resolution

I previously mentioned that the resolution of archive symbols is not associative, therefore parallelism is difficult.
mold seems to be willing to sacrifice the compatibility a bit.

```asm
# RUN: rm -rf %t && split-file %s %t
# RUN: llvm-mc -filetype=obj -triple=x86_64 %t/a.s -o %t/a.o
# RUN: llvm-mc -filetype=obj -triple=x86_64 %t/b.s -o %t/b.o
# RUN: llvm-mc -filetype=obj -triple=x86_64 %t/c.s -o %t/c.o
# RUN: llvm-ar rc %t/b.a %t/b.o
# RUN: ld.lld -shared -soname=b %t/b.o -o %t/b.so
# RUN: ld.lld -shared -soname=b %t/c.o -o %t/c.so
# RUN: ld.lld %t/a.o %t/c.so %t/b.a -o %t/out

#--- a.s
.globl _start
_start:

#--- b.s
.globl foo
foo:

#--- c.s
call foo
```

In GNU ld, gold, and ld.lld, the undefined symbol in `c.so` causes `b.a` to be extracted.
In mold, `b.a` is not extracted.
If `b.a` has a strong definition which can override a weak definition in `a.o`, mold'semantics will not make that happen.
The semantics also suppresses a possible duplicate definition error.

I know 99.99% software don't leverage this point, but personally I think the 0.01% case is important and indicates a symbol resolution ideal I want to keep up with.

mold assigns lower priorities to archive symbols than shared symbols.
The `b.so` definition wins, regardless of the position of `a.a`.

According to mold `--perf`, the symbol initialization and resolution performance is like the following:

* 1 thread: 1.3x slower than ld.lld (locks, atomics, and split passes have costs)
* 2 threads: 1.75x faster than 1 thread
* 4 threads: 3x faster than 1 thread
* 8 threads: 5.3x faster than 1 thread
* 16 threads: 5.4x faster than 1 thread

## Relocation scanning

ld.lld's relocation scanning is noteworthily slower than mold's. I think there are two reasons:

* Generic relocation type abstraction (`RelExpr`) and virtual function calls have overhead.
* `InputSectionBase::relocations` is present. For an `InputSection` with N relocations, the field needs to allocate N entries.
* (minor) Mips hacks and more sophisticated error checking add some small overhead.

`InputSectionBase::relocations` seems wasteful at the first glance, but it provides some flexibility.
Relocation computation can be centralized, preventing architecture-specific errors.
Range extension thunks can be generalized a bit and avoid re-analyzing the relocation type.
In the future, when we introduce a more compact format for static relocations (compacter than REL), we can keep the information extraction in one place instead of scattering it all over the code base.

Updated in 2022: `sec->relocations.reserve(rels.size());` in a function called by `elf::scanRelocations` is now parallel.
However, the simple malloc-once-in-a-parallel-task pattern poses challenges to many malloc implementations.
On the machine I tested, with more than 8 threads, glibc malloc is very slow. Even with some fast malloc implementations, more-than-16 threads may show slowdown.
See my testing with many malloc implementations: <https://gist.github.com/MaskRay/219effe23a767b85059097f863ebc085>

## Global transforms

### `--gc-sections`

See [Linker garbage collection](https://maskray.me/blog/2021-02-28-linker-garbage-collection).
For a `-ffunction-sections -fdata-sections` build, GC can take little time (say, 2%) if there is debug information or a lot (10%) when there is little debug information.
`.debug_*` sections are large and monolithic (fewer than `.text.*` and `.data.*`) and take significant write time, so they dilute the proportion of GC time.

The code is straightforward to parallelize, unfortunately llvm-project does not provide a parallel producer–consumer framework like oneTBB `[algorithms.parallel_for_each.feeder]`.
Once the library is available, it is not difficult to parallelize this part.

### `--icf`

If enabled, the pass may take 3~15% time.
It is more expensive than the current sequential `--gc-sections`.

That said, ld.lld's algorithm is pretty good.
It computes a lightweight message digest for each section, then collects hashes into buckets.
For each bucket, it does pairwise comparison which is easy to reason about and expensive computation can be lazy.

Another idea is to use message digests to fully describe a section (mold).
The message digest can be more expensive to compute as the pairwise comparison approach may skip many condition branches.

My test shows that mold's algorithm is sometimes slower, for example when linking Clang.

ICF costs ld.lld `SectionBase::repl` and indirection costs in several places.
There is complexity in `.eh_frame` deduplication (`EhFrameSection::isFdeLive`) and dead reloc resolution for debug informtion.

## Scan relocations

An input section may have a dependent `SHT_REL`/`SHT_RELA`.
ld.lld scans relocation records to decide what actions are taken:

* link-time constants: the location should be updated in-place
* GOT: `.got` and `.rela.dyn` need an update
* PLT: `.plt`, `.got.plt`, and `.rela.plt` need an update
* [copy relocations, canonical PLT entries](https://maskray.me/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected#summary)
* TLS dynamic relocations
* report a diagnostic

For single-threading, the speed of mold's relocation scan is 1.7~2.3x of ld.lld's.
I think fundamentally it is because ld.lld does more work.
It has more dispatches and abstraction costs (e.g. virtual function), and provides more diagnostics and handles many tricky cases that mold doesn't do:

* support all of REL/RELA/mips64el for one architecture (even if the architecture typically doesn't use REL)
* use a generic relocation expression (RelExpr) design which costs two virtual functions processing one relocation
* handle non-preemptible [GNU indirect function](https://maskray.me/blog/2021-01-18-gnu-indirect-function)
* retain otherwise unused `.got` and `.got.plt` for certain relocation types
* Hexagon/Mips/PowerPC hacks
* Unfortunate Fortran COMMON symbol semantics (only needed by PowerPC64 AFAIK)

Every check has a small cost. Many a little makes a mickle.

I have a patch refactoring the way ld.lld does relocations but it may increase lines of code a lot: <https://reviews.llvm.org/D113228>.
I do not know whether it will be a right trade-off.

Another thing worth pointing out is that this process passes some relocations (`InputSectionBase::relocations`) which are needed by subsequent passes like range extension thunks.
Many input relocations are currently pass-through, so we pay `push_back` time and memory.
If we speculately think most relocations will be pass-through we can omit `InputSectionBase::relocations`, but subsequent passes will take input which is more difficult to deal with.

Except for the constraints mentioned above, relocation scan is conceptually straightforward to parallelize.
Local symbols do not need communication while non-local symbols just need atomic states.

My <https://reviews.llvm.org/D114783> is an important refactoring in ld.lld's relocation scan.
To enable parallelism, there is still much to do.
We would lightly have to exclude some architectures needing special treatment Mips/PowerPC.
With other constraints it may or may not be a good trade-off.

One way to reduce dispatch and virtual function costs is to change `template <class ELFT, class RelTy> void scanRelocs(...)` to `template <class Arch, class RelTy> void scanRelocs(...)`.
For one architecture, dead code elimination will help.
This approach will however lead to some code bloat.

### mold

In a Clang build, I measured the following for mold:

* 1 thread: 2.3x faster than ld.lld
* 2 threads: 1.88x faster than 1 thread
* 4 threads: 3.28x faster than 1 thread
* 16 threads: 11.13x faster than 1 thread

On the other hand, with little debug information, relocation scan takes 15% total time.
IMO parallelism does not pay off if we don't optimize other critical paths to stand out the 15% time.
I raised this point during the 2021 LLVM Dev Meeting round table "LLD for Mach-O".

## Map input sections and synthetic sections into output sections

To support linker scripts, ld.lld's model was designed for output section descriptions and input section descriptions.
A typical scenario where a user complains about link performance does not involve a linker script.
Nevertheless, the mapping process takes a slight performance hit as we reuse the framework in the absence of a `SECTIONS` command.
For a large executable with little debug information "Assign sections" took 2% time.

If we want to optimize this case, we will need to introduce a separate code path which will lead to implementation complexity.

## `SHF_MERGE` duplicate elimination

A section with the `SHF_MERGE|STRINGS` flags has data elements consisting of NUL-terminated strings.
In a program with debug information, `.debug_str` (with the `SHF_MERGE|STRINGS` flags) may be the hottest pass.
A section with just the `SHF_MERGE` flag has data elements of a uniform size.
In either case, duplicate elements can be eliminated as an optional optimization.
ld.lld performs the optimization with (the default) `-O1` and above.

ld.lld splits the section content into pieces, collects pieces from all sections, then eliminates duplicates with a hash table.
An obvious idea is to use a concurrent hash table. As said previously, llvm-project does not provide one.
ld.lld duplicates work and does the following instead:

```cpp
size_t concurrency = PowerOf2Floor(numThreads, 32);
for each thread {
  for (MergeInputSection *sec : sections)
    for (auto &piece : sec->pieces)
      if (piece.live && (piece.shardId & (concurrency-1)) == threadId)
        shards[piece.shardId].add(piece.data);
}

// Compute an in-section offset for each shard.
size_t off = 0;
for (size_t i = 0; i < numShards; ++i) {
  shards[i].finalizeInOrder();
  if (shards[i].getSize() > 0)
    off = alignTo(off, alignment);
  shardOffsets[i] = off;
  off += shards[i].getSize();
}
size = off;

parallelForEach(sections, [&](MergeInputSection *sec) {
  for (auto &piece : sec->pieces)
    if (piece.live)
      piece.outputOff += shardOffsets[getShardId(piece.hash)];
});
```

It would be nice if we could explore alternative algorithms.

## Open output file

For `-o out`, ld.lld unlinks `out` asynchronously, creates `out` in read-write mode, and resizes `out` to the output size.
The output size is computed in the writer and has to wait after output section addresses are assigned, so it is difficult to overlap this time consuming pass with other passes.

I recently noticed that the way LLVM's Support library resizes the file is problematic for Linux tmpfs.
The API calls `posix_fallocate` which has a nice property of reporting `ENOSPC` for some filesystems in some operating systems but may be bad for tmpfs. See <https://reviews.llvm.org/D115957>.
In my system, `posix_fallocate` on an 1.4GiB output costs 0.3s, which is significant.

An alternative design is to overwrite the output. This would lead to old/new file inconsistency problems.
If the old file has hard links, this would corrupt the other links.
Also, you have to be carefuly to zero all bytes, otherwise you may get non-reproducible output.

## Write sections

Some output sections contain one single synthetic section.
I plan to analyze some expensive sections more carefully.
For my experiments this week I use the additional time tracing (<https://reviews.llvm.org/D115984>) a lot.

I found that `.rela.dyn` sorting (`-z combreloc`) took 10+% time for a position-independent executable with millions of relocations.
I make it parallel and optimized it further in <https://reviews.llvm.org/D115993>.
This is a great example demonstrating that generality and feature creep bring us constraints and abstraction costs:

* `outputSec` is in the dynamic relocation representation because of Mips.
* We want to support all of REL/RELA/mips64el. We value code brevity, so manual loop unswitching would not be OK.
* We support ld.lld's partition feature, so obtaining the symbol index has an extra condition which adds a little cost.
* We support Android packed relocations which have very different encoding schemes, so refactoring `encodeDynamicReloc` needs to touch it.

ld.lld iterates over output sections and parallelizes input sections for each output section.
This strategy works well when an output section has many input sections.
However, for output sections created by linker (`.rela.dyn`, `.eh_frame`, `.symtab`, `.strtab`, etc), an output section typically has only one input section.
Many synthetic sections are not parallelized.

```cpp
template <class ELFT> void Writer<ELFT>::writeSections() {
  ...
  // In -r or --emit-relocs mode, write the relocation sections first as in
  // ELf_Rel targets we might find out that we need to modify the relocated
  // section while doing it.
  for (OutputSection *sec : outputSections)
    if (sec->type == SHT_REL || sec->type == SHT_RELA)
      sec->writeTo<ELFT>(Out::bufferStart + sec->offset);

  for (OutputSection *sec : outputSections)
    if (sec->type != SHT_REL && sec->type != SHT_RELA)
      sec->writeTo<ELFT>(Out::bufferStart + sec->offset);
  ...
}

template <class ELFT> void OutputSection::writeTo(uint8_t *buf) {
  ...
  // sections.size() is typically one for .rela.dyn/.eh_frame/.symtab while many for .text/.data/.rodata.
  parallelForEachN(0, sections.size(), [&](size_t i) {
    ...
  }
}
```

Unfortunately `llvm::parallelForEachN` does not support nesting (when nested, only the outmost loop is parallel).
Intel oneTBB's ABIs support nesting.

### `--compress-debug-sections=zlib`

In one invocation linking a Clang executable (`-DCMAKE_BUILD_TYPE=RelWithDebInfo`), ld.lld spent 36.7s compressing `.debug_*` sections while the total link time was 41.5s.

There are several problems.

* No format except zlib is standard. This is a big ecosystem issue. ld.lld is part of the ecosystem and can drive it, but with significant buy-in from debug information consumers. As a generic ELF feature, a new format needs a generic-abi discussion.
* llvm-project offloads to the system zlib. It does not leverage parallelism.
* Technically ld.lld can compress different `.debug_*` output sections but the degree of parallelism would be slow and there would be implementation complexity.

See [Compressed debug sections](/blog/2022-01-23-compressed-debug-sections) that this has been parallelized in 14.0.0.

## `--build-id`

According to <https://fedoraproject.org/w/index.php?title=RolandMcGrath/BuildID&oldid=16098>, the feature was added as approximation of true uniqueness across all binaries that might be used by overlapping sets of people".

In ld.lld, `--build-id` is implemented as a tree hash, which is pretty fast.
The performance is limited by the library code checked into llvm-project.
Some library code is not optimal (see <https://reviews.llvm.org/D69295> for SHA-1).

Note: ld.lld's `--build-id` feature, as I know, has no validation tool.

## malloc implementation

mold uses mimalloc by default. The llvm-project build system just uses the system malloc, i.e. glibc on a typical Linux distribution.

If I link lld against a mimalloc static library with `-DCMAKE_EXE_LINKER_FLAGS=-Wl,--push-state,/tmp/p/mimalloc/out/release/libmimalloc.a,--pop-state`, ld.lld linking Chrome and Clang is 1.12x as fast.

## Executable size

The excutable size costs a little of performance.
This matters very little but I still want to mention it.

The lld codebase has `make<XXX>` for types which need a dedicated `BumpPtrAllocator`.
Many singleton objects unfortunately use `make<XXX>` as well. The return value references the first element in a 4096-byte slab.
```cpp
Configuration *elf::config;

bool elf::link(...) {
  errorHandler().cleanupCallback = []() {
    freeArena(); // destroy `make<XXX>` allocated objects
  };

  config = make<Configuration>();
}
```
Every `XXX` pulls in lots of template code from `BumpPtrAllocator` and makes the x86-64 executable more than 1KiB larger.

* If we change the global variable to `std::unique_ptr<XXX>`, there will be a non-trivial destructor at exit time. Fortunately lld exits via `_exit`...
* If we manually destroy global variables, it would be bad ergonomics.
* Global states are difficult to remove from ld.lld.

If your lld executable is a position-independent executable or is linked against `libLLVM.so` or `libLLD*.so`, you pay some relocation processing costs in ld.so.
For an `lld` executable, the `-pie` one spends extra 8ms to resolve the 34000+ more relocations (mostly are `R_X86_64_RELATIVE`).

lld is large because LTO pulls in a substantial portion of LLVM libraries, including middle-end optimizations and code generation.
I did an experiment (<https://github.com/MaskRay/picolld>) that if one removed LTO and non-ELF ports, the ELF lld executable was just ~3MiB on Linux x86-64.
This is competitive as the universal-targeting (supporting multiple architectures in one executable) gold is more than 2MiB.

## `-Map`

The feature is for debugging and binary analysis.
The link map output is tremendous so therefore inherently slow.
ld.lld parallelizes the computation of some symbol properties.

ld64.lld adopts an approach that makes the link map computation asynchronous, interleaved with other parallelizable passes.

## Cost of new features

Most links don't have `--strip-debug` or `--strip-all`.
However, "Split sections" is used by partitions, so we cannot simply drop the code path for the common case.
"Split sections" may cost 1~2% link time.

## New hope?

With multi-threading, why is mold faster than ld.lld? Ordered by my rough estimate of importance:

* `SHF_MERGE` duplicate elimination
* eager loading of archive members and lazy object files
* parallel section/symbol table initialization
* parallel relocation scan
* faster synthetic sections
* parallel --gc-sections
* parallel symbol resolution

I think ld.lld should explore parallel symbol table initialization and (with an option) eager loading of archive members and lazy object files.
Because of some constraints I mentioned, the degree of parallelism is limited and the speed-up may never be as amazing as mold.

The lack of concurrent LLVM data structures and parallel facility reduce some parallelism design space we can make.
We should solve this problem.

See [lld 14 ELF changes](/blog/2022-02-20-lld-14-elf-changes) and [lld 15 ELF changes](/blog/2022-09-05-lld-15-elf-changes) that the two releases have improved performance a lot.
It requires significant changes to various passes, but I will work on the performance improvement from time to time.

## Epilogue

mold brings more parallelizable passes to the table. This is great.

Features and generality require abstraction. Abstraction has costs.
Costs can be tangible (performance) and intangible (reduced design space).
For symbol table initialization, symbol resolution, relocation scan, I believe ld.lld has found a good balancing among different trade-offs.
These passes have been ruthlessly challenged by mold.
However, with various constraints they may be difficult to change and there is an upper bound on what we can improve.
Perhaps 99% people will not feel sympathy for that:(
When archive semantics and COMDAT were designed, did the designers anticipated that after some decades there are linker developers struggling with finding a good balance? :)
