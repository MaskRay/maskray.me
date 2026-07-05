---
layout: post
title: Recent lld/ELF performance improvements
author: MaskRay
tags: [linker,llvm,performance]
---

Updated in 2026-05.

Since the LLVM 22 branch was cut, I've landed patches that parallelize more link phases and cut task-runtime overhead.
This post compares current `main` against lld 22.1, [mold](https://github.com/rui314/mold), and [wild](https://github.com/davidlattimore/wild).

Headline: a Release+Asserts clang `--gc-sections` link is 1.34x as fast as lld 22.1; Chromium debug with `--gdb-index` is 1.09x as fast. mold and wild are still ahead — the last section explains why.

<!-- more -->

## Benchmark

`lld-0201` is main at 2026-02-01 (6a1803929817); `lld-HEAD` is main at 2026-05-16 (20b0089ea340), which includes [`[ELF] Parallelize input file loading`](https://github.com/llvm/llvm-project/commit/83f8eee57d5a2e9215b506a5ddc82db980fb47d0) (commit 83f8eee57d5a) plus the later task-runtime and `Symbol` cleanups below.
`mold` and `wild` run with `--no-fork` so the wall-clock numbers include the linker process itself.

Three reproduce tarballs, `--threads=8`, `hyperfine -w 2 -r 10` (`-w 3 -r 30` for the sub-second clang-relassert link), pinned to CPU cores with `numactl -C 20-28`.

| Workload | lld-0201 | lld-HEAD | mold | wild |
| --- | --- | --- | --- | --- |
| clang-23 Release+Asserts, `--gc-sections` | 1.262 s | 940.7 ms | 599.4 ms | 375.8 ms |
| clang-23 Debug (no `--gdb-index`) | 4.409 s | 4.038 s | 2.745 s | 1.472 s |
| clang-23 Debug (`--gdb-index`) | 6.038 s | 5.627 s | 4.418 s | N/A |
| Chromium Debug (no `--gdb-index`) | 6.094 s | 5.654 s | 2.864 s | 2.033 s |
| Chromium Debug (`--gdb-index`) | 7.708 s | 7.070 s | 4.196 s | N/A |

Note that `llvm/lib/Support/Parallel.cpp` design keeps the main thread idle during `parallelFor`, so `--threads=N` really utilizes `N+1` threads.

wild does not yet implement `--gdb-index` — it silently warns and skips, producing an output about 477 MB smaller on Chromium. For fair 4-way comparisons I also strip `--gdb-index` from the response file; the `no --gdb-index` rows above use that setup.

A few observations before diving in:

* The `--gdb-index` surcharge on the Chromium link is `+1.42 s` for lld (5.65 s → 7.07 s) versus `+1.33 s` for mold (2.86 s → 4.20 s). This is currently one of the biggest remaining gaps.
* Excluding `--gdb-index`, mold is 1.47x–1.97x as fast and wild 2.50x–2.78x as fast on this machine. There is plenty of room left.
* `clang-23 Release+Asserts --gc-sections` (workload 1) has collapsed from 1.262 s to 941 ms, a 1.34x speedup over ~15 weeks. Most of that came from the parallel `--gc-sections` mark, parallel input loading, and the task-runtime cleanup below — each contributing a multiplicative factor.

### macOS (Apple M4) notes

The same `clang-23 Release+Asserts --gc-sections` workload, same `lld-0201` (6a1803929817) and `lld-HEAD` (20b0089ea340) commits, on an Apple M4 (macOS 26.2, system allocator for all four linkers), `--threads=8`, `hyperfine -w 3 -r 30`. wild has no `--chroot`, so it gets `--sysroot=` pointed at the reproduce directory to resolve the absolute paths in the `libc.so`/`libm.so` `GROUP` scripts; the work performed is identical.

| Linker | Wall | User | Sys | (User+Sys)/Wall |
| --- | --- | --- | --- | --- |
| lld-0201 | 365.4 ± 4.3 ms | 515.7 ms | 265.8 ms | 2.14x |
| lld-HEAD | 261.9 ± 4.2 ms | 493.0 ms | 471.9 ms | 3.68x |
| mold | 256.6 ± 3.0 ms | 909.9 ms | 345.5 ms | 4.89x |
| wild | 131.4 ± 1.0 ms | 468.0 ms | 319.6 ms | 5.99x |


## Parallelize `--gc-sections` mark

Garbage collection had been a single-threaded BFS over `InputSection` graph.
On a Release+Asserts clang link, `markLive` was ~315 ms of the 1562 ms wall time (20%).

[commit 6f9646a598f2](https://github.com/llvm/llvm-project/commit/6f9646a598f25efa3c4db066d2d51fb248b13526) adds `markParallel`, a level-synchronized BFS.
Each BFS level is processed with `parallelFor`; newly discovered sections land in per-thread queues, which are merged before the next level.
The parallel path activates when `!TrackWhyLive && partitions.size() == 1`.
Implementation details that turned out to matter:

* Depth-limited inline recursion (`depth < 3`) before pushing to the next-level queue. Shallow reference chains stay hot in cache and avoid queue overhead.
* Optimistic "load then compare-exchange" section-flag dedup instead of atomic fetch-or. The vast majority of sections are visited once, so the load almost always wins.

On the Release+Asserts clang link, `markLive` dropped from 315 ms to 82 ms at `--threads=8` (from 199 ms to 50 ms at `--threads=16`); total wall time 1.16x–1.18x.

Two prerequisite cleanups were needed for correctness:

* [commit 6a874161621e](https://github.com/llvm/llvm-project/commit/6a874161621ec52b8efa125790e3e8e72bb9167a) moved `Symbol::used` into the existing `std::atomic<uint16_t> flags`. The bitfield was previously racing with other mark threads.
* [commit 2118499a898b](https://github.com/llvm/llvm-project/commit/2118499a898b514f70fb1754ad8713a4267f7bd3) decoupled `SharedFile::isNeeded` from the mark walk. `--as-needed` used to flip `isNeeded` inside `resolveReloc`, which would have required coordinated writes across threads; it is now a post-GC scan of global symbols.

## Parallelize input file loading

Historically, `LinkerDriver::createFiles` walked the command line and called `addFile` serially.
`addFile` maps the file (`MemoryBuffer::getFile`), sniffs the magic, and constructs an `ObjFile`, `SharedFile`, `BitcodeFile`, or `ArchiveFile`.
For thin archives it also materializes each member.
On workloads with hundreds of archives and thousands of objects, this serial walk dominates the early part of the link.

[commit 83f8eee57d5a](https://github.com/llvm/llvm-project/commit/83f8eee57d5a2e9215b506a5ddc82db980fb47d0) rewrites `addFile` to record a `LoadJob` for each non-script input together with a snapshot of the driver's state machine (`inWholeArchive`, `inLib`, `asNeeded`, `withLOption`, `groupId`).
After `createFiles` finishes, `loadFiles` fans the jobs out to worker threads.
Linker scripts stay on the main thread because `INPUT()` and `GROUP()` recursively call back into `addFile`.

A few subtleties made this harder than it sounds:

* `BitcodeFile` and fatLTO construction call `ctx.saver` / `ctx.uniqueSaver`, both of which are non-thread-safe `StringSaver` / `UniqueStringSaver`. I serialized those constructors behind a mutex; pure-ELF links hit it zero times.
* Thin-archive member buffers used to be appended to `ctx.memoryBuffers` directly. To keep the output deterministic across `--threads` values, each job now accumulates into a per-job `SmallVector` which is merged into `ctx.memoryBuffers` in command-line order.
* `InputFile::groupId` used to be assigned inside the `InputFile` constructor from a global counter. With parallel construction the assignment race would have been unobservable but still ugly; [b6c8cba516daabced0105114a7bcc745bc52faae](https://github.com/llvm/llvm-project/commit/b6c8cba516daabced0105114a7bcc745bc52faae) hoists `++nextGroupId` into the serial driver loop and stores the value into each file after construction.

The output is byte-identical to the old lld and deterministic across `--threads` values, which I verified with `diff` across `--threads={1,2,4,8}` on Chromium.

A `--time-trace` breakdown is useful to set expectations. On Chromium, the serial portion of `createFiles` accounts for only ~81 ms of the 5.9 s wall, and `loadFiles` (after this patch) runs in ~103 ms in parallel. Serial `readFile`/mmap is not the bottleneck. What moves the needle is overlapping the per-file constructor work — magic sniffing, archive member materialization, bitcode initialization — with everything else that now kicks off on the main thread while workers chew through the job list.

## Extending parallel relocation scanning

Relocation scanning has been parallel since LLVM 17, but three cases had opted out via `bool serial`:

1. `-z nocombreloc`, because `.rela.dyn` merged relative and non-relative relocations and needed deterministic ordering.
2. MIPS, because `MipsGotSection` is mutated during scanning.
3. PPC64, because `ctx.ppc64noTocRelax` (a `DenseSet` of `(Symbol*, offset)` pairs) was written without a lock.

[commit 076226f378df](https://github.com/llvm/llvm-project/commit/076226f378df115622d0d959e975ee7c7fb3c051) and [commit dc4df5da886e](https://github.com/llvm/llvm-project/commit/dc4df5da886e09d36577b3302952bc91b5e7e154) separate relative and non-relative dynamic relocations unconditionally and always build `.rela.dyn` with `combreloc=true`; the only remaining effect of `-z nocombreloc` is suppressing `DT_RELACOUNT`.
[commit 2f7bd4fa9723](https://github.com/llvm/llvm-project/commit/2f7bd4fa97232dfab7f2347c745005eb9e2ffd2d) then protects `ctx.ppc64noTocRelax` with the already-existing `ctx.relocMutex`, which is only taken on rare slow paths.
After these changes, only MIPS still runs scanning serially.

## Target-specific relocation scanning

Relocation scanning used to go through a generic loop in `Relocations.cpp` that called `Target->getRelExpr` through a virtual for every relocation — once to classify the expression kind (PC-relative, PLT, TLS, etc.) and again from the TLS-optimization dispatch. On any realistic link that is a hot inner loop running over tens of millions of relocations, and the virtual call plus its post-dispatch switch are a real fraction of the cost.

The fix is to move the whole per-section scan loop into target-specific code, so each `Target::scanSection` / `scanSectionImpl` pair can inline its own `getRelExpr`, handle TLS optimization in-place, and specialize for the two or three relocation kinds that dominate on that architecture. Rolled out across most backends in early 2026:

* [4b887533389c](https://github.com/llvm/llvm-project/commit/4b887533389c78e3e678b3af85d1dc8e3bf59e83) x86 (i386 / x86-64). On lld's own object files, `R_X86_64_PC32` and `R_X86_64_PLT32` make up ~95% of relocations and now hit an inlined hot path.
* [371e0e2082e9](https://github.com/llvm/llvm-project/commit/371e0e2082e9) AArch64, [4ea72c1e8cbd](https://github.com/llvm/llvm-project/commit/4ea72c1e8cbd) RISC-V, [cd01e6526af6](https://github.com/llvm/llvm-project/commit/cd01e6526af6) LoongArch, [c04b00de7508](https://github.com/llvm/llvm-project/commit/c04b00de7508) ARM, [6d9169553029](https://github.com/llvm/llvm-project/commit/6d9169553029) Hexagon, [aec1c984266c](https://github.com/llvm/llvm-project/commit/aec1c984266c) SystemZ, [5e87f8147d68](https://github.com/llvm/llvm-project/commit/5e87f8147d68) PPC32, [aecc4997bf12](https://github.com/llvm/llvm-project/commit/aecc4997bf12) PPC64.

Besides devirtualization, inlining TLS relocation handling into `scanSectionImpl` let the TLS-optimization-specific expression kinds be replaced with general ones: `R_RELAX_TLS_GD_TO_LE` / `R_RELAX_TLS_LD_TO_LE` / `R_RELAX_TLS_IE_TO_LE` fold into `R_TPREL`, `R_RELAX_TLS_GD_TO_IE` folds into `R_GOT_PC`, and `getTlsGdRelaxSkip` goes away. What remains in the shared dispatch path — `getRelExpr` called from `relocateNonAlloc` and `relocateEH` — is a much smaller set.

Average `Scan relocations` wall time on a clang-14 link (`--threads=8`, x86-64, 50 runs, measured via `--time-trace`) drops from 110 ms to 102 ms, ~8% from the x86 commit alone.

## Faster `getSectionPiece`

Merge sections (`SHF_MERGE`) split their input into "pieces".
Every reference into a merge section needs to map an offset to a piece.
The old implementation was always a binary search in `MergeInputSection::pieces`, called from `MarkLive`, `includeInSymtab`, and `getRelocTargetVA`.

[commit 42cc45477727](https://github.com/llvm/llvm-project/commit/42cc454777274a06933abcd098ec3281158717f9) changes this in two ways:

1. For non-string fixed-size merge sections, `getSectionPiece` uses `offset / entsize` directly.
2. For non-section `Defined` symbols pointing into merge sections, the piece index is pre-resolved during `splitSections` and packed into `Defined::value` as `((pieceIdx + 1) << 32) | intraPieceOffset`.

The binary search is now limited to references via section symbols (addend-based), which is common on AArch64 but rare on x86-64 where the assembler emits local labels for `.L` references into mergeable strings.
The clang-relassert link with `--gc-sections` is 1.05x as fast.

## Optimizing the underlying `llvm/lib/Support/Parallel.cpp`

All of the wins above rely on `llvm/lib/Support/Parallel.cpp`, the tiny work-stealing-ish task runtime shared by lld, dsymutil, and a handful of debug-info tools.
Four changes in that file mattered:

* [commit c7b5f7c635e2](https://github.com/llvm/llvm-project/commit/c7b5f7c635e2534a9b2b2b204998b0bc39921b7e) — `parallelFor` used to pre-split work into up to `MaxTasksPerGroup` (1024) tasks and spawn each through the executor's mutex + condvar. It now spawns only `ThreadCount` workers; each grabs the next chunk via an atomic `fetch_add`. On a clang-14 link (`--threads=8`), futex calls dropped from ~31K to ~1.4K (glibc release+asserts); wall time 927 ms → 879 ms. This is the reason the parallel mark and parallel scan numbers are worth quoting at all — on the old runtime, spawn overhead was a real fraction of the work being parallelized.
* [commit 9085f74018a4](https://github.com/llvm/llvm-project/commit/9085f74018a4f465afa84815d64af850f09b733f) — `TaskGroup::spawn()` replaced the mutex-based `Latch::inc()` with an atomic `fetch_add` and passes the `Latch&` through `Executor::add()` so the worker calls `dec()` directly. Eliminates one `std::function` construction per spawn.
* [commit 5b1be759295c](https://github.com/llvm/llvm-project/commit/5b1be759295c4a2f357fbad852e04c74fc012dc1) — removed the `Executor` abstract base class. `ThreadPoolExecutor` was always the only implementation; `add()` and `getThreadCount()` are now direct calls instead of virtual dispatches.
* [commit 8daaa26efdda](https://github.com/llvm/llvm-project/commit/8daaa26efdda3802f73367d844b267bda3f84cbe) — enables nested parallel `TaskGroup` via work-stealing. Historically, nested groups ran serially to avoid deadlock (the thread that was supposed to run a nested task might be blocked in the outer group's `sync()`). Worker threads now actively execute tasks from the queue while waiting, instead of just blocking. Root-level groups on the main thread keep the efficient blocking `Latch::sync()`, so the common non-nested case pays nothing. In lld this lets `SyntheticSection::writeTo` calls with internal parallelism (`GdbIndexSection`, `MergeNoTailSection`) parallelize automatically when called from inside `OutputSection::writeTo`, instead of degenerating to serial execution on a worker thread — which was the exact situation [D131247](https://reviews.llvm.org/D131247) had worked around by threading a root `TaskGroup` all the way down.

## Small wins worth mentioning

* [036b755daedb](https://github.com/llvm/llvm-project/commit/036b755daedb) parallelizes `demoteAndCopyLocalSymbols`. Each file collects local `Symbol*` pointers in a per-file vector via `parallelFor`, which are merged into the symbol table serially. Linking clang-14 (`--no-gc-sections`) with its 208K `.symtab` entries is 1.04x as fast.

The xxh3 `hash_combine` swap ([71d78b2220e4](https://github.com/llvm/llvm-project/commit/71d78b2220e4dc4b022fd74aec16ed8d93fc419e)) and the `Symbol` constructor-init plus redundant-`memset` removal ([905a88b92343](https://github.com/llvm/llvm-project/commit/905a88b923433eb8cd83677ea55bee82eb9ba498), [20b0089ea340](https://github.com/llvm/llvm-project/commit/20b0089ea340d6c9355e631c81f0ce82d80263e5)) have minor improvements.

| lld build | clang-relassert link |
| --- | --- |
| pre-xxh3 (525fab579da1) | 939.0 ± 29.0 ms |
| xxh3 (71d78b2220e4) | 932.2 ± 23.3 ms |
| pre Symbol-init (2e4c820c05fd) | 928.4 ± 31.4 ms |
| HEAD (20b0089ea340) | 926.3 ± 30.0 ms |

## Where lld still loses time

To locate the gap I ran `lld --time-trace`, `mold --perf`, and `wild --time` on the clang-relassert link (clang-23 Release+Asserts, `--gc-sections`, `--threads=8`; per-phase numbers are 5-run averages). Grouped into comparable phases:

| Work scope | lld-0201 | lld-HEAD | mold | wild |
| --- | --- | --- | --- | --- |
| mmap + parse sections + merge strings + symbol resolve | 391 ms | 320 ms | 218 ms | 120 ms |
| `--gc-sections` mark | 285 ms | 76 ms | 36 ms | — \* |
| Scan relocations | 116 ms | 91 ms | 61 ms | — \* |
| Assign / finalize / symtab | 77 ms | 86 ms | 27 ms | 86 ms |
| Write sections | 87 ms | 87 ms | 77 ms | 103 ms |
| **Wall (hyperfine)** | **1262 ms** | **941 ms** | **599 ms** | **376 ms** |

\* wild fuses `--gc-sections` marking and relocation-driven live-section propagation into one `Find required sections` pass (62 ms), so these two rows are effectively merged.

A subtlety on wild's parse number: wild's `Load inputs into symbol DB` phase by itself is only 24 ms, but it does only `mmap` + `.symtab` scan + global-name hash bucketing. Section-header parsing, mergeable-string splitting, COMDAT handling, and symbol resolution are deferred to later wild phases. The 120 ms row above sums those (`Load inputs into symbol DB` 24 + `Resolve symbols` 13 + `Resolve alternative symbol definitions` 4 + `Section resolution` 21 + `Merge strings` 58) so it covers the same work lld calls `Parse input files`.

Meaningful gaps, in order of absolute impact:

**Parse: lld-HEAD 320 ms vs wild 120 ms ≈ 2.7x.** The biggest remaining cross-linker gap on this workload, and the same pattern holds on the larger workloads below. The phase is already parallel; the gap is constant factor in the per-object parse path (reading section headers, interning strings, splitting CIEs/FDEs, merging globals into the symbol table). On clang-relassert the 200 ms parse gap alone accounts for ~35% of the 565 ms wall-clock gap between lld-HEAD and wild.

**Assign / finalize / symtab: 86 ms vs mold 27 ms ≈ 3.2x.** `finalizeAddressDependentContent`, `assignAddresses`, `finalizeSynthetic`, `Add symbols to symtabs`, and `Finalize .eh_frame` together cost ~86 ms on this workload; mold's equivalents (`compute_section_sizes`, `compute_symtab_size`, `create_output_sections`, `set_osec_offsets`) total 27 ms. This gap grows linearly with the number of `.symtab` entries — on clang-debug it's 127 ms lld vs 27 ms mold, on Chromium 570 ms vs ~80 ms. I have a local branch that turns `SymbolTableBaseSection::finalizeContents` into a prefix-sum-driven parallel fill and replaces the `stable_partition` + `MapVector` shuffle with per-file `lateLocals` buffers. 1640 ELF tests pass; not posted yet.

**`markLive`: 76 ms, 3.7x faster than the Feb 1 baseline (285 ms).** This is apples-to-oranges comparison:
lld supports `__start_`/`__stop_` edges, `SHF_LINK_ORDER` dependencies, linker scripts `KEEP`, and others features.
lld correctly handles `--gc-sections --as-needed` with `Symbol::used` (tests `gc-sections-shared.s`, `weak-shared-gc.s`, `as-needed-not-in-regular.s`):

* **mold over-approximates `DT_NEEDED` on two axes**: it emits `DT_NEEDED` for DSOs referenced only via weak relocs, and for DSOs referenced only from GC'd sections. It also retains undefined symbols that are only reachable from dead sections in `.dynsym`.
* **wild handles weak refs correctly but not dead-section refs**: weak-only references do not force `DT_NEEDED` (matching lld), but DSOs referenced only from GC'd sections still get `DT_NEEDED` entries. wild does drop the corresponding undefined symbols from `.dynsym`, so its `DT_NEEDED` decision and its symtab-inclusion decision diverge slightly.
* **lld is strictest on all three axes**

**Scan relocations: 91 ms vs 61 ms.** Clean 1.5x ratio, small absolute. Target-specific scanning (the `Add target-specific relocation scanning for …`) removed some dispatch overhead; what remains is `InputSectionBase::relocations` overhead. wild folds relocation-driven liveness into `Find required sections`, which is why there's no separate wild row.

Interestingly, **writing section content is not a gap** (77–103 ms across all four). The earlier assumption that `.debug_*` section writes were a lld weakness didn't survive measurement.

One cost that only shows up on debug-info-heavy workloads is `--gdb-index` construction, which lld does in ~1.3 s vs mold's ~0.9 s on Chromium. The work is embarrassingly parallel per input, but lld funnels string interning through a sharded `DenseMap`; mold uses a lock-free `ConcurrentMap` sized by HyperLogLog. wild does not yet implement `--gdb-index`.

wild is worth calling out separately: its user time is comparable to lld's but its system time is roughly half, and its parse phase is 4-8x faster than either of the C++ linkers across all three workloads.
mold is at the other extreme — the highest user time on every workload, bought back by aggressive parallelism.
