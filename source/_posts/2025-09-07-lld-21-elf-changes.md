---
layout: post
title: lld 21 ELF changes
author: MaskRay
tags: [linker,llvm]
---

LLVM 21.1 have been released.
As usual, I maintain lld/ELF and have added some notes to <https://github.com/llvm/llvm-project/blob/release/21.x/lld/docs/ReleaseNotes.rst>.
I've meticulously reviewed nearly all the patches that are not authored by me.
I'll delve into some of the key changes.

<!-- more -->

* Added `-z dynamic-undefined-weak` to make undefined weak symbols dynamic
  when the dynamic symbol table is present.
  ([#143831](https://github.com/llvm/llvm-project/pull/143831))
* For `-z undefs` (default for `-shared`), relocations referencing undefined
  strong symbols now behave like relocations referencing undefined weak symbols.
* `--why-live=<glob>` prints for each symbol matching `<glob>` a chain of
  items that kept it live during garbage collection. This is inspired by the
  Mach-O LLD feature of the same name.
* `--thinlto-distributor=` and `--thinlto-remote-compiler=` options are
  added to support Integrated Distributed ThinLTO.
  ([#142757](https://github.com/llvm/llvm-project/pull/142757))
* Linker script `OVERLAY` descriptions now support virtual memory regions
  (e.g. `>region`) and `NOCROSSREFS`.
* When the last `PT_LOAD` segment is executable and includes BSS sections,
  its `p_memsz` member is now correct.
  ([#139207](https://github.com/llvm/llvm-project/pull/139207))
* Spurious `ASSERT` errors before the layout converges are now fixed.
* For ARM and AArch64, `--xosegment` and `--no-xosegment` control whether
  to place executable-only and readable-executable sections in the same
  segment. The default option is `--no-xosegment`.
  ([#132412](https://github.com/llvm/llvm-project/pull/132412))
* For AArch64, added support for the `SHF_AARCH64_PURECODE` section flag,
  which indicates that the section only contains program code and no data.
  An output section will only have this flag set if all input sections also
  have it set. ([#125689](https://github.com/llvm/llvm-project/pull/125689),
  [#134798](https://github.com/llvm/llvm-project/pull/134798))
* For AArch64 and ARM, added `-zexecute-only-report`, which checks for
  missing `SHF_AARCH64_PURECODE` and `SHF_ARM_PURECODE` section flags
  on executable sections.
  ([#128883](https://github.com/llvm/llvm-project/pull/128883))
* For AArch64, `-z nopac-plt` has been added.
* For AArch64 and X86_64, added `--branch-to-branch`, which rewrites branches
  that point to another branch instruction to instead branch directly to the
  target of the second instruction. Enabled by default at `-O2`.
* For AArch64, added support for `-zgcs-report-dynamic`, enabling checks for
  GNU GCS Attribute Flags in Dynamic Objects when GCS is enabled. Inherits value
  from `-zgcs-report` (capped at `warning` level) unless user-defined,
  ensuring compatibility with GNU ld linker.
* The default Hexagon architecture version in ELF object files produced by
  lld is changed to v68. This change is only effective when the version is
  not provided in the command line by the user and cannot be inferred from
  inputs.
* For LoongArch, the initial-exec to local-exec TLS optimization has been implemented.
* For LoongArch, several relaxation optimizations are supported, including relaxation for
  `R_LARCH_PCALA_HI20/LO12` and `R_LARCH_GOT_PC_HI20/LO12` relocations, instruction
  relaxation for `R_LARCH_CALL36`, TLS local-exec (`LE`)/global dynamic (`GD`)/
  local dynamic (`LD`) model relaxation, and TLSDESC code sequence relaxation.
* For RISCV, an oscillation bug due to call relaxation is now fixed.
  ([#142899](https://github.com/llvm/llvm-project/pull/142899))
* For x86-64, the `.ltext` section is now placed before `.rodata`.

---

Link: [lld 20 ELF changes](/blog/2025-02-02-lld-20-elf-changes)
