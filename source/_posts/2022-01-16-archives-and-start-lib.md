layout: post
title: Archives and --start-lib
author: MaskRay
tags: [binutils,linker,llvm]
---

## .a archives

Unix-like systems represent static libraries as `.a` archives.
A `.a` archive consists of a header and a collection of files with metadata.
Its usage is tightly coupled with the linker.
An archive almost always contains only relocatable object files and the linker has built-in support for reading it.

```sh
% as /dev/null -o a.o
% rm -f b.a && ar rc b.a a.o
% ar t b.a
a.o
```

One may add other types of files to `.a` but that is almost assuredly a bad thing.

```sh
% rm -f a.a && ar rc a.a a.o b.a  # archive in archive, bad
% ar t a.a
a.o
b.a
% echo hello > a.txt
% rm -f a.a && ar rc a.a a.o a.txt  # text file in archive, bad
```

The original linker designers noticed that for many programs not every member was needed, so they tried to allow the linker to skip unused members.
Therefore, they invented the interesting but confusing archive member extraction rule.
See [Symbol processing#Archive processing](/blog/2021-06-20-symbol-processing#archive-processing) for details.

<!-- more -->

To allow skipping members, they added a symbol table (also called index) to the archive.
The symbol table is a list of (filename, member) pairs for defined non-local symbols.
It is represented as a special member of the archive. Its name varies on different systems (no name, `/SYM64`, `__.SYMDEF`, `__.SYMDEF_64`, etc).

```sh
% echo '.globl foo; foo: .data; .dc.a bar' | as - -o a.o
% echo '.globl bar; bar:' | as - -o b.o
% rm -f a.a && ar rc a.a a.o b.o
% nm --print-armap a.a

Archive index:
foo in a.o
bar in b.o

a.o:
                 U bar
0000000000000000 T foo

b.o:
0000000000000000 T bar
```

Quote FreeBSD `ar(5)`:

> In the SVR4/GNU archive format, the archive symbol table starts with a 4-byte binary value containing the number of entries contained in the archive symbol table.  This count of entries is stored most significant byte first.
> Next, there are count 4-byte numbers, each stored most significant byte first.  Each number is a binary offset to the archive header for the member in the archive file for the corresponding symbol table entry.
> After the binary offset values, there are count NUL-terminated strings in sequence, holding the symbol names for the corresponding symbol table entries.

You may read [64-bit Archives Needed](http://www.linker-aliens.org/blogs/ali/entry/64_bit_archives_needed/) for more information.

## Thin archives

Google investigated techniques to improve build speed.
In 2008-03, Cary Coutant (then at Google) posted [PATCH: Add support for "thin" archives](https://sourceware.org/pipermail/binutils/2008-March/055648.html) on the binutils mailing list.
Here is the key insight:

> At other times, I've received requests from customers who are using archive libraries as intermediate collections of .o files during a build of a large project. In this usage model, the archives aren't intended for distribution outside that project; they're just used during the build. In these situations, the .o files will remain where they are in the build directories, and the copying of the files into the archive libraries is a waste of time and space -- useful only because the archive library serves as a useful collection mechanism to simplify the later link command.

A thin archive does not copy the member contents. It just stores the filename (which is typically relative).
While a regular archive is self-contained, a thin archive needs to reference files.
This is perfectly fine for throw-away archives created as intermediate build artifacts.
If one distributes static libraries, a thich archive should be the choice as a thin archive needs the referenced files, which is cumbersome.

A thin archive has the nice property that it is transparent.
When the referenced files exist, a thin archive is indistinguishable from a regular archive from the view of some tools (ld, make).
Some, however, may consider this a disadvantage as they need to run some command (e.g. `file`) to know whether an archive is trick or thin, when distributing a static library.
I think the transparent design philosophy might be the reason a thin archive reuses the `.a` suffix name.

In the thin archive patch for GNU ar, the modifier `T` is picked to create a thin archive.
I got to know the portability issue when emaste from FreeBSD created [[META] Make llvm-ar a drop-in replacement for BSD ar](https://github.com/llvm/llvm-project/issues/25899) and noted that in FreeBSD `/usr/bin/ar`, `T` means `Use only the first fifteen characters of the archive member name or command line file name argument when naming archive members.`
So I researched a bit. X/Open System Interface (XSI) says

> Allow filename truncation of extracted files whose archive names are longer than the file system can support. By default, extracting a file with a name that is too long shall be an error; a diagnostic message shall be written and the file shall not be extracted.

The text dated at least as early as 2004. Many implementations including elfutils ar (2007-02), FreeBSD ar, and macOS ar have adopted this interpretation.
This modifier `T` is, however, quite obscure.

Nevertheless, I think for binutils ar, adding `--thin` and educating projects to use the long option will be best for portability.
binutils 2.38 will include my submitted patch. In the NEWS I call `T` deprecated without diagnostics.
It is unclear when a future binutils ar can issue diagnostic.
The Linux kernel has used the thin archive `-T` semantics since 2016-09: [kbuild: allow architectures to use thin archives instead of ld -r](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5967db9af51a84f5e181600954714a9e4c69f1f).

The 14.0.0 release includes my patch [[llvm-ar] Add --thin for creating a thin archive](https://reviews.llvm.org/D116979).

Note: for linking, a regular archive may be faster than a thin archive due to better disk locality, but likely only a little.
The linker being asynchronous or parallel on reading input files may decrease the performance gap.
It can also be argued that a regular archive may be slower because a on-disk object file and its copy in the archive costs 2x buffer cache.

### Thin archives without a symbol table

A thin archive has a very compact representation: a header, minimum metadata (mainly the filename) for each member, and a symbol table.
It may be attempted to remove the symbol table to decrease the size further.
When creating an archive, ar/llvm-ar support the modifier `S` to skip the symbol table.
However, GNU ld does not accept an archive without a symbol table (index):
```text
% ld.bfd a.o a.a
ld.bfd: a.a: error adding symbols: archive has no index; run ranlib to add one
```

ld.lld used to have a similar diagnostic `archive has no index; run ranlib to add one` unless all members were LLVM bitcode files.
lld 14.0.0 will include my change ([D117284](https://reviews.llvm.org/D117284)) which parse an archive without a symbol table using the `--start-lib` code path (see below).

Normally ld.lld tends to be strict and catch user errors. We need to think of ecosystem influence when dropping a diagnostic.
Here, the influence is that users may get addicted to the behavior and build archives not working with GNU ld and gold.
I think this addiction is acceptable, since the repair is straightforward (running ranlib on the archive).

## `--start-lib`

In 2010-03, Rafael Ávila de Espíndola posted [[gold][patch] Add the --start-lib end --end-lib options](https://sourceware.org/pipermail/binutils/2010-March/066146.html).
The insight was to skip archive creation.

If `a.a` contains `b.o c.o`, `ld ... --start-lib b.o c.o --end-lib` behaves exactly the same way as `ld ... a.a`.

The apparent pros of `--start-lib` over thin archives are:

* The build system can skip building archives. All files must be available to build the archive symbol table (to satisfy GNU linkers). GNU ar has the requirement even if you enable `D` (deterministic mode) and thin archives. In a distributed build system, this adds a gather step before linking.
* We don't waste archiver's time on building the thin archive symbol table.
* No regular and thin archive confusion.

Cons:

* The linker command line built by the compiler driver is longer.

Now let's reflect on the archive symbol table.
Due to the `archive has no index; run ranlib to add one` GNU ld diagnostic, every archive needs a symbol table.
To build the symbol table, for a member of an ELF relocatable object file, ar needs to parse its symbol table and find the non-local definitions.
For a member of a GCC LTO object file or an LLVM bitcode file, ar needs a plugin to parse the symbol table.
A build system mistake is to mix LTO files and an ELF relocatable object file in one archive.
The archive still has a symbol table, but the definition list is incomplete, which may lead to weird "undefined reference" errors.

`--start-lib` and `--end-lib` allow us to stop wasting time/space for the archive symbol table and worrying about the LTO case.

I created a feature request for GNU ld in 2019: [ld: Support --start-lib --end-lib](https://sourceware.org/bugzilla/show_bug.cgi?id=24600).

### ld64.lld

I have submitted [D116913](https://reviews.llvm.org/D116913) adding `--start-lib` and `--end-lib`:
The feature request suggests that we can replace the pair of options with one single `-objlib`: `-objlib a.o -objlib b.o`.
I prefer `--start-lib` and `--end-lib` mainly because they convey more information.

* `--start-lib a.o --end-lib --start-lib b.o --end-lib` means that `a.o` and `b.o` belong to different virtual libraries.
* `--start-lib a.o b.o --end-lib` means that `a.o` and `b.o` belong to the same virtual library.

In ld.lld, this information has usage in `--warn-backrefs`, as described by [Dependency related linker options](2021-06-13-dependency-related-linker-options).
I do not find an immediate use case for ld64.lld but just think this flexibility will turn out to be useful (e.g. if we assign a virtual library a name).

### Symbol interning

In a traditional linker implementation, the linker interns symbol names in archive symbol tables and `.o` symbol tables.
For every definition in an extracted archive member, we intern it twice, once for the archive symbol table, once for the `.o` symbol table after extraction.
Unfortunately the interning result cannot be reused.

On the other hand, symbols in a `--start-lib` object file can be interned only once.
The object file is initially parsed in a lazy state, then possibly parsed again in a non-lazy state.
Since the linker iterates over the same symbol table, the interning result in the lazy state can be reused by the non-lazy state.

Now let's ask what if we parse an archive using the `--start-lib` code path.
First, we can completely ignore the archive symbol table. `--start-lib` allows us to intern every definition only once.

For an extracted archive member, this strategy is strictly superior. In the archive symbol, the memory access to symbols associated with the archive member is just wasted.

For an unextracted archive member, this strategy accesses individual ELF symbol tables instead of the aggregated archive symbol table.
The archive symbol table has a compact representation. Parsing an ELF symbol table needs to skip undefined symbols.
Therefore, the `--start-lib` code path may regress on the performance, though I doubt it is noticeable.

In the end this probably depends on the archive member extraction ratio.
As of 2022-02, when linking chrome in the default build, 79% archive members are extracted according to `--print-archive-stats=`.
It turns out that with `--threads=8`, using the `--start-lib` code path is 1.05x as fast.

With a lower ratio, I have measured 1.01x when linking Clang.
With my experience, for projects large enough that the performance matters, the archive utilitization ratio is typically high.
So I would say the archive symbol table is nearly useless.

ld.lld 15.0.0 will include my patch [[ELF] Parse archives as --start-lib object files](https://reviews.llvm.org/D119074) which handles archives using the `--start-lib` code path.

## Build system integration

`--start-lib` and `--end-lib` have poor support in build systems.
Bazel supports them under the feature `supports_start_end_lib`, but CMake doesn't support them.
On the other hand, thin archives without a symbol table are quite good alternatives.

In CMake, you can specify `-DCMAKE_CXX_ARCHIVE_CREATE='path/to/llvm-ar cr --thin <TARGET> <OBJECTS>' -DCMAKE_CXX_ARCHIVE_FINISH=:` with llvm-ar 14.0.0.

Alexander Shaposhnikov noted in <https://reviews.llvm.org/D116850>:

> P.S. for a clean run ninja lld using thin archives seems to reduce the size of the build directory from ~14GB to ~8GB

`-DCMAKE_CXX_ARCHIVE_FINISH=:` is to prevent ranlib from adding the symbol table.
While playing with this, I have found that llvm-ranlib may convert a thin archive to a regular one.
[D117443](https://reviews.llvm.org/D117443) will fix it.

Many ar implementations ensure the symbol table is up-to-date automatically, no need for a separate ranlib command.

If you do not care about GNU ld compatibility, you can use `-DCMAKE_CXX_ARCHIVE_CREATE='path/to/llvm-ar crS --thin <TARGET> <OBJECTS>' -DCMAKE_CXX_ARCHIVE_FINISH=:` with LLD 14 or above.

## Deterministic mode

When binutils is configured with the default `--disable-deterministic-archives`, the produced archive symbol table contains UIDs, GIDs, and timestamps.
The timestamps are obviously non-deterministic while UIDs and GIDs are sometimes undesired as well.

llvm-ar defaults to the deterministic mode so you do not need to specify the `D` modifier.
