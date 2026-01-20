---
layout: post
title: lld 14 ELF changes
author: MaskRay
tags: [linker,llvm]
---

llvm-project 14 will be released soon.
I added some lld/ELF notes to <https://github.com/llvm/llvm-project/blob/release/14.x/lld/docs/ReleaseNotes.rst>.
Here I will elaborate on some changes.

<!-- more -->

* `--export-dynamic-symbol-list` has been added.
  ([D107317](https://reviews.llvm.org/D107317))
  When I added `--export-dynamic-symbol` to GNU ld, H.J. Lu asked me to add this option.
  I asked myself whether this was necessary but then realized this may help deprecate `--dynamic-list` in the long term.
  [`--dynamic-list`](/blog/2020-11-15-explain-gnu-linker-options#bsymbolic-and---dynamic-list) is confusing. It has a different semantics for executables and shared objects. The symbolic intention for shared objects isn't clear.
* `--why-extract` has been added to query why archive members/lazy object files are extracted.
  ([D109572](https://reviews.llvm.org/D109572))
  This was a long missing feature from ld.lld `-Map`. I picked a separate option because I realized that this need is often orthogonal to input section to output section map.
* If `-Map` is specified, `--cref` will be printed to the specified file.
  ([D114663](https://reviews.llvm.org/D114663))
  A linker's stdout output is often interleaved with different information, so being able to redirect a piece of information to a file is useful.
  I think it would be nice if GNU ld had `--cref=<file>` and not reused `-Map`.
* `-z bti-report` and `-z cet-report` are now supported.
  ([D113901](https://reviews.llvm.org/D113901))
* `--lto-pgo-warn-mismatch` has been added.
  ([D104431](https://reviews.llvm.org/D104431))
* Archives without an index (symbol table) are now supported and work with `--warn-backrefs`. One may build such an archive with `llvm-ar rcS [--thin]` to save space.
  ([D117284](https://reviews.llvm.org/D117284))
  In 15.0.0, the archive symbol table will be [entirely ignored](https://reviews.llvm.org/D119074).
  [Archives and --start-lib](/blog/2022-01-16-archives-and-start-lib) has more context.
* No longer deduplicate local symbol names at the default optimization level of `-O1`.
  This results in a larger `.strtab` (usually less than 1%) but a faster link time.
  Use optimization level `-O2` to restore the deduplication.
  In 15.0.0, the `-O2` deduplication is dropped to help parallel `.symtab` write.
* In relocatable output, relocations to discarded symbols now use tombstone values.
  ([D116946](https://reviews.llvm.org/D116946))
* `--compress-debug-sections=zlib` is now run in parallel. `{clang,gcc} -gz` link
  actions are significantly faster.
  ([D117853](https://reviews.llvm.org/D117853))
  [Compressed debug sections#linkers](/blog/2022-01-23-compressed-debug-sections#linkers) has more context.
* "relocation out of range" diagnostics and a few uncommon diagnostics
  now report an object file location beside a source file location.
  ([D112518](https://reviews.llvm.org/D112518))
* The write of `.rela.dyn` and `SHF_MERGE|SHF_STRINGS` sections (e.g. `.debug_str`) is now run in parallel.

Linker script changes:

* Orphan section placement now picks a more suitable segment. Previously the
  algorithm might pick a read-only segment for a writable orphan section and make
  the segment writable.
  ([D111717](https://reviews.llvm.org/D111717))
* An empty output section moved by an `INSERT` comment now gets appropriate flags.
  ([D118529](https://reviews.llvm.org/D118529))
* Negation in a memory region attribute is now correctly handled.
  ([D113771](https://reviews.llvm.org/D113771))

Architecture specific changes:

* The AArch64 port now supports adrp+ldr and adrp+add optimizations. `--no-relax` can suppress the optimization.
  This is a time that ld.lld gets an optimization (for the linked output) earlier than GNU ld.
  ([D112063](https://reviews.llvm.org/D112063))
  ([D117614](https://reviews.llvm.org/D117614))
* The x86-32 port now supports TLSDESC (`-mtls-dialect=gnu2`).
  ([D112582](https://reviews.llvm.org/D112582))
* The x86-64 port now handles non-RAX/non-adjacent `R_X86_64_GOTPC32_TLSDESC`
  and `R_X86_64_TLSDESC_CALL` (`-mtls-dialect=gnu2`).
  ([D114416](https://reviews.llvm.org/D114416))
* The x86-32 and x86-64 ports now support mixed TLSDESC and TLS GD, i.e. mixing
  objects compiled with and without `-mtls-dialect=gnu2` referencing the same
  TLS variable is now supported.
  ([D114416](https://reviews.llvm.org/D114416))
* For x86-64, `--no-relax` now suppresses `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` GOT optimization
  ([D113615](https://reviews.llvm.org/D113615))
* `R_X86_64_PLTOFF64` is now supported.
  ([D112386](https://reviews.llvm.org/D112386))
* `R_AARCH64_NONE`, `R_PPC_NONE`, and `R_PPC64_NONE` in input REL
  relocation sections are now supported.

Breaking changes

* `e_entry` no longer falls back to the address of `.text` if the entry symbol does not exist.
  Instead, a value of 0 will be written.
  ([D110014](https://reviews.llvm.org/D110014))
* `--lto-pseudo-probe-for-profiling` has been removed. In LTO, the compiler
  enables this feature automatically.
  ([D110209](https://reviews.llvm.org/D110209))
* Use of `--[no-]define-common`, `-d`, `-dc`, and `-dp` will now get a
  warning. They will be removed or ignored in 15.0.0.
  (`llvm-project#53660 <https://github.com/llvm/llvm-project/issues/53660>`_)

## Speed

(Compared with glibc malloc, linking against libmimalloc.a is 1.12x as fast.)
I use a `-DCMAKE_BUILD_TYPE=Release -DCMAKE_EXE_LINKER_FLAGS=$HOME/Dev/mimalloc/out/release/libmimalloc.a -DLLVM_ENABLE_PROJECTS='clang;lld' -DLLVM_TARGETS_TO_BUILD=X86` `-fno-pic -no-pie` build.
The host compiler is a close-to-main clang.
Both input and output is in tmpfs.

I have made dozens of changes scattering across the lld/ELF codebase to improve performance, e.g.

* Some changes as mentioned in the release notes
* [[ELF] Parallelize MergeNoTailSection::writeTo](https://github.com/llvm/llvm-project/commit/3aae04c744b03eb3eec7376f9d34fa3e42f8d108) 2021-12-17
* [[ELF] Replace LazyObjFile with lazy ObjFile/BitcodeFile](https://github.com/llvm/llvm-project/commit/3a5fb57393c3bc77be9e7afc2ec9d4ec3c9bbf70) 2022-12-22
* [[ELF] Optimize replaceCommonSymbols](https://github.com/llvm/llvm-project/commit/40fae4d8fcbd6224f16fdf3ba84a0d89b713e7d5) 2021-12-24
* [[ELF] Optimize --wrap to only check non-local symbols](https://github.com/llvm/llvm-project/commit/e694180033d1e9d6e215bbc2f956092d30c1e3cd) 2021-12-24
* [[ELF] Avoid referencing SectionBase::repl after ICF](https://reviews.llvm.org/D116093)
* [[ELF] Optimize MergeInputSection::splitNonStrings with resize_for_overwrite. NFC](https://github.com/llvm/llvm-project/commit/bc1369fae35b7a6fc2cff5282bf24fe3b9a944ee)
* [[ELF] Remove unneeded SyntheticSection memset(*, 0, *)](https://github.com/llvm/llvm-project/commit/9c4292a59da2d8fc31ead9649868ec5bfb3cf883) 2022-01-16
* [[ELF] Move duplicate symbol check after input file parsing](https://github.com/llvm/llvm-project/commit/88d66f6ed1e5a3a9370a3181b7307fe65590e3ac) 2022-02-22. This is essential to unlock parallel initialization of sections and symbols

Linking a `-DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON` build of clang:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{13,14}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      1.133 s ±  0.007 s    [User: 1.277 s, System: 0.436 s]
  Range (min … max):    1.119 s …  1.142 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      1.003 s ±  0.012 s    [User: 1.286 s, System: 0.439 s]
  Range (min … max):    0.988 s …  1.025 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.13 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8'
```
0.04s of the saving is credited to the removal of `.strtab` deduplication.
(`--threads=2` => 1.16x, `--threads=2` => 1.17x)

Linking a `-DCMAKE_BUILD_TYPE=Debug` build of clang:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{13,14}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      5.237 s ±  0.032 s    [User: 8.976 s, System: 1.831 s]
  Range (min … max):    5.194 s …  5.288 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      4.480 s ±  0.024 s    [User: 8.674 s, System: 1.756 s]
  Range (min … max):    4.442 s …  4.522 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.17 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--threads=1` => 1.05x, `--threads=2` => 1.09x)

Linking a `-DCMAKE_BUILD_TYPE=RelWithDebInfo` build of clang:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{13,14}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      1.520 s ±  0.017 s    [User: 3.797 s, System: 1.210 s]
  Range (min … max):    1.479 s …  1.545 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      1.101 s ±  0.012 s    [User: 3.679 s, System: 1.244 s]
  Range (min … max):    1.084 s …  1.125 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.38 ± 0.02 times faster than 'numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--threads=2` => 1.16x)
0.13s of the saving is credited to [parallel write of `.debug_str`](https://github.com/llvm/llvm-project/commit/3aae04c744b03eb3eec7376f9d34fa3e42f8d108).
0.08s is credited to [not using `posix_fallocate`](https://reviews.llvm.org/D115957).
More is credited to optimized computation and sort of `.rela.dyn`.

Linking a default build of chrome:
```
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{13,14}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      8.017 s ±  0.042 s    [User: 7.440 s, System: 3.238 s]
  Range (min … max):    7.946 s …  8.089 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      6.857 s ±  0.052 s    [User: 6.921 s, System: 3.006 s]
  Range (min … max):    6.796 s …  6.982 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.17 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-13/out/release/bin/ld.lld @response.txt --threads=8'
```

## Memory usage

I have made some changes decreasing `sizeof(SymbolUnion)` and `sizeof(InputSection)`.
There is a 1~2% decrease for some programs with several malloc implementations.

ThinLTO application will see more reduction.
lld uses file-backed mmap to read input files. For ThinLTO indexing, the page buffers are nearly unused after symbol resolution.
I have changed lld to call `madvise(MADV_DONTNEED)` to overlap the page buffer memory with the memory allocated by LTO library (mostly ThinLTO import and export lists): <https://reviews.llvm.org/D116367>.
This change led to a 16% reduction when linking a large executable.

I have made another change that changed the `-–start-lib` code path to cache the symbol interning result, which led to 0.6% reduction: <https://reviews.llvm.org/D116390>.

## Executable size

I have audited commits by others. Almost all have nearly no size difference or slightly increase code size.
I have made some patches which improve flexibility and increase code size, but a dozen which decreases the code size.
In a `-DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++` builds,

```sh
% stat -c %s llvm-13/out/release/lib/../tools/lld/ELF/CMakeFiles/lldELF.dir/**/*.o | awk '{s+=$1}END{print s}'
6490096
% stat -c %s llvm-14/out/release/lib/../tools/lld/ELF/CMakeFiles/lldELF.dir/**/*.o | awk '{s+=$1}END{print s}'
5177744
```

## Thoughts

mold 1.1 was just released. People who wonder about lld's performance can check out [Why isn't ld.lld faster?](https://maskray.me/blog/2021-12-19-why-isnt-ld.lld-faster).

If one module consists of N features, the time complexity incrementally updating this module may be more than O(N), because the N features may interact and result in more than linear edges (though possibly still smaller than O(N^2)).
This rule applies to introducing multi-threading to ld.lld's symbol processing.
