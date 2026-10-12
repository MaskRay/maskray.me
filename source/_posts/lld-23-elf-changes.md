---
layout: post
title: lld 23 ELF changes
date: 2026-09-12
tags: [linker,llvm,release]
---

LLVM 23.1 has been released.
As usual, I maintain lld/ELF and as volunteer work have added some notes to <https://github.com/llvm/llvm-project/blob/release/23.x/lld/docs/ReleaseNotes.rst>.

Like last time, I used Claude Code to summarize `git log llvmorg-23-init..origin/release/23.x -- lld/ELF`, excluding changes cherry-picked into 22.x (`git rev-list llvmorg-23-init..llvmorg-22.1.8 -- lld`), and then edited the draft.

This was a busy cycle: 141 commits landed in lld/ELF between the branch point (2026-01-13) and 23.1.0-rc1 (2026-07-16), compared with 72 in the 22 cycle.
Much of the increase is performance work, which I described in [Recent lld/ELF performance improvements](/blog/2026-04-12-recent-lld-elf-performance-improvements).
lld 23 is the first release that ships all of it.

<!-- more -->

* Input file loading is now parallelized, meaningfully reducing link time for large links.
  ([#191690](https://github.com/llvm/llvm-project/pull/191690))
* `--gc-sections` mark phase is now parallelized.
  ([#189321](https://github.com/llvm/llvm-project/pull/189321))
* Relocation scanning was rewritten as target-specific scanners for all targets with shared library support, devirtualizing the hot relocation-classification path.
* Added `--bp-compression-sort-section=<glob>[=<layout_priority>[=<match_priority>]]`, replacing the old coarse `--bp-compression-sort` modes with a way to split input sections into multiple compression groups, run balanced partitioning independently per group, and leave out sections that are poor candidates for BP.
  ([#185661](https://github.com/llvm/llvm-project/pull/185661))
* Added `-z memtag-{mode,heap,stack}` as generic replacements for the Android-specific `--android-memtag-*` flags; `--android-memtag-note` keeps the Android-specific memtag note opt-in.
  ([#188205](https://github.com/llvm/llvm-project/pull/188205))
* Unused space in executable output sections is now filled with trap instructions, primarily for `-z separate-code` mode.
  ([#176845](https://github.com/llvm/llvm-project/pull/176845))
* `.eh_frame_hdr` now supports the `DW_EH_PE_sdata8` encoding, auto-upgrading from `sdata4` when a table entry or the frame pointer exceeds the 32-bit range, instead of erroring out for large executables.
  ([#179089](https://github.com/llvm/llvm-project/pull/179089))
* `vna_flags` is now set to `VER_FLG_WEAK` when all undefined references to a version are weak, allowing glibc's dynamic loader to warn instead of error when the version is missing at runtime.
  ([#176673](https://github.com/llvm/llvm-project/pull/176673))
* `.ltext.*` input sections are now merged into a single `.ltext` output section, matching the existing `.ldata.*`/`.lrodata.*`/`.lbss.*` handling for the large code model with `-ffunction-sections`.
  ([#190305](https://github.com/llvm/llvm-project/pull/190305))
* `.gnu.build.attributes.*` input sections are now concatenated into one output section, matching GNU ld.
  ([#208737](https://github.com/llvm/llvm-project/pull/208737))
* `.tbss` output sections may now use an explicit address expression; previously it was silently overridden to follow the preceding `.tbss`.
  ([#196447](https://github.com/llvm/llvm-project/pull/196447))
* `--discard-locals`/`--discard-all` combined with `-r`/`--emit-relocs` no longer discard local symbols that are referenced only from retained non-`SHF_ALLOC` sections (e.g. `.L` symbols referenced by `.debug_info`), fixing DWARF corruption in the output.
  ([#209035](https://github.com/llvm/llvm-project/pull/209035))
  ([#209042](https://github.com/llvm/llvm-project/pull/209042))
* `--retain-symbols-file` now filters `.symtab` instead of `.dynsym`, matching GNU ld.
  ([#209063](https://github.com/llvm/llvm-project/pull/209063))
* When `-o` is `-`, the output is written to the `lld::outs()` stream instead of the process's stdout, so that library users can capture it.
  ([#209064](https://github.com/llvm/llvm-project/pull/209064))
* `INCLUDE` in linker scripts now fully parses its own content instead of sharing a lexer buffer stack with the includer, fixing spurious acceptance of malformed scripts.
  ([#193427](https://github.com/llvm/llvm-project/pull/193427))
* The `OVERLAY` linker script command now accepts any output-section-command (e.g. symbol assignments), not just input section descriptions.
  ([#203524](https://github.com/llvm/llvm-project/pull/203524))
* Thunks are no longer reused across an `OVERLAY` boundary unless the target output section is guaranteed to be resident at the same time.
  ([#200415](https://github.com/llvm/llvm-project/pull/200415))
* When a `SECTIONS` command interleaves relro and non-relro sections, lld now emits one `PT_GNU_RELRO` segment per contiguous run of relro sections instead of reporting a `not contiguous with other relro sections` error.
  ([#203675](https://github.com/llvm/llvm-project/pull/203675))
* `SHT_NOBITS` sections are now excluded from LMA overlap checks, matching GNU ld and allowing e.g. a startup section to share an LMA with `.bss` in embedded linker scripts.
  ([#196423](https://github.com/llvm/llvm-project/pull/196423))
* LTO: the middle-end no longer emits new references to, or internalizes, symbols defined in bitcode after the extracted-bitcode set has been fixed, preventing undefined symbol references from transforms that run after linking has determined which bitcode files to extract.
  ([#164916](https://github.com/llvm/llvm-project/pull/164916))
* DTLTO: significantly improved the performance of adding backend output files to the link, especially on Windows.
  ([#186366](https://github.com/llvm/llvm-project/pull/186366))
* For AArch64, fixed `.relr.auth.dyn` -> `.rela.dyn` movement to properly adjust `__rela_iplt_start`/`__rela_iplt_end` and size the `.dynamic` section for both tags.
  ([#195649](https://github.com/llvm/llvm-project/pull/195649))
* For AArch64, handle Memtag globals for `R_AARCH64_AUTH_ABS64`.
  ([#173291](https://github.com/llvm/llvm-project/pull/173291))
* For AArch64, fixed TLS GD against non-preemptible dynamic symbols (e.g. `protected` or `-Bsymbolic`) in DSOs, which previously produced an inconsistent GOT entry and spurious preemption.
  ([#207881](https://github.com/llvm/llvm-project/pull/207881))
* For AArch64, `adrp`+`ldr` GOT relaxation is now decided per-symbol, all-or-nothing, avoiding invalid relaxation when a branch target sits between the `adrp` and `ldr` of a pair.
  ([#208396](https://github.com/llvm/llvm-project/pull/208396))
* For AArch64, a redundant local-exec TLS `add` with a zero high-12-bits immediate is now relaxed to a `nop`.
  ([#204286](https://github.com/llvm/llvm-project/pull/204286))
* For AArch64, `-z bti-report=none` and `-z gcs-report=none` now silence the warnings implied by `-z force-bti` and `-z gcs=always`.
  ([#186343](https://github.com/llvm/llvm-project/pull/186343))
* For Hexagon, fixed out-of-range PLT branch thunks and TLS GD PLT entry creation.
  ([#186545](https://github.com/llvm/llvm-project/pull/186545))
  ([#180297](https://github.com/llvm/llvm-project/pull/180297))
* For LoongArch, fixed range checking of `R_LARCH_*_PCADD_HI20` relocations on 64-bit and DTPREL relocations in debug sections.
  ([#183233](https://github.com/llvm/llvm-project/pull/183233))
  ([#199327](https://github.com/llvm/llvm-project/pull/199327))
* For MIPS, fixed the addend for preemptible static TLS.
  ([#150729](https://github.com/llvm/llvm-project/pull/150729))
* For x86-64, CFI jump table relaxation reduces the runtime overhead of indirect calls under Control Flow Integrity by opportunistically moving eligible function bodies into the jump table itself.
  ([#147424](https://github.com/llvm/llvm-project/pull/147424))

Breaking changes:

* The symbol partition feature has been removed.
  lld no longer recognizes `SHT_LLVM_SYMPART` sections, which are now treated as ordinary sections.
  ([#198718](https://github.com/llvm/llvm-project/pull/198718))
  ([#199186](https://github.com/llvm/llvm-project/pull/199186))
* An output section that has an address expression, and is also assigned to a `MEMORY` region, now uses the address expression in preference to the next available location in the region, matching GNU ld.
  ([#197293](https://github.com/llvm/llvm-project/pull/197293))
* The default extension for time trace files is now `.time-trace.json`.
  ([#122207](https://github.com/llvm/llvm-project/pull/122207))

Jessica Clarke (jrtc27) and Peter Smith (smithp35) continued to contribute patches and reviews.
jrtc27 fixed several TLS corner cases (AArch64 TLS GD against non-preemptible symbols, MIPS static TLS addends, SystemZ `R_390_TLS_LDO` in non-`SHF_ALLOC` sections).
Brian Cain fixed a batch of Hexagon issues.

## Performance

The parallelization work from the first half of 2026 all landed before release/23.x was branched in July:

* Parallel input file loading ([#191690](https://github.com/llvm/llvm-project/pull/191690)).
  `createFiles` records a job per non-script input, and `loadFiles` fans the jobs out to worker threads.
  Linker scripts stay on the main thread since `INPUT()`/`GROUP()` recurse into the driver.
* Parallel `--gc-sections` mark ([#189321](https://github.com/llvm/llvm-project/pull/189321)), a level-synchronized BFS with per-shard queues.
  A follow-up replaced `parallel::getThreadIndex` with a shared counter and explicit shards, which improved load balancing ([#208974](https://github.com/llvm/llvm-project/pull/208974)).
* Relocation scanning is parallel for `-z nocombreloc` and PPC64 as well ([#190309](https://github.com/llvm/llvm-project/pull/190309)); only MIPS still scans serially.
* Target-specific relocation scanning: each target's `scanSectionImpl` inlines `getRelExpr` and TLS optimization dispatch.
  x86 ([#178846](https://github.com/llvm/llvm-project/pull/178846)), AArch64 ([#181099](https://github.com/llvm/llvm-project/pull/181099)), ARM ([#182440](https://github.com/llvm/llvm-project/pull/182440)), RISC-V ([#181332](https://github.com/llvm/llvm-project/pull/181332)), LoongArch ([#182236](https://github.com/llvm/llvm-project/pull/182236)), Hexagon ([#181596](https://github.com/llvm/llvm-project/pull/181596)), SystemZ ([#181563](https://github.com/llvm/llvm-project/pull/181563)), PPC32 ([#181517](https://github.com/llvm/llvm-project/pull/181517)), PPC64 ([#181496](https://github.com/llvm/llvm-project/pull/181496)), and SPARCV9 ([#206284](https://github.com/llvm/llvm-project/pull/206284)).
* Faster `getSectionPiece` for merge sections ([#187916](https://github.com/llvm/llvm-project/pull/187916)).
* Parallel `demoteAndCopyLocalSymbols` ([#187970](https://github.com/llvm/llvm-project/pull/187970)) and `demoteSymbolsAndComputeIsPreemptible` ([#207310](https://github.com/llvm/llvm-project/pull/207310)), parallel orphan output section name computation ([#207321](https://github.com/llvm/llvm-project/pull/207321)), and reusing `SHT_GROUP` selection verdicts in `initializeSections` ([#207437](https://github.com/llvm/llvm-project/pull/207437)).
* `llvm/lib/Support/Parallel.cpp` improvements: `parallelFor` spawns only `ThreadCount` workers that claim chunks with an atomic counter, `TaskGroup::spawn` avoids a mutex, the `Executor` virtual base class is gone, and nested `TaskGroup`s use work stealing instead of degenerating to serial execution.

Two of these needed extra care after they landed.
`--reproduce` output became non-deterministic with parallel loading, fixed in [#196773](https://github.com/llvm/llvm-project/pull/196773).
`handleTlsIe` could call the unsynchronized `addRelativeReloc` from concurrent scan tasks (`R_386_TLS_IE` in `-shared` links), fixed in [#208956](https://github.com/llvm/llvm-project/pull/208956); the test was restructured so that a ThreadSanitizer build detects the race.

Here is lld 22.1.8 versus lld 23.1 (release/23.x at 069ef0e7cb36) on an Intel i7-14700K.
Both were built with the same Clang at `-O3 -DNDEBUG` without assertions, as `-fPIE` code in PIE executables with `-Wl,-z,pack-relative-relocs`, and both link against the system mimalloc 3.5 with `-lmimalloc`.

The workloads are from Rui Ueyama: nine x86-64 release-mode links captured with `--reproduce` (Clang 21.1.8, Chromium 145, Firefox 149 `libxul.so`, Blender 5.2, ClickHouse 26.1, Godot 4.6, LibreOffice `libmergedlo.so`, PyTorch `libtorch_cpu.so`, TensorFlow `libtensorflow_cc.so`), linked with `--threads=8`, pinned to the 8 performance cores, transparent huge pages enabled, output written to tmpfs, `hyperfine -w 2 -r 10`.
The debug bundles have 13-19 GiB of input and were I/O bound on this machine, so I left them out.

| Workload | lld 22.1.8 | lld 23.1 | Speedup |
| --- | --- | --- | --- |
| Clang 21.1.8 | 220.2 ms | 189.7 ms | 1.16x |
| Chromium 145 (`--gc-sections --icf=all`) | 4.774 s | 4.325 s | 1.10x |
| Firefox 149 `libxul.so` (`--gc-sections`) | 495.4 ms | 402.6 ms | 1.23x |
| Blender 5.2 | 514.5 ms | 485.1 ms | 1.06x |
| ClickHouse 26.1 (`--gc-sections`) | 1.874 s | 1.570 s | 1.19x |
| Godot 4.6 | 199.6 ms | 166.4 ms | 1.20x |
| LibreOffice `libmergedlo.so` | 412.7 ms | 362.8 ms | 1.14x |
| PyTorch `libtorch_cpu.so` | 272.7 ms | 261.0 ms | 1.04x |
| TensorFlow `libtensorflow_cc.so` (`--gc-sections`) | 5.998 s | 5.776 s | 1.04x |

lld 23.1 is 1.04x to 1.23x as fast as lld 22.1 on these links.
The two slowest links are each dominated by a single phase according to `--time-trace`.
On TensorFlow, `Process symbol versions` takes 4.5 s of the 5.6 s link: the version script has 29 `*pattern*` globs, and each wildcard pattern is matched against every symbol on one thread. (The bottleneck will be parallelized in [LLVM 24](https://github.com/llvm/llvm-project/pull/223208))
On Chromium, `--icf=all` takes 2.7 s of the 4.2 s link.

For the full breakdown, including comparisons with mold and wild, see [Recent lld/ELF performance improvements](/blog/2026-04-12-recent-lld-elf-performance-improvements).

## CFI jump table relaxation

Clang's Control-Flow Integrity (`-fsanitize=cfi-icall` and friends) implements indirect call checks with jump tables.
For every address-taken function `f`, the `LowerTypeTests` pass renames the body to `f.cfi` and creates a jump table entry `f` that is just `jmp f.cfi`.
Functions with the same type signature get consecutive entries, so checking that an indirect call target has the expected type is a range and alignment check on the address.
The address-taken symbol `f` refers to the jump table entry, not the body.

On x86-64 each entry is 8 bytes (a 5-byte `jmp` padded with `int3`):

```asm
.section .text.cfi.jumptable,"ax",@llvm_cfi_jump_table,8
f:
  jmp f.cfi
  int3; int3; int3
g:
  jmp g.cfi
  int3; int3; int3
h:
  jmp h.cfi
  int3; int3; int3
```

Every indirect call pays an extra `jmp`, and the jump tables are clustered together, typically far from the callees, which hurts the i-cache and the iTLB.
Google's internal benchmarking attributed around 30% of the total CFI overhead to this indirection.
lld 21 added `--branch-to-branch` (enabled at `-O2`), which retargets a direct `call f` to `f.cfi` ([#145579](https://github.com/llvm/llvm-project/pull/145579)), but indirect calls still went through the table.

lld 23 adds CFI jump table relaxation ([#147424](https://github.com/llvm/llvm-project/pull/147424)), also controlled by `--branch-to-branch`.
LLVM 22 defined a new section type `SHT_LLVM_CFI_JUMP_TABLE` with `sh_entsize` set to the entry size, and Clang 23 emits CFI jump tables with this type.
`X86_64::relaxCFIJumpTables` walks such sections and eliminates the indirection in two cases:

* If `f.cfi` is at most `sh_entsize` bytes (and no more aligned than that), the function body is moved into the jump table in place of the entry.
  The jump table is split into slices around the moved bodies.
* The last entry's target is handled specially: the jump table is placed immediately before the target function, and the last `jmp` is deleted so that control falls through.
  The function stays where it is, on the assumption that functions of the same type (and their callees) are more likely to share a page with each other than with the jump table's original location.

The eligibility check is conservative: the branch must be a `R_X86_64_PC32`/`R_X86_64_PLT32` to the start of a non-preemptible, non-IFUNC `Defined` symbol, the section may not be over-aligned relative to the entry size, and a function body is only moved once even if multiple jump tables reference it.
The commit message reports a 0.2 to 0.5 percentage point reduction in CFI overhead (10-25% of the overhead) on a large internal Google benchmark.

## Symbol partitions removed

The symbol partition feature (2019, [D60242](https://reviews.llvm.org/D60242) and [D60353](https://reviews.llvm.org/D60353)) allowed an executable or shared object to be split into a main partition and several loadable partitions.
Symbols were assigned to partitions with `SHT_LLVM_SYMPART` sections, `llvm-objcopy --extract-partition` extracted each partition into a separate file, and the loader could `dlopen` a feature partition on demand, sharing the main partition's memory.
It was built for Chrome on Android to lazily load features.

The feature touched a surprising amount of the linker: per-partition `MarkLive` runs, per-partition `.dynamic`/`.dynsym`/`.gnu.hash`/`.eh_frame`/`.ARM.exidx`, cross-partition pull in `InputSection::replace`, thunk compatibility checks, `PartitionIndexSection`, and more.
Chrome hasn't needed it for years, and I am not aware of any other user.
[#198718](https://github.com/llvm/llvm-project/pull/198718) reduced the machinery to a shim, and [#199186](https://github.com/llvm/llvm-project/pull/199186) removed the rest.
`SHT_LLVM_SYMPART` sections are now treated as ordinary sections, and `ctx.mainPart`/`ctx.partitions` are gone from the code base.

Removing an experimental feature that never found a second user simplifies a lot of code paths, particularly synthetic sections, which no longer need to be instantiated per partition.

## Linker script robustness

Several linker script fixes landed this cycle.

`INCLUDE` used to push a buffer onto the lexer's buffer stack, so a construct started in the included file could be closed by the parent.
This accepted malformed scripts (a missing `;` in the include, terminated by the parent's `;`) and computed undefined-behavior spans in `readAssignment`.
[#193427](https://github.com/llvm/llvm-project/pull/193427) makes each `INCLUDE` parse its own content like a call frame; the top-level, `SECTIONS`, output section, and `MEMORY` call sites pass a context-appropriate parser callback.

`OVERLAY` now accepts any output-section-command, e.g. symbol assignments, not just input section descriptions ([#203524](https://github.com/llvm/llvm-project/pull/203524)).
Thunks in an `OVERLAY` are no longer reused by callers in a different output section, since the thunk may not be resident when the caller runs ([#200415](https://github.com/llvm/llvm-project/pull/200415)).
Peter Smith notes that Tightly Coupled Memory regions could be modeled the same way.

When a `SECTIONS` command interleaves relro and non-relro sections, lld used to error with `not contiguous with other relro sections` ([D40359](https://reviews.llvm.org/D40359)).
glibc only honors the first `PT_GNU_RELRO`, but Bionic and FreeBSD rtld protect every `PT_GNU_RELRO` segment, so [#203675](https://github.com/llvm/llvm-project/pull/203675) emits one `PT_GNU_RELRO` per contiguous run of relro sections.
Each relro-to-non-relro transition still starts a fresh `PT_LOAD`.

`SHT_NOBITS` sections are excluded from LMA overlap checks ([#196423](https://github.com/llvm/llvm-project/pull/196423)), matching GNU ld, which lets embedded scripts place a startup section at the same load address as `.bss`.

The `MEMORY` change is worth calling out as a behavior change.
For `s02 . : { ... } > FLASH`, lld evaluated `.` against the memory region's current position instead of the global location counter.
[#197293](https://github.com/llvm/llvm-project/pull/197293) fixes this to match GNU ld.
If you relied on the old behavior, the section address may move.

## GNU ld compatibility

`--retain-symbols-file` was implemented in 2017 using the symbol version machinery (`{local: *; global: listed;}`), which removed unlisted symbols from `.dynsym` and left `.symtab` untouched.
GNU ld does the opposite: it keeps only the listed symbols in `.symtab` and does not touch `.dynsym`.
[#209063](https://github.com/llvm/llvm-project/pull/209063) reimplements the option on top of the `--discard-{locals,all}` mechanism.
Use `--export-dynamic-symbol` or a version script to control `.dynsym`.

While reworking `--discard-*`, I found that `-r`/`--emit-relocs` with `--discard-locals` could discard `.L` symbols referenced only from retained non-`SHF_ALLOC` sections (e.g. `.debug_str_offsets` referenced by `.debug_info`), rewriting the relocations to reference the null symbol and corrupting DWARF ([#209042](https://github.com/llvm/llvm-project/pull/209042)).
This is common on RISC-V, where the assembler prefers symbol-relative relocations, but reachable on any target with `--reloc-section-sym=none`.

`.gnu.build.attributes.*` sections (emitted by annobin) are now concatenated into `.gnu.build.attributes` ([#208737](https://github.com/llvm/llvm-project/pull/208737)).
GNU ld has done this since 2018; without it, an output with tens of thousands of sections broke tools like `file`.

`vna_flags` is set to `VER_FLG_WEAK` when all undefined references to a version are weak ([#176673](https://github.com/llvm/llvm-project/pull/176673)).
glibc since 2.30 tolerates a missing versioned symbol when the shared object defines the version; with this flag it also tolerates a completely missing version, printing `weak version 'v1' not found` instead of failing.
This is useful for optional dependencies.

## AI-assisted patches

The commit count is only part of the story.
Excluding my own and the backport bot's, 147 pull requests carried the `lld:ELF` label between the two branch points, compared with 99 in the 22 cycle; 82 people opened one, up from 49, and 49 of them had not sent an lld/ELF patch before.
Many of the new contributors are unfamiliar with lld and use coding agents, and a large fraction of these patches were of poor quality.

Some marks are unmistakable: "Initial plan", multiple headings, agent trailers, a `copilot/...` branch, and a first CI run that fails on a missing `#include` because the patch was never compiled locally.

Most are subtler, and the patches fall into a few kinds.

**Wrong root cause**

The agent finds the place where the symptom can be suppressed and adds a special case there, complete with a plausible test, while the actual bug lives elsewhere.
Explaining why the fix belongs somewhere else takes longer than writing the right fix, so several times this cycle I closed the pull request, fixed the issue myself, and (depending on the concrete scenario) credited the original PR link/author in the commit message.

**Real issue, fine fix, but super verbose description or bad tests**

... the commit message is four times longer than it needs to be, the comments restate the code beneath them, and a new test file is added where an existing test could have been extended by two lines.

**Feature-sized patch that skipped a design discussion**

The tool makes it easy to produce a thousand lines that look like lld code, but the hard part is the design discussion, and that part can't be delegated.

---

I don't object to the tools; I used Claude Code to draft this post.
The problem is what they do to the social system around code.
The cost shifts from the contributor to the maintainer.
The effort it took to write a patch used to be a commitment signal; reviewing generated code that its author only half understands removes the fun part and keeps the exhausting part.

Unless you are the BDFL, the social norm is still "mentor, don't rewrite their patch".
That norm made sense when a patch represented hours of work.
It doesn't now, and it hasn't adapted.

For a component maintained as volunteer work, maintainer time is the bottleneck, and LLVM's [AI tool use policy](https://llvm.org/docs/AIToolPolicy.html) calls such patches extractive contributions for a reason.

If you are new to LLVM, start with unaided or lightly aided patches on small, well-scoped issues: extend an existing test rather than adding a file, run tests before posting, and keep the commit message proportional to the change.
A track record on small work is the fastest route to using agents productively on larger changes later.
In addition, be prepared that your work may be taken over by a maintainer.

---

Link: [lld 22 ELF changes](/blog/lld-22-elf-changes)
