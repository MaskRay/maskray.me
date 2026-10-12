---
layout: post
title: ClangIR and the cost of MLIR
date: 2026-10-11
tags: [llvm,clang]
---

ClangIR (CIR) is an MLIR dialect between the AST and LLVM IR, and in October 2026 the clang area team [accepted building it by default](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/100).
CIR is built on MLIR because each step along the way was locally reasonable, and together the steps locked MLIR in.
The costs (build time, binary size, MLIR API churn) land largely on Clang contributors who neither chose MLIR nor use CIR.

This post traces how CIR came to depend on MLIR, collects what the dependency costs, and uses an MLIR-free CIR as a thought experiment to separate the costs inherent to MLIR from those that can be reduced.

<!-- more -->

This post relies heavily on an LLM agent (Claude Code).
It read the Discourse threads, GitHub PRs, slides, and the source tree, computed the build-closure and contributor numbers, and drafted the text, which I then edited.
My part was steering, and some of what I added to its context:

* the build-by-default and premerge-testing RFCs, and a discussion on Discord #llvm;
* the opportunity-cost framing: contributors without fast machines pay for MLIR (and Flang) on every llvm/ change, though most of them never use either.

Treat the numbers as checked but not peer-reviewed.

## What ClangIR replaces

Classic Clang lowers the AST to LLVM IR in `clang/lib/CodeGen`.
With `-fclangir`, `clang/lib/CIR/CodeGen` (CIRGen) emits CIR instead, a set of CIR passes runs, CIR is converted to MLIR's `llvm` dialect, and MLIR translates that to LLVM IR.

* CIRGen mirrors the AST-to-IR half of `clang/lib/CodeGen` file by file: `CodeGenFunction` becomes `CIRGenFunction`, `CGExprScalar.cpp` becomes `CIRGenExprScalar.cpp`, and so on.
  It is 66K lines against CodeGen's 214K, with 786 `errorNYI` calls marking what is missing.
* Work that classic CodeGen does while emitting IR becomes separate CIR passes: C++ ABI lowering (`CXXABILowering`), calling conventions (`CallConvLowering`), exception handling (`EHABILowering`), and structured control flow to a CFG (`FlattenCFG`).
* The optimizer and backend are not replaced.
  CIR's frontend action calls `emitBackendOutput` from `clang/lib/CodeGen/BackendUtil.cpp`, so `clangCIRFrontendAction` links `clangCodeGen`.
* Some code is shared rather than duplicated: `clang/lib/CodeGenUtils` ([#221001](https://github.com/llvm/llvm-project/pull/221001), September 2026) holds AST-only helpers used by both, and `llvm/lib/ABI` (LLVMABI) does ABI classification for CIR's calling-convention lowering and, under `-fexperimental-abi-lowering`, for classic CodeGen.
* Nothing in `clang/lib/Analysis` is replaced yet.
  The CFG, the analyses behind Sema's analysis-based warnings, the dataflow framework, lifetime safety, and the Static Analyzer all still work on the AST and the CFG.
  Upstream CIR's only analysis is an alias analysis used by ABI lowering.
  Replacing CFG-based analysis is an aspiration rather than a plan: the 2022 RFC argued that the CFG is "discarded after analysis" and "has lots of known problems", and the area team expects CIR to replace "the current CodeGen and CFG analysis", but no ClangIR document schedules it.

The in-tree roadmap, [`clang/docs/CIR/CodeDuplication.md`](https://github.com/llvm/llvm-project/blob/main/clang/docs/CIR/CodeDuplication.md), is about CodeGen.
It plans to raise CIR to a higher level, move ABI details into later lowering passes, and "replace relevant parts of the existing implementation, piece by piece" with implementations shared by both code generators.
Calling conventions are the first case (LLVMABI, plus an MLIR adapter that FIR can also use).
The roadmap lists the next candidates: constant expression evaluation, complex multiplication and division, builtins, exception handling and C++ cleanups, inline assembly, and C++ ABI handling (vtables, virtual calls, constructor and destructor arguments, dynamic casts, base class addresses, type descriptors, array new and delete).
Linkage, alignment, debug info, TBAA, sanitizers, and lifetime markers are spread through CodeGen and listed as harder to share.
It also aims to run all classic CodeGen tests through the CIR path.

## Where things stand

The area team's decision says "ClangIR is likely to reach the point where it becomes the replacement for the current CodeGen and CFG analysis."
The same post promises that `-DCLANG_ENABLE_CIR=Off` "will still be a supported configuration for the forseeable future", and `-fclangir` stays off by default.
On main, `clang/CMakeLists.txt` tests `CLANG_ENABLE_CIR` but declares no `option()` for it, so the flip is still pending.

A separate RFC, [Add CIR testing to clang and MLIR CI](https://discourse.llvm.org/t/rfc-add-cir-testing-to-clang-and-mlir-ci/92452) ([#227834](https://github.com/llvm/llvm-project/pull/227834)), would build and test CIR in premerge for clang/ and mlir/ changes.
nikic is [weakly -1](https://discourse.llvm.org/t/rfc-add-cir-testing-to-clang-and-mlir-ci/92452/9): if contributors have to fix CIR failures, "they also need to build and test CIR locally", so premerge testing and build-by-default "should go hand in hand".
rengolin [calls](https://discourse.llvm.org/t/rfc-add-cir-testing-to-clang-and-mlir-ci/92452/10) asking non-CIR developers to stub out CIR breakage "an unreasonable request" and suggests a different CI setup.

As for the compiler itself, the [ClangIR progress report for September 2026](https://discourse.llvm.org/t/clangir-progress-report-september-2026/91947) measures `-fclangir` against classic CodeGen: 1.32x the wall time on a typical file, 1.57x across the tree, 1.17x the memory, and 3% more code.

## Why MLIR

### Meta, 2022: framework reuse for a lifetime checker

ClangIR started at Meta in October 2021.
Bruno Cardoso Lopes and Nathan Lanza proposed it in [RFC: An MLIR based Clang IR (CIR)](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319) in June 2022, and it moved to an LLVM incubator in August 2022 (timeline from the [2025 keynote](https://llvm.org/devmtg/2025-10/slides/keynotes/kaylor_lopes.pdf)).
The primary goal was "improved diagnostics for modern C++"; the design had "grown out of implementing a lifetime analysis/checker pass"; optimization and lowering were "a secondary goal for now".
Clang lacked a suitable IR: the AST is awkward for dataflow, LLVM IR is too low level, and the Clang CFG is discarded after analysis.
The RFC's entire "Why MLIR?" section reads:

> MLIR provides a solid and tested framework to build custom IR and run passes. Among several other capabilities, it's also in already in tree and used by Flang to build FIR.

Nobody in the thread argued for a bespoke IR; the closest was John McCall's [aside](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/31) that if MLIR needed in-tree dialects to be useful, "it would be better to just not use MLIR" and write the IR "from scratch on top of llvm::ilist".
efriedma [wrote](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/2) "using MLIR to build it seems like the right approach" and objected to structured control flow instead.
Gábor Horváth [argued](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/4) that the CFG suffices for lifetime analysis, and John McCall [wanted](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/15) a two-stage lowering like Swift's.
Reid Kleckner [asked](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/12) for an agreed compile-time ceiling, "like +15-25%, before making this part of the default pipeline".
`-fclangir` is still opt-in, and 1.32x/1.57x is already past that ceiling.

Lopes's later talks do not revisit the choice.
The [2023 Developers' Meeting talk](https://web.archive.org/web/2024/http://brunocardoso.cc/resources/2023-LLVMDevMtgClangIR.pdf) ([video](https://www.youtube.com/watch?v=XNOPO3ogdfQ)) describes CIR as "Implemented as an MLIR dialect for C/C++".
Its pipeline diagram goes from CIRGen through CIR passes, including the lifetime checker, to LLVM lowering, with a dashed side path through SCF and Affine.
The lifetime checker was real: the talk reports integration with clangd, clang-tidy, and an internal IDE, and "Successful detection of lifetime bug that caused a major internal outage".
The same talk lists "Compile & execution time, binary size, memory usage" among the challenges.
The [project site](https://llvm.github.io/clangir/Development/motivation.html) gives today's rationale: MLIR enables "the development of passes and IR, rapid design iteration, and reuse of community-provided analyses and transformations".
Apart from a few OpenMP passes, clang's CIR pipeline runs only CIR's own passes.

The lifetime checker never reached upstream CIR.
Clang's lifetime-safety analysis ([#142313](https://github.com/llvm/llvm-project/pull/142313), July 2025) is built on the CFG, and Horváth is among its authors.

### NVIDIA, 2023: OpenACC lives in MLIR

In October 2023 Erich Keane (NVIDIA) posted [RFC: Implementation of OpenACC 3.3 for offload in Clang](https://discourse.llvm.org/t/rfc-implementation-of-openacc-3-3-for-offload-in-clang/74405).
Flang lowers OpenACC through MLIR's `acc` dialect: "It is our intention to use that as well, so NVIDIA is beginning to participate in the CIR/MLIR effort".
Clang shares more than the dialect with Flang.
CIR and FIR implement the same `acc` type interfaces (`CIROpenACCTypeInterfaces.cpp`, `FIROpenACCTypeInterfaces.cpp`), and acc-to-GPU passes are landing in `mlir/lib/Dialect/OpenACC/Transforms`.
Classic CodeGen has no OpenACC lowering (`CodeGenFunction.h`: "TODO OpenACC: Implement this"), so CIR plus `acc` is Clang's only OpenACC implementation.

Strictly, the requirement was MLIR's `acc` dialect, not CIR.
Keane's [fallback](https://discourse.llvm.org/t/rfc-implementation-of-openacc-3-3-for-offload-in-clang/74405/1) was "LLVM-IR based metadata to enable the translation to the OpenACC MLIR dialect", and the [second fallback](https://discourse.llvm.org/t/rfc-implementation-of-openacc-3-3-for-offload-in-clang/74405/9), "lowering as OMP does", gives up sharing with Flang.
OpenMP has the same sharing benefit without the necessity: CIR emits `omp` ops, but classic CodeGen already lowers OpenMP.

By 2026 the motive had widened.
Keane [wrote](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/33) that the goal is to "enable further advancement for our compiler stack … the way to allow those to integrate with C/C++/Clang is to have a dialect for it".

### 2024–2026: lock-in

Lopes [proposed upstreaming](https://discourse.llvm.org/t/rfc-upstreaming-clangir/76587) in January 2024, Lanza landed the scaffolding in April 2024, and NVIDIA's first upstream CIR commit (David Olsen) followed in November 2024.
Today CIR is 120K lines (lib + include) on ODS, MLIR interfaces, and MLIR rewrite drivers, with ~230 ops and 1,269 test files, 868 of which check `-emit-cir` output in MLIR's textual format.

Readers might ask whether NVIDIA led the upstreaming.
I counted commits touching `clang/lib/CIR`, both upstream and in the archived llvm/clangir incubator (excluding incubator commits that also exist upstream).
Affiliation comes from email domains, GitHub profiles, and LinkedIn; a GitHub profile shows the current employer, not the employer at commit time.

| Author | Incubator only (2,498, 2021-08..2026-02) | Upstream (2,023) |
|---|---:|---:|
| Meta | 55% | 1.5% |
| NVIDIA | 2.7% | 47% |
| Amr Hesham (GSoC 2025, then kept going) | 5% | 14% |
| unknown | 28% | 27% |

Meta wrote the incubator: its share was 100% in 2021, 97% in 2022, about half in 2023–2024, and 11% in 2025.
NVIDIA wrote 45–49% of upstream commits each year, so by volume NVIDIA led the upstreaming; by authorship of the RFC and the scaffolding, Meta did.
Of the 260 upstream commits titled "upstream …", the explicit ports from the incubator, Amr Hesham wrote 37%, NVIDIA 31%, and unknown authors 31%.
Most NVIDIA commits are new code written upstream.
The maintainers are Andy Kaylor (NVIDIA), Bruno Cardoso Lopes (Meta), and Henrich Lauko (NVIDIA since August 2026, previously Trail of Bits and NextSilicon).

The stated long-term value is mixing dialects.
From Lopes's [SC24 talk](https://llvm.github.io/clangir/Files/sc24-clangir.pdf) (November 2024) on, the slides describe CIR as a way to "Move Clang onto the MLIR substrate" and "Use MLIR from C, C++ and extensions", and to "Mix CIR with downstream and custom dialects".
The 2025 keynote with Andy Kaylor lists "Lower to other MLIR dialects" among CIR's uses, next to the lifetime checker, and has a slide titled "CIR Won't Always Go To LLVM IR", where CIR is consumed by MLIR to "Offload to GPU or other targets".
`clang/docs/CIR/CodeDuplication.md`: "even now we generate the ACC (OpenACCC) [sic] dialect in combination with CIR, and when support for OpenMP and CUDA are added, similar mixing will occur".
The area team's acceptance ratifies this direction.
Every year of upstreaming raised the cost of switching, and the main funder's motive now depends on MLIR itself, not just on the original team's choice of framework.

## Promises and delivery

CIR's case has been made in RFCs and talks since 2022.
Here is what was promised and where it stands upstream in October 2026.

| Promise | Source | Upstream today |
|---|---|---|
| Diagnostics: a lifetime checker, coroutine checks, `std::optional` dereference checks, remarks on expensive copies, cross-TU summaries | [2022 RFC](https://discourse.llvm.org/t/rfc-an-mlir-based-clang-ir-cir/63319/1) | None. The incubator's lifetime checker, points-to and live-object analyses, and clang-tidy module were archived in February 2026; Kaylor's [transition post](https://discourse.llvm.org/t/clangir-transition-plans-for-the-clangir-incubator/89583) lists the analysis passes as not yet upstreamed. Lifetime safety landed on the CFG. |
| Objective-C "in the future" | 2022 RFC | No Objective-C CIRGen; those paths hit `errorNYI`. |
| A bridge to Polygeist | 2022 and 2024 RFCs | None; ThroughMLIR stayed in the incubator. |
| Compile-time benefits from MLIR's "function pass level parallelism" | [2024 RFC](https://discourse.llvm.org/t/rfc-upstreaming-clangir/76587/1) | CIR's passes are scheduled on the whole module, so the pass manager has nothing to run in parallel. |
| It "wouldn't be feasible to ever flip the switch without proper compile time due diligence and improvements" | [Lopes, 2024](https://discourse.llvm.org/t/rfc-upstreaming-clangir/76587/31) | Build-by-default was accepted at a claimed 1.2x; nikic [measures](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/103) +50% on c-t-t. |
| On breakage, "the burden should be on CIR folks to catch up, no revert needed"; expectations on other maintainers: "none" | [Lopes, 2024](https://discourse.llvm.org/t/rfc-upstreaming-clangir/76587/31) | A patch that breaks CIR "would require 'fixing' ClangIR tests" ([2026](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/1)), and premerge testing is proposed for every clang/ and mlir/ change. |
| "low hanging fruits" and "New CIR passes and optimizations" | [AsiaLLVM 2025](https://llvm.github.io/clangir/Files/asiallvm-brunolopes-25-cir-compile-time.pdf) | `LibOpt` is a 75-line skeleton; `IdiomRecognizer` recognizes a few library calls such as `strlen`. |
| "optimization passes that have shown SIGNIFICANT improvements in certain workloads" | [Keane, 2026](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/1) | None upstream; Keane [describes](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/9) "admittedly only a few months of work" by an intern. |
| "Eventually we'd like to run all existing codegen tests with CIR enabled" | [2025 keynote](https://llvm.org/devmtg/2025-10/slides/keynotes/kaylor_lopes.pdf) | Pending. |
| Modules, debug info, sanitizers, and non-x86 targets, "work we intend to progress on" | [Keane, 2026](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/1) | In September, `-g` produced no debug info under `-fclangir`, and the [progress report](https://discourse.llvm.org/t/clangir-progress-report-september-2026/91947) had "Still no ARM64 numbers". |

The functional milestones have been met, some on schedule.
The MultiSource test suite passes, and the stage-2 self-build Keane [predicted](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/36) "within another month" in September works: all 4,744 translation units compile, and Lopes [reports](https://discourse.llvm.org/t/clangir-progress-report-september-2026/91947/5) check-clang passing on the self-built clang.
Shared AST queries moved to `clang/lib/CodeGenUtils` ([#221001](https://github.com/llvm/llvm-project/pull/221001)), ABI lowering has a shared library, and unimplemented features produce `errorNYI` diagnostics, as the 2024 RFC planned.
What has not arrived is the reason the project was started, better diagnostics, and the optimizations that later took its place, along with the promises about the cost to everyone else.

## The costs

### Build

aengelke's [-1](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/5) in the build-by-default thread starts from "I usually don't build MLIR".
His measurements of `ninja clang` in a dylib build track the trimming work:

| Date | Landed before | CPU time | Wall time | [c-t-t](https://llvm-compile-time-tracker.com/) build time |
|---|---|---:|---:|---:|
| [2026-09-05](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/22) | | +130% | +155% | |
| [2026-09-09](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/61) | [#221616](https://github.com/llvm/llvm-project/pull/221616), [#221813](https://github.com/llvm/llvm-project/pull/221813) | +40% | +83% | +58% |
| [2026-09-14](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/87) | [#222576](https://github.com/llvm/llvm-project/pull/222576) | +30% | +71% | +47% |

My #221616 links only the MLIR dialects CIR uses; on 192 cores PragmaTwice [measured](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/53) it alone taking `ninja clang` wall time from +109% to +75%.
#221813 shards the LLVM dialect's generated code, and #222576 gives MLIRIR a precompiled header.
On 2026-09-14, `libclang-cpp.so` in the dylib build was still +91% (+83% stripped); on c-t-t, the `clang` binary was +51% (+68 MiB) and the build's max-rss +40%.
The area team's "roughly 1.2x for a build only building clang" is wall time of a parallel build, which hides much of the CPU cost.

The c-t-t [comparison](https://llvm-compile-time-tracker.com/compare_clang.php?from=22287cb048390352d95d8a4925dfb9f96835444d&to=aa13846b3edeaa2795b16dd3d59d54e6890ea2c3&stat=wall-time) for the "enable clangir" commit lists per-file compile times.
Summed, they add 1,644 s (+48%): 751 s for CIR's own 100 object files (`CIRDialect.cpp.o` alone takes 44 s) and 893 s for MLIR (853 s in 380 object files, the rest in its precompiled headers and TableGen tools).
These sums measure work, not wall time.

`clang` still links a closure of 90 MLIR libraries, and some are unused:

* `MLIRArithToLLVM` depends on `MLIRArithTransforms` for one helper, `populateCeilFloorDivExpandOpsPatterns` ([#118839](https://github.com/llvm/llvm-project/pull/118839)), which drags in Bufferization, SparseTensor, Tensor, Vector, and Shard (~177 s); see [Reducing the cost](#reducing-the-cost).
* `MLIROpenACCDialect` depends on `MLIRGPUDialect` (~38 s, [#174671](https://github.com/llvm/llvm-project/pull/174671)).
* PDL (~21 s) can be disabled with the existing `MLIR_ENABLE_PDL_IN_PATTERNMATCH` option.

The often-cited NVVM and ROCDL dialects (`NVVMDialect.cpp.o` 108 s and `ROCDLDialect.cpp.o` 134 s in aengelke's 2026-09-05 list) are no longer in the closure after #221616; only 3.3 s of their TableGen output remains, via the `mlir-headers` umbrella target.

MLIR's headers are heavy.
For files under `tools/mlir/`, the frontend is 75% of compile time and header parsing 55% (aengelke's numbers in [#213412](https://github.com/llvm/llvm-project/pull/213412)).
MLIRIR now has a precompiled header, but CIR's own libraries do not, and `AddLLVM.cmake` lets a library reuse the precompiled header of at most one direct dependency.
CIRGen reuses clangAST's, so each of its files reparses the MLIR and CIR dialect headers; a private precompiled header like clangCodeGen's ([#206018](https://github.com/llvm/llvm-project/pull/206018)) cuts CIRGen's summed compile time by 72%.

### Contributors without a build farm

Most of the numbers above come from many-core machines, and wall time on a many-core machine hides the cost.
On aengelke's 96-thread machine, `ninja clang` used 30% more CPU time but 71% more wall time, because a few slow files dominate the critical path (on c-t-t, `CIRDialect.cpp.o` takes 44 s and MLIR's `OpenACC.cpp.o` 31 s).
With fewer cores, wall time approaches CPU time divided by the core count: the extra 2,100 CPU-seconds of that build are 4–5 minutes on 8 threads, for every clean build.

Incremental builds matter more to someone iterating on a patch.
From the c-t-t per-file times and the header dependencies of my MLIR-enabled build:

* Linking `clang` is 38% slower on c-t-t (105 s to 145 s); with lld it stays under a second.
* Touching `llvm/ADT/DenseMap.h`, `StringRef.h`, or `raw_ostream.h` recompiles 2,504 files (3,274 s summed) without CIR; MLIR adds 380 files (853 s) and CIR another 100 (751 s), +49%.
* Touching `llvm/IR/Function.h` recompiles 58 more MLIR files (+16%), before counting CIR's lowering.
* Touching `clang/AST/Decl.h` recompiles 521 files (1,198 s) without CIR; CIRGen adds at least 51 files and 415 s, +35%.
  CIRGen alone costs 2.2x classic CodeGen (104 files, 189 s); in my parallel build, 1.6x (351 s vs 221 s).

Memory and disk grow too: c-t-t's build max-rss is +40% (+1 GiB), which matters when running many jobs on a 16 or 32 GiB machine, and aengelke's build directory grew from 1.5 GiB to 2.4 GiB.

`-DCLANG_ENABLE_CIR=Off` stays supported, but once premerge tests CIR, a contributor who opts out learns about CIR breakage from CI and then has to build CIR anyway.
Flang shows where this goes.
`.ci/compute_projects.py` tests flang and mlir for every llvm/ change, so fixing a breakage means building Flang and MLIR.
In my full build (clang, mlir, flang, lld, …), summed per-file compile time splits clang 25.6%, mlir 19.7%, llvm 16.2%, flang 14.5%.
Users who need Flang or OpenACC can get them from a vendor toolchain; a contributor whose llvm/ change breaks them has to build them to fix it.
Companies that fund CIR and Flang have build farms; many volunteers and contributors from smaller companies have a desktop or a laptop, and each component that becomes everyone's responsibility raises the bar for them.

### Runtime, for every clang user

Building CIR in costs something even without `-fclangir`.
On c-t-t, stage2-O2-g compile time is +0.18% and max-rss +2.6%.
In aengelke's dylib build, `clang -emit-llvm -c empty.c` executes 12% more instructions, down from +88% when clang still linked `libMLIR.so`, and `clang -c empty.c` still crashed on 2026-09-14.
aengelke names dynamic relocations, global constructors, and additional `cl::opt` as probable causes.

With `-fclangir`, the five worst files in the progress report took 2.0–4.1x as long, "almost all in lowering CIR to LLVM IR: 30–92 s per file, roughly split between the CIR-to-LLVM-dialect conversion and the translation to LLVM IR".
A linear scan for comdats made that quadratic; [#227127](https://github.com/llvm/llvm-project/pull/227127) cut the lowering to 8–11 s.
Quadratic paths keep surfacing through the `llvm` dialect: [#198424](https://github.com/llvm/llvm-project/pull/198424) for `insertvalue` globals, and Flang's [#140268](https://github.com/llvm/llvm-project/pull/140268) and [#209393](https://github.com/llvm/llvm-project/issues/209393).

For typical files, I compiled 29 random C++ files from `llvm/lib` and `clang/lib` with their build flags (-O3) and `-ftime-trace`, with and without `-fclangir`, using a Release+Asserts clang at a160747d89a3 (after #227127).
`-fclangir` took 1.15x as long in total, 1.12x for the median file, below the progress report's 1.32x.
A later run on 40 files at -O0 and -O2 (1.20x and 1.17x with assertions) found that about a third of the extra time in assertions builds is a type walk: `cir::FuncType::verify` ([#215922](https://github.com/llvm/llvm-project/pull/215922)) walks every parameter type, and MLIR verifies types inside an assert on every `FuncType::get`.
A release clang should be about 1.14x at -O0 and 1.12x at -O2, though it still runs the CIR verifier after CIRGen and every CIR pass, while the driver disables LLVM's verifier in release builds (about 10% of the extra).
The extra time splits roughly in thirds:

| Phase | Extra time | Share of the `-fclangir` compile |
|---|---:|---:|
| CIRGen (vs. classic IR generation) | 3.6 s | 4.4% |
| CIR-to-CIR passes | 3.0 s | 3.6% |
| CIR to LLVM IR (pre-lowering passes, conversion, translation) | 3.8 s | 4.7% |
| optimizer and backend | 0.4 s | 0.5% |

For the median file, lowering to LLVM IR is 3.6% of the compile, and 6.5% with the CIR-to-CIR passes.
The lowering row includes the pre-lowering passes (`HoistAllocas`, `FlattenCFG`, `EHABILowering`, `GotoSolver`); `GotoSolver` alone is 0.64 s, 17% of it.
CIRGen's deferred emission and the module verifier run outside any `-ftime-trace` scope, so I attribute the untraced time to CIRGen.

Lopes's [AsiaLLVM 2025 report](https://llvm.github.io/clangir/Files/asiallvm-brunolopes-25-cir-compile-time.pdf) measured the incubator in June 2025.
On the three CTMark programs that compiled, `-fclangir` took 1.9–2.9x the user time at -O0 and 1.1–1.6x at -O2.
For `clamr_cpuonly.cpp` at -O0, classic CodeGen took 15 ms and CIRGen 289 ms, 265 ms of it in `CIRGenModule::applyReplacements`, which the slides call "low hanging fruit".

Some costs are not measured yet.
CIR emits no debug info; in Flang, [#146543](https://github.com/llvm/llvm-project/pull/146543) cut one type's debug-info nodes from 1.75M to 8K.
MLIRContext uniques types, attributes, and locations and [never frees them](https://discourse.llvm.org/t/mlircontext-doesnt-free-denseelementsattributes-and-bloats-memory/3691).
`CIRGenFunction::getLoc` eagerly calls `getPresumedLoc` and creates a uniqued `FileLineColLoc` for each call.

### Maintenance

CIR re-implements classic CodeGen (ABI lowering, builtins, …), and `CodeDuplication.md` accepts "a significant amount of technical debt" kept in sync by hand.
That would be true of any new IR, MLIR or not.

MLIR's API churn is MLIR-specific.
The 2025 keynote names it among the growing pains: "The ClangIR build sometimes gets broken by changes in MLIR or other parts of clang".
I counted at least six CIR fixups since June 2025 caused by MLIR interface changes, e.g. after the `RegionBranchOpInterface` revamp ([#165441](https://github.com/llvm/llvm-project/pull/165441)) and a `MemorySlotInterfaces` rename ([#229068](https://github.com/llvm/llvm-project/pull/229068)).
Premerge testing would turn them into every clang and mlir author's problem.

### Compile-time culture

llvm-compile-time-tracker.com measures every LLVM commit, and regressions get bisected and discussed.
I found no per-commit tracker for MLIR or Flang, and MLIR's in-tree benchmark runner `mlir/utils/mbr` has had no functional change since July 2022.
MLIR compile-time work happens after users hit a wall: the [One-Shot Dialect Conversion driver](https://discourse.llvm.org/t/rfc-a-new-one-shot-dialect-conversion-driver/79083) (dialect conversion was "by far the slowest part"), the LLVM dialect sharding, the MLIRIR precompiled header.
Flang's [#58730](https://github.com/llvm/llvm-project/issues/58730), at 4x gfortran's time and memory, has been open since 2022.
My reading is that MLIR's main users (machine learning compilers that compile a model once and run it many times) tolerate costs that a C++ build, recompiling large translation units on every edit, does not.
Clang's bar has to be enforced on the CIR side.

## Reducing the cost

I keep this table updated as patches land.
Local numbers come from my machine (i7-14700K, 28 threads, Release with assertions, lld), with summed per-file times from `.ninja_log`.

| Patch | Status | Effect |
|---|---|---|
| [#221616](https://github.com/llvm/llvm-project/pull/221616) link only the MLIR dialects CIR uses | landed 2026-09-07 | `ninja clang` wall time +109% to +75% (PragmaTwice, 192 cores) |
| [#221813](https://github.com/llvm/llvm-project/pull/221813) shard the LLVM dialect's generated ops | landed 2026-09-07 | splits `LLVMDialect.cpp.o`, 121 s in aengelke's list |
| [#222576](https://github.com/llvm/llvm-project/pull/222576) precompiled header for MLIRIR | landed 2026-09-10 | c-t-t build time +58% to +47% across it |
| [#227127](https://github.com/llvm/llvm-project/pull/227127) cached symbol tables for comdat lookups | landed 2026-10-02 | lowering of the five worst files from 30–92 s to 8–11 s |
| precompiled header for clangCIR | [#231068](https://github.com/llvm/llvm-project/pull/231068), open | CIRGen's 54 files from 312 s to 85 s summed (including 6 s for the header); user CPU 282 s to 78 s |
| move the ceil/floor div expansion out of MLIRArithTransforms | [#231064](https://github.com/llvm/llvm-project/pull/231064), under review | `clang` links 16 fewer MLIR libraries (90 to 74); 64 fewer object files, 147 s summed (177 s on c-t-t) |
| `MLIROpenACCDialect` without `MLIRGPUDialect` | open | ~38 s; `acc` declares `gpu` as a dependent dialect and uses its types |
| PDL off when MLIR is only a dependency | open | ~21 s |
| shard `CIRDialect.cpp` | open | 58 s locally, the slowest file in the build; its generated `CIROps.cpp.inc` has 102K lines |

## Thought experiment: an MLIR-free ClangIR

What does CIR take from MLIR?

* Dialects: `cir`, `builtin`, `llvm` (~900 references), `acc` (402), `ptr` (74), `omp` (~65), `dlti`.
* ODS: ~230 ops, 164 declarative assembly formats, 53 verifiers, 22 folders/canonicalizers.
* `OpBuilder`, symbol tables, and the region-branch, loop, call, memory-slot, and data-layout interfaces.
* The pass manager with 12 CIR passes, the greedy rewrite driver, and the conversion driver (205 lowering patterns, 125 with hand-written bodies).
* The `llvm` dialect and `ModuleTranslation`.

Not used in clang's pipeline: MLIR's mem2reg and SROA (CIR implements the memory-slot interfaces, but only `cir-opt` tests run `-mem2reg`), CSE, the inliner, the dataflow framework, and the loop dialects.
The incubator's ThroughMLIR path (to `affine`/`scf`) never went upstream; efriedma [noted](https://discourse.llvm.org/t/rfc-upstreaming-clangir/76587/43) that "translating from ClangIR to the MLIR's loop-based dialects is probably more complicated than ClangIR itself."
So for C and C++, `cir` is the only dialect between the AST and the `llvm` dialect, and the `llvm` dialect is a second copy of the module in LLVM IR's terms, built only to be translated.
MLIR cannot skip it: `ModuleTranslation` finds functions and globals only as `llvm` dialect ops, its type translation rejects CIR types, and the OpenMP translation expects `llvm` dialect bodies, so a direct path would be a second `ModuleTranslation` written for CIR.

CIRGen would port almost unchanged.
Only 8.6% of its 66K lines mention `mlir::`, mostly `Value`, `Location`, `Type`, `OpBuilder`, `Block`, and `Region`.
An API-compatible core would keep the ~1,300 `XOp::create` calls, which depend only on generated signatures.
The `.td` files are TableGen data, but they carry 818 embedded C++ blocks (521 `mlir::` references) and declare MLIR interfaces, so they are reusable only with a source-compatible core.

The real cost is everything else: an IR core; an ODS-equivalent emitter including the assembly-format generator (the MLIR emitters it replaces are ~14K lines, plus 5K in `mlir/lib/TableGen`); the interfaces; 15K lines of passes rewritten off MLIR's drivers; and the 868 `-emit-cir` tests reproduced byte for byte or churned.
CIR would also lose MLIR's reviewers; Lauko was [nominated](https://github.com/llvm/llvm-project/pull/147365) as maintainer for "overseeing the quality of CIR dialect (the MLIR part)".
The gains: most of MLIR's ~890 s of build work, no conversion step or extra IR copy, `SourceLocation` as the location type, memory freed with the module, and no MLIR API churn.

OpenACC forces one of three choices: drop it, duplicate Flang's `acc` pipeline, or keep MLIR as an optional island reached through LLVM IR metadata.
Only Keane's one fallback sentence supports the third; the route (metadata, then `ModuleImport`, then `acc` ops, then the shared passes) is my extrapolation, and I found no work on it.

So an MLIR-free CIR is technically plausible, but it removes the property the funders want, and I see nobody who would fund it.
Its value is as a lens.
It separates what only leaving MLIR fixes (immortal context uniquing, the volume of ODS-generated code, the core library closure) from what can be fixed inside MLIR.

## Conclusion

Clang needs a better IR than the AST and the CFG, and CIR is the one that got funded.
Who funds Clang has changed, and that explains much of this post.
Google and Apple wrote 84% of Clang commits in 2010 and 16% in 2026.
NVIDIA's, AMD's, Arm's, and Microsoft's Clang commits are mostly specific to a language mode or target, NVIDIA's mostly CIR, while the shared core is written mostly by individuals, Google, Apple, and Red Hat; [Who pays for Clang](/blog/who-pays-for-clang) has the numbers.

The new funders build GPU and accelerator compilers, and their other compilers (Flang, Triton, IREE, their own stacks) are built on MLIR.
For them, a Clang that emits an MLIR dialect is the point, not an implementation detail.
For the C++ programmers who were Clang's main constituency, CIR has to earn its place by being better at what the AST, the CFG, and classic CodeGen do today: diagnostics, safety analyses, and optimizations that need C++ semantics, without making the compiler slower to build or run.
Lifetime safety went to the CFG in 2025 rather than wait for CIR, which shows the bar is real.

The decision also divides the people who ship and maintain Clang.
Chrome's toolchain team [expects](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/42) to opt out: "If this lands, we'd have to disable the flag downstream."
Red Hat's LLVM maintainer [warned](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/30) that "distros may still decide to disable it" and [measured](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/76) `libclang-cpp.so` growing from 76 MB to 141 MB.
Android's LLVM distribution [plans](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/93) to configure it off, since it doesn't "want to pay any costs associated with this".
The compile-time tracker now [builds without CIR](https://discourse.llvm.org/t/rfc-enable-clangir-build-by-default/91730/103).
If large distributors opt out, building CIR by default makes every contributor pay while many users never get a clang that can run `-fclangir`, and whether Clang has its second code generator depends on who built the compiler.

The question was never really whether CIR should use MLIR.
It is whether anyone is funded to keep the cheap path cheap.
Today the measurable costs fall on Clang contributors and users, most heavily on those who build on their own machines, while the benefits of MLIR accrue to the organizations funding CIR.
Tracking `-fclangir` on c-t-t, keeping `-DCLANG_ENABLE_CIR=Off` working, and landing the cheap build fixes in [Reducing the cost](#reducing-the-cost) are how the people paying the cost get something back.
