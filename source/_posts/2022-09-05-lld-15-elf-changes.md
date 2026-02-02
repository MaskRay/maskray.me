---
layout: post
title: lld 15 ELF changes
author: MaskRay
tags: [linker,llvm,release]
---

llvm-project 15 was just released.
I added some lld/ELF notes to <https://github.com/llvm/llvm-project/blob/release/15.x/lld/docs/ReleaseNotes.rst>.
Here I will elaborate on some changes.

<!-- more -->

* `-z pack-relative-relocs` is now available to support `DT_RELR` for glibc 2.36+.
  ([D120701](https://reviews.llvm.org/D120701))
  This exciting size reduction optimization called `DT_RELR` finally landed in glibc.
  The linker option is all you need to use to enable this optimization.
  It took me a lot of energy to lobby for the feature in glibc. The benefit will pay off.
  See [Relative relocations and RELR](/blog/2021-10-31-relative-relocations-and-relr).
* `--package-metadata=` has been added to create package metadata notes
  [D131439](https://reviews.llvm.org/D131439)
  Fedora uses this to embed an allocable section in a linked image so that the core file will have the information.
  See [Package Metadata for Core Files](https://systemd.io/ELF_PACKAGE_METADATA/).
* `--no-fortran-common` (pre 12.0.0 behavior) is now the default.
  `--fortran-common` matches GNU ld's behavior but is more likely to cause problems when users mix COMMON and `STB_GLOBAL` symbols.
  See [All about COMMON symbols](https://maskray.me/blog/2022-02-06-all-about-common-symbols) for detail.
* `--load-pass-plugin` has been added to load a new pass manager plugin.
  ([D120490](https://reviews.llvm.org/D120490))
* `--android-memtag-{mode=,stack,heap}` have been added to synthesize `SHT_NOTE` for memory tags on Android.
  ([D119384](https://reviews.llvm.org/D119384))
* `FORCE_LLD_DIAGNOSTICS_CRASH` environment variable is now available to force LLD to crash.
  ([D128195](https://reviews.llvm.org/D128195))
  I am not so fond of adding such an envrionment variable, but I see that it can be useful for testing the crash reporting behavior. For example, Clang Driver may rerun a link action with `--reproduce=` to get a tarball.
* `--wrap` semantics have been refined.
  ([rG7288b85cc80f1ce5509aeea860e6b4232cd3ca01](https://reviews.llvm.org/rG7288b85cc80f1ce5509aeea860e6b4232cd3ca01))
  ([D118756](https://reviews.llvm.org/D118756))
  ([D124056](https://reviews.llvm.org/D124056))
  `--wrap` is quite complex. Previous releases have repeatedly tuned it. At this point I am mostly confident to say that the ld.lld behavior is desired.
  In the behaviors that ld.lld and GNU ld differ, I am confident to say that GNU ld's is not good enough:)
* `--build-id={md5,sha1}` are now implemented with truncated BLAKE3.
  ([D121531](https://reviews.llvm.org/D121531))
  The C implementation of BLAKE3 was imported into llvm-project.
  It is superior to MD5 and SHA1. We can drop the slow MD5 and SHA1 llvm-project implementations.
* `--emit-relocs`: `.rel[a].eh_frame` relocation offsets are now adjusted.
  ([D122459](https://reviews.llvm.org/D122459))
  The use case is rare. I recently learned that gold crashes on the use case <https://sourceware.org/bugzilla/show_bug.cgi?id=25968>.
* `--emit-relocs`: fixed missing `STT_SECTION` when the first input section is synthetic.
  ([D122463](https://reviews.llvm.org/D122463))
* `(TYPE=<value>)` can now be used in linker scripts.
  ([D118840](https://reviews.llvm.org/D118840))
  The syntax can be used to create an output section of the specified type.
  With data commands, we can create an output section even if there is no corresponding input section.
* Local symbol initialization is now performed in parallel.
  ([D119909](https://reviews.llvm.org/D119909))
  ([D120626](https://reviews.llvm.org/D120626))

Breaking changes

* Archives are now parsed as `--start-lib` object files. If a member is neither
  an ELF relocatable object file nor an LLVM bitcode file, ld.lld will give a warning.
  ([D119074](https://reviews.llvm.org/D119074))
  This change improved linking performance for chrome quite a bit.
  See [Archives and --start-lib](/blog/2022-01-16-archives-and-start-lib).
* The GNU ld incompatible `--no-define-common` has been removed.
* The obscure `-dc`/`-dp` options have been removed.
  ([D119108](https://reviews.llvm.org/D119108))
* `-d` is now ignored.
* If a prevailing COMDAT group defines STB_WEAK symbol, having a STB_GLOBAL symbol in a non-prevailing group is now rejected with a diagnostic.
  ([D120626](https://reviews.llvm.org/D120626))
* Support for the legacy `.zdebug` format has been removed. Run
  `objcopy --decompress-debug-sections` in case old object files use `.zdebug`.
  ([D126793](https://reviews.llvm.org/D126793))
* `--time-trace-file=<file>` has been removed.
  Use `--time-trace=<file>` instead.
  ([D128451](https://reviews.llvm.org/D128451))

## Speed

For many users, the most outstanding difference may be speed.
I have made dozens of changes scattered across the lld/ELF codebase to improve performance, with the `--start-lib` and local symbol initialization changes being the most significant changes.

Parallel input file parsing is difficult. As I mentioned in [Why isn't ld.lld faster?](/blog/2021-12-19-why-isnt-ld.lld-faster), there are

* initialization of sections (embarrassingly parallel)
* COMDAT group resolution
* initialization of local symbols (embarrassingly parallel)
* initialization of non-local symbols
* symbol resolution

As of the 15.0.0 release, I have added parallelism to parts not involving symbol resolution.
(Some diagnostics may now be non-deterministic, see [Non-deterministic diagnostics due to parallelism](https://discourse.llvm.org/t/non-deterministic-diagnostics-due-to-parallelism/64389).)

Thanks to Peter Smith and Igor Kudrin for making many good suggestions.

Note: initialization of non-local symbols and symbol resolution may affect semantics and is difficult to do with the current lld architecture.
It is challenging to keep every feature working, even a minor one like whether a symbol diagnostic is still emitted.

Beside symbol resolution, I am wondering: what are the next major performance opportunities?

Below are some programs benchmarked on an Intel Skylake machine.
I use `-DCMAKE_BUILD_TYPE=Release -DCMAKE_EXE_LINKER_FLAGS=$HOME/Dev/mimalloc/out/release/libmimalloc.a -DLLVM_ENABLE_PROJECTS='clang;lld' -DLLVM_TARGETS_TO_BUILD=X86` `-fPIC -pie` builds of lld.
(Compared with glibc malloc, linking against libmimalloc.a is 1.12x as fast.)
The host compiler is a close-to-main clang.

I run `ninja -v $target`, extract the linking command, rerun it with `-Wl,--reproduce=/tmp/rep.tar` to get a tarball which expands to linker input and a response file.
Both input and output is in tmpfs.

Linking a `-DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON` build of clang:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{14,15}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):     987.4 ms ±  10.0 ms    [User: 1230.7 ms, System: 491.6 ms]
  Range (min … max):   966.7 ms … 1009.1 ms    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):     952.0 ms ±   7.5 ms    [User: 1231.7 ms, System: 522.3 ms]
  Range (min … max):   934.1 ms … 962.3 ms    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.04 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--threads=2` => 1.01x (1.038s => 1.026s), `--threads=4` => 1.03x (1.020s => 0.9879s))

Linking a `-DCMAKE_BUILD_TYPE=Debug` build of clang:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{14,15}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.839 s ±  0.027 s    [User: 7.407 s, System: 1.838 s]
  Range (min … max):    3.786 s …  3.877 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.451 s ±  0.016 s    [User: 7.145 s, System: 1.879 s]
  Range (min … max):    3.416 s …  3.472 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.11 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--threads=1` => 1.04x (7.449s => 7.169s), `--threads=2` => 1.05x (5.514s => 5.255s))

Linking a default build of chrome:
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{14,15}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
 ⠇ Performing warmup runs
  Time (mean ± σ):      5.488 s ±  0.033 s    [User: 5.751 s, System: 2.661 s]
  Range (min … max):    5.424 s …  5.543 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      4.912 s ±  0.030 s    [User: 5.418 s, System: 2.632 s]
  Range (min … max):    4.864 s …  4.961 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.12 ± 0.01 times faster than 'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--threads=2` => 1.09x (6.349s => 5.828s), `--threads=4` => 1.11x (5.744s => 5.170s))

Linking a default build of scylladb (`./tools/toolchain/dbuild ninja build/release/scylla`):
```text
% hyperfine --warmup 2 --min-runs 16 "numactl -C 20-27 "/tmp/llvm-{14,15}/out/release/bin/ld.lld" @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      2.487 s ±  0.025 s    [User: 11.367 s, System: 1.990 s]
  Range (min … max):    2.433 s …  2.532 s    16 runs

Benchmark 2: numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      2.153 s ±  0.024 s    [User: 8.994 s, System: 1.997 s]
  Range (min … max):    2.112 s …  2.209 s    16 runs

Summary
  'numactl -C 20-27 /tmp/llvm-15/out/release/bin/ld.lld @response.txt --threads=8' ran
    1.16 ± 0.02 times faster than 'numactl -C 20-27 /tmp/llvm-14/out/release/bin/ld.lld @response.txt --threads=8'
```
(`--thread=1` => 1.26x (11.552s => 9.194s), `--thread=4` => 1.18x (3.871s => 3.267s))
