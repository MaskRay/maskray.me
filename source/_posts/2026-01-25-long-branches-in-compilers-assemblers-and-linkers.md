---
layout: post
title: Long branches in compilers, assemblers, and linkers
author: MaskRay
tags: [assembler,binutils,llvm,linker]
---

Branch instructions on most architectures use PC-relative addressing with a limited range.
When the target is too far away, the branch becomes "out of range" and requires special handling.

Consider a large binary where `main()` at address 0x10000 calls `foo()` at address 0x8010000-over 128MiB away.
On AArch64, the `bl` instruction can only reach ±128MiB, so this call cannot be encoded directly.
Without proper handling, the linker would fail with an error like "relocation out of range."
The toolchain must handle this transparently to produce correct executables.

This article explores how compilers, assemblers, and linkers work together to solve the long branch problem.

* Compiler (IR to assembly): Handles branches within a function that exceed the range of conditional branch instructions
* Assembler (assembly to relocatable file): Handles branches within a section where the distance is known at assembly time
* Linker: Handles cross-section and cross-object branches discovered during final layout

<!-- more -->

## Branch range limitations

Different architectures have different branch range limitations.
Here's a quick comparison of unconditional / conditional branch ranges:

| Architecture | Cond | Uncond | Call | Notes |
|-----|------|------|-----|-------------------|
| AArch64 | ±1MiB | ±128MiB | ±128MiB | Thunks |
| AArch32 (A32) | ±32MiB | ±32MiB | ±32MiB | Thunks, interworking |
| AArch32 (T32) | ±1MiB | ±16MiB | ±16MiB | Thunks, interworking |
| LoongArch | ±128KiB | ±128MiB | ±128MiB | Linker relaxation |
| M68k (68020+) | ±2GiB | ±2GiB | ±2GiB | Assembler picks size |
| MIPS (pre-R6) | ±128KiB | ±128KiB (`b offset`) | ±128KiB (`bal offset`) | In `-fno-pic` code, pseudo-absolute `j`/`jal` can be used for a 256MiB region. |
| MIPS R6 | ±128KiB | ±128MiB | ±128MiB | |
| PowerPC64 | ±32KiB | ±32MiB | ±32MiB | Thunks |
| RISC-V | ±4KiB | ±1MiB | ±1MiB | Linker relaxation |
| SPARC | ±1MiB | ±8MiB | ±2GiB | No thunks needed |
| SuperH | ±256B | ±4KiB | ±4KiB | Use register-indirect if needed
| x86-64 | ±2GiB | ±2GiB | ±2GiB | Large code model changes call sequence |
| Xtensa | ±2KiB | ±128KiB | ±512KiB | Linker relaxation |
| z/Architecture | ±64KiB | ±4GiB | ±4GiB | No thunks needed |

The following subsections provide detailed per-architecture information, including relocation types relevant for linker implementation.

### AArch32

In A32 state:

* Branch (`b`/`b<cond>`), conditional branch and link (`bl<cond>`) (`R_ARM_JUMP24`): ±32MiB
* Unconditional branch and link (`bl`/`blx`, `R_ARM_CALL`): ±32MiB

Note: `R_ARM_CALL` is for unconditional `bl`/`blx` which can be relaxed to BLX inline; `R_ARM_JUMP24` is for branches which require a veneer for interworking.

In T32 state (Thumb state pre-ARMv8):

* Conditional branch (`b<cond>`, `R_ARM_THM_JUMP8`): ±256 bytes
* Short unconditional branch (`b`, `R_ARM_THM_JUMP11`): ±2KiB
* ARMv5T branch and link (`bl`/`blx`, `R_ARM_THM_CALL`): ±4MiB
* ARMv6T2 wide conditional branch (`b<cond>.w`, `R_ARM_THM_JUMP19`): ±1MiB
* ARMv6T2 wide branch (`b.w`, `R_ARM_THM_JUMP24`): ±16MiB
* ARMv6T2 wide branch and link (`bl`/`blx`, `R_ARM_THM_CALL`): ±16MiB. `R_ARM_THM_CALL` can be relaxed to BLX.

### AArch64

* Test bit and branch (`tbz`/`tbnz`, `R_AARCH64_TSTBR14`): ±32KiB
* Compare and branch (`cbz`/`cbnz`, `R_AARCH64_CONDBR19`): ±1MiB
* Conditional branches (`b.<cond>`, `R_AARCH64_CONDBR19`): ±1MiB
* Unconditional branches (`b`/`bl`, `R_AARCH64_JUMP26`/`R_AARCH64_CALL26`): ±128MiB

The compiler's `BranchRelaxation` pass handles out-of-range conditional branches by inverting the condition and inserting an unconditional branch.
The AArch64 assembler does not perform branch relaxation; out-of-range branches produce linker errors if not handled by the compiler.

### LoongArch

* Conditional branches (`beq`/`bne`/`blt`/`bge`/`bltu`/`bgeu`, `R_LARCH_B16`): ±128KiB (18-bit signed)
* Compare-to-zero branches (`beqz`/`bnez`, `R_LARCH_B21`): ±4MiB (23-bit signed)
* Unconditional branch/call (`b`/`bl`, `R_LARCH_B26`): ±128MiB (28-bit signed)
* Medium range call (`pcaddu12i`+`jirl`, `R_LARCH_CALL30`): ±2GiB
* Long range call (`pcaddu18i`+`jirl`, `R_LARCH_CALL36`): ±128GiB

### M68k

* Short branch (`Bcc.B`/`BRA.B`/`BSR.B`): ±128 bytes (8-bit displacement)
* Word branch (`Bcc.W`/`BRA.W`/`BSR.W`): ±32KiB (16-bit displacement)
* Long branch (`Bcc.L`/`BRA.L`/`BSR.L`, 68020+): ±2GiB (32-bit displacement)

GNU Assembler provides [pseudo opcodes](https://sourceware.org/binutils/docs/as/M68K_002dBranch.html) (`jbsr`, `jra`, `jXX`) that "automatically expand to the shortest instruction capable of reaching the target".
For example, `jeq .L0` emits one of `beq.b`, `beq.w`, and `beq.l`  depending on the displacement.

With the long forms available on 68020 and later, M68k doesn't need linker range extension thunks.

### MIPS

* Conditional branches (`beq`/`bne`/`bgez`/`bltz`/etc, `R_MIPS_PC16`): ±128KiB
* PC-relative jump (`b offset` (`bgez $zero, offset`)): ±128KiB
* PC-relative call (`bal offset` (`bgezal $zero, offset`)): ±128KiB
* Pseudo-absolute jump/call (`j`/`jal`, `R_MIPS_26`): branch within the current 256MiB region, only suitable for `-fno-pic` code. Deprecated in R6 in favor of `bc`/`balc`

16-bit instructions removed in Release 6:

* Conditional branch (`beqz16`, `R_MICROMIPS_PC7_S1`): ±128 bytes
* Unconditional branch (`b16`, `R_MICROMIPS_PC10_S1`): ±1KiB

MIPS Release 6:

* Unconditional branch, compact (`bc16`, unclear toolchain implementation): ±1KiB
* Compare and branch, compact (`beqc`/`bnec`/`bltc`/`bgec`/etc, `R_MIPS_PC16`): ±128KiB
* Compare register to zero and branch, compact (`beqzc`/`bnezc`/etc, `R_MIPS_PC21_S2`): ±4MiB
* Branch (and link), compact (`bc`/`balc`, `R_MIPS_PC26_S2`): ±128MiB

Compiler long branch handling: Both GCC (`mips_output_conditional_branch`) and LLVM (`MipsBranchExpansion`) handle out-of-range conditional branches by inverting the condition and inserting an unconditional jump:

LLVM's `MipsBranchExpansion` pass handles out-of-range branches.

lld implements LA25 thunks for MIPS PIC/non-PIC interoperability, but not range extension thunks.
GNU ld also does not implement range extension thunks for MIPS.

GCC's mips port ported [added `-mlong-calls`](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d1399bd0ff3893bb9ebea7b977c7f3ec91b728b0) in 1993-03.
In `-mno-abicalls` mode, GCC's `-mlong-calls` option ([added in 1993](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d1399bd0ff3893bb9ebea7b977c7f3ec91b728b0))
generates indirect call sequences that can reach any address.

### PowerPC

* Conditional branch (`bc`/`bcl`, `R_PPC64_REL14`): ±32KiB
* Unconditional branch (`b`/`bl`, `R_PPC64_REL24`/`R_PPC64_REL24_NOTOC`): ±32MiB

GCC-generated code relies on linker thunks. However, the legacy `-mlongcall` can be used to generate long code sequences.

### RISC-V

* Compressed `c.beqz`: ±256 bytes
* Compressed `c.jal`: ±2KiB
* `jalr` (I-type immediate): ±2KiB
* Conditional branches (`beq`/`bne`/`blt`/`bge`/`bltu`/`bgeu`, B-type immediate): ±4KiB
* `jal` (J-type immediate, `PseudoBR`): ±1MiB (notably smaller than other RISC architectures: AArch64 ±128MiB, PowerPC64 ±32MiB, LoongArch ±128MiB)
* `PseudoJump` (using `auipc` + `jalr`): ±2GiB
* `beqi`/`bnei` (Zibi extension, 5-bit compare immediate (1 to 31 and -1)): ±4KiB

Qualcomm uC Branch Immediate extension (Xqcibi):

* `qc.beqi`/`qc.bnei`/`qc.blti`/`qc.bgei`/`qc.bltui`/`qc.bgeui` (32-bit, 5-bit compare immediate): ±4KiB
* `qc.e.beqi`/`qc.e.bnei`/`qc.e.blti`/`qc.e.bgei`/`qc.e.bltui`/`qc.e.bgeui` (48-bit, 16-bit compare immediate): ±4KiB

Qualcomm uC Long Branch extension (Xqcilb):

* `qc.e.j`/`qc.e.jal` (48-bit, `R_RISCV_VENDOR(QUALCOMM)+R_RISCV_QC_E_CALL_PLT`): ±2GiB

For function calls:

* The [Go compiler](https://go-review.googlesource.com/c/go/+/345051) emits a single `jal` for calls and relies on its linker to generate trampolines when the target is out of range.
* In contrast, GCC and Clang emit `auipc`+`jalr` and rely on linker relaxation to shrink the sequence when possible.

The `jal` range (±1MiB) is notably smaller than other RISC architectures (AArch64 ±128MiB, PowerPC64 ±32MiB, LoongArch ±128MiB).
This limits the effectiveness of linker relaxation ("start large and shrink"), and leads to frequent trampolines when the compiler optimistically emits `jal` ("start small and grow").

### SPARC

* Compare and branch (`cxbe`, `R_SPARC_5`): ±64 bytes
* Conditional branch (`bcc`, `R_SPARC_WDISP19`): ±1MiB
* Unconditional branch (`b`, `R_SPARC_WDISP22`): ±8MiB
* `call` (`R_SPARC_WDISP30`/`R_SPARC_WPLT30`): ±2GiB

With ±2GiB range for `call`, SPARC doesn't need range extension thunks in practice.

### SuperH

SuperH uses fixed-width 16-bit instructions, which limits branch ranges.

* Conditional branch (`bf`/`bt`): ±256 bytes (8-bit displacement)
* Unconditional branch (`bra`): ±4KiB (12-bit displacement)
* Branch to subroutine (`bsr`): ±4KiB (12-bit displacement)

For longer distances, register-indirect branches (`braf`/`bsrf`) are used. The compiler inverts conditions and emits these when targets exceed the short ranges.

SuperH is supported by GCC and binutils, but not by LLVM.

### Xtensa

Xtensa uses variable-length instructions: 16-bit (narrow, `.n` suffix) and 24-bit (standard).

* Narrow conditional branch (`beqz.n`/`bnez.n`, 16-bit): -28 to +35 bytes (6-bit signed + 4)
* Conditional branch (compare two registers) (`beq`/`bne`/`blt`/`bge`/etc, 24-bit): ±256 bytes
* Conditional branch (compare with zero) (`beqz`/`bnez`/`bltz`/`bgez`, 24-bit): ±2KiB
* Unconditional jump (`j`, 24-bit): ±128KiB
* Call (`call0`/`call4`/`call8`/`call12`, 24-bit): ±512KiB

The assembler performs branch relaxation: when a conditional branch target is too far, it inverts the condition and inserts a `j` instruction.

Per <https://www.sourceware.org/binutils/docs/as/Xtensa-Call-Relaxation.html>, for calls, GNU Assembler pessimistically generates indirect sequences (`l32r`+`callx8`) when the target distance is unknown.
GNU ld then performs linker relaxation.

### x86-64

* Short conditional jump (`Jcc rel8`): -128 to +127 bytes
* Short unconditional jump (`JMP rel8`): -128 to +127 bytes
* Near conditional jump (`Jcc rel32`): ±2GiB
* Near unconditional jump (`JMP rel32`): ±2GiB

With a ±2GiB range for near jumps, x86-64 rarely encounters out-of-range branches in practice.
That said, Google and Meta Platforms deploy mostly statically linked executables on x86-64 production servers and have run into the huge executable problem for certain configurations.

### z/Architecture

* Short conditional branch (`BRC`, `R_390_PC16DBL`): ±64KiB (16-bit halfword displacement)
* Long conditional branch (`BRCL`, `R_390_PC32DBL`): ±4GiB (32-bit halfword displacement)
* Short call (`BRAS`, `R_390_PC16DBL`): ±64KiB
* Long call (`BRASL`, `R_390_PC32DBL`): ±4GiB

With ±4GiB range for long forms, z/Architecture doesn't need linker range extension thunks.
LLVM's `SystemZLongBranch` pass relaxes short branches (`BRC`/`BRAS`) to long forms (`BRCL`/`BRASL`) when targets are out of range.

## Compiler: branch range handling

Conditional branch instructions usually have shorter ranges than unconditional ones, making them less suitable for linker thunks (as we will explore later).
Compilers typically keep conditional branch targets within the same section, allowing the compiler to handle out-of-range cases via branch relaxation.

Within a function, conditional branches may still go out of range.
The compiler measures branch distances and relaxes out-of-range branches by inverting the condition and inserting an unconditional branch:

```asm
# Before relaxation (out of range)
beq .Lfar_target       # ±4KiB range on RISC-V

# After relaxation
bne .Lskip             # Inverted condition, short range
j .Lfar_target         # Unconditional jump, ±1MiB range
.Lskip:
```

Some architectures have conditional branch instructions that compare with an immediate, with even shorter ranges due to encoding additional immediates.
For example, AArch64's `cbz`/`cbnz` (compare and branch if zero/non-zero) and `tbz`/`tbnz` (test bit and branch) have only ±32KiB range.
RISC-V Zibi `beqi`/`bnei` have ±4KiB range.
The compiler handles these in a similar way:

```asm
// Before relaxation (cbz has ±32KiB range)
  cbz w0, far

// After relaxation
  cbnz w0, .Lskip       // Inverted condition
  b far                 // Unconditional branch, ±128MiB range
.Lskip:
```

An Intel employee contributed <https://reviews.llvm.org/D41634> (in 2017) when inversion of a branch condintion is impossible.
This is for an out-of-tree backend. As of Jan 2026 there is no in-tree test for this code path.

In LLVM, this is handled by the `BranchRelaxation` pass, which runs just before `AsmPrinter`.
Different backends have their own implementations:

* `BranchRelaxation`: AArch64, AMDGPU, AVR, RISC-V
* `HexagonBranchRelaxation`: Hexagon
* `PPCBranchSelector`: PowerPC
* `SystemZLongBranch`: SystemZ
* `MipsBranchExpansion`: MIPS
* `MSP430BSel`: MSP430

The generic `BranchRelaxation` pass computes block sizes and offsets, then iterates until all branches are in range.
For conditional branches, it tries to invert the condition and insert an unconditional branch.
For unconditional branches that are still out of range, it calls `TargetInstrInfo::insertIndirectBranch` to emit an indirect jump sequence (e.g., `adrp`+`add`+`br` on AArch64) or a long jump sequence (e.g., pseudo `jump` on RISC-V).

Note: The size estimates may be inaccurate due to inline assembly.
LLVM uses heuristics to estimate inline assembly sizes, but for certain assembly constructs the size is not precisely known at compile time.

Unconditional branches and calls can target different sections since they have larger ranges.
If the target is out of reach, the linker can insert thunks to extend the range.

For x86-64, the large code model uses multiple instructions for calls and jumps to support text sections larger than 2GiB (see [Relocation overflow and code models: x86-64 large code model](/blog/2023-05-14-relocation-overflow-and-code-models#x86-64-large-code-model)).
This is a pessimization if the callee ends up being within reach.
Google and Meta Platforms have interest in allowing range extension thunks as a replacement for the multiple instructions.

## Assembler: instruction relaxation

The assembler converts assembly to machine code.
When the target of a branch is within the same section and the distance is known at assembly time, the assembler can select the appropriate encoding.
This is distinct from linker thunks, which handle cross-section or cross-object references where distances aren't known until link time.

Assembler instruction relaxation handles two cases (see [Clang -O0 output: branch displacement and size increase](/blog/2024-04-27-clang-o0-output-branch-displacement-and-size-increase) for examples):

* **Span-dependent instructions**: Select an appropriate encoding based on displacement.
  + On x86, a short jump (`jmp rel8`) can be relaxed to a near jump (`jmp rel32`) when the target is far.
  + On RISC-V, `beqz` may be assembled to the 2-byte `c.beqz` when the displacement fits within ±256 bytes.
* **Conditional branch transform**: Invert the condition and insert an unconditional branch. On RISC-V, a `blt` might be relaxed to `bge` plus an unconditional branch.

The assembler uses an iterative layout algorithm that alternates between fragment offset assignment and relaxation until all fragments become legalized.
See [Integrated assembler improvements in LLVM 19](/blog/2024-06-30-integrated-assembler-improvements-in-llvm-19) for implementation details.

## Linker: range extension thunks

When the linker resolves relocations, it may discover that a branch target is out of range.
At this point, the instruction encoding is fixed, so the linker cannot simply change the instruction.
Instead, it generates **range extension thunks** (also called veneers, branch stubs, or trampolines).

A thunk is a small piece of linker-generated code that can reach the actual target using a longer sequence of instructions.
The original branch is redirected to the thunk, which then jumps to the real destination.

Range extension thunks are one type of linker-generated thunk. Other types include:

* **ARM interworking veneers**: Switch between ARM and Thumb instruction sets (see [Linker notes on AArch32](/blog/2023-04-23-linker-notes-on-aarch32))
* **MIPS LA25 thunks**: Enable PIC and non-PIC code interoperability (see [Toolchain notes on MIPS](/blog/2023-09-04-toolchain-notes-on-mips))
* **PowerPC64 TOC/NOTOC thunks**: Handle calls between functions using different TOC pointer conventions (see [Linker notes on Power ISA](/blog/2023-02-26-linker-notes-on-power-isa))

### Short range vs long range thunks

A **short range thunk** (see [lld/ELF's AArch64 implementation](https://reviews.llvm.org/D148701)) contains just a single branch instruction.
Since it uses a branch, its reach is also limited by the branch range—it can only extend coverage by one branch distance.
For targets further away, multiple short range thunks can be chained, or a long range thunk with address computation must be used.

Long range thunks use indirection and can jump to (practically) arbitrary locations.

```asm
// Short range thunk: single branch, 4 bytes
__AArch64AbsLongThunk_dst:
  b dst                         // ±128MiB range

// Long range thunk: address computation, 12 bytes
__AArch64ADRPThunk_dst:
  adrp x16, dst                 // Load page address (±4GiB range)
  add x16, x16, :lo12:dst       // Add page offset
  br x16                        // Indirect branch
```

### Thunk examples

**AArch32 (PIC)** (see [Linker notes on AArch32](/blog/2023-04-23-linker-notes-on-aarch32)):
```asm
__ARMV7PILongThunk_dst:
  movw ip, :lower16:(dst - .)   ; ip = intra-procedure-call scratch register
  movt ip, :upper16:(dst - .)
  add ip, ip, pc
  bx ip
```

**PowerPC64 ELFv2** (see [Linker notes on Power ISA](/blog/2023-02-26-linker-notes-on-power-isa)):
```asm
__long_branch_dst:
  addis 12, 2, .branch_lt@ha    # Load high bits from branch lookup table
  ld 12, .branch_lt@l(12)       # Load target address
  mtctr 12                      # Move to count register
  bctr                          # Branch to count register
```

### Thunk impact on debugging and profiling

Thunks are transparent at the source level but visible in low-level tools:

* **Stack traces**: May show thunk symbols (e.g., `__AArch64ADRPThunk_foo`) between caller and callee
* **Profilers**: Samples may attribute time to thunk code; some profilers aggregate thunk time with the target function
* **Disassembly**: `objdump` or `llvm-objdump` will show thunk sections interspersed with regular code
* **Code size**: Each thunk adds bytes; large binaries may have thousands of thunks

### lld/ELF's thunk creation algorithm

lld/ELF uses a multi-pass algorithm in `finalizeAddressDependentContent`:

```cpp
assignAddresses();
for (pass = 0; pass < 30; ++pass) {
  // Pre-create empty ThunkSections with a step size of about 2 * thunkSectionSpacing.
  // This ensures that for the most common thunk-needing relocation type, the relocations
  // can find a ThunkSection within thunkSectionSpacing bytes.
  if (pass == 0)
    createInitialThunkSections();

  bool changed = false;
  for (relocation : all_relocations) {
    // If this relocation needs a thunk and the thunk is still in range, skip.
    // Otherwise, restore the original relocation.
    if (pass > 0 && normalizeExistingThunk(rel))
      continue;

    if (!needsThunk(rel)) continue;
    Thunk *t = getOrCreateThunk(rel);
    ts = findOrCreateThunkSection(rel, src);
    ts->addThunk(t);
    rel.sym = t->getThunkTargetSym();  // redirect
    changed = true;
  }
  // Intersperse ThunkSections and regular input sections.
  mergeThunks();
  if (!changed) break;
  assignAddresses();  // recalculate with new thunks
}
```

Key details:

* **Multi-pass**: Iterates until convergence (max 30 passes). Adding thunks changes addresses, potentially putting previously-in-range calls out of range.
* **Pre-allocated ThunkSections**: On pass 0, `createInitialThunkSections` places empty `ThunkSection`s at regular intervals (`thunkSectionSpacing`). For AArch64: 128 MiB - 0x30000 ≈ 127.8 MiB.
* **Thunk reuse**: `getThunk` returns existing thunk if one exists for the same target; `normalizeExistingThunk` checks if a previously-created thunk is still in range.
* **ThunkSection placement**: `getISDThunkSec` finds a ThunkSection within branch range of the call site, or creates one adjacent to the calling InputSection.

### lld/MachO's thunk creation algorithm

lld/MachO uses a single-pass algorithm in `TextOutputSection::finalize`:

```cpp
for (callIdx = 0; callIdx < inputs.size(); ++callIdx) {
  // Finalize sections within forward branch range (minus slop)
  while (finalIdx < endIdx && fits_in_range(inputs[finalIdx]))
    finalizeOne(inputs[finalIdx++]);

  // Process branch relocations in this section
  for (Relocation &r : reverse(isec->relocs)) {
    if (!isBranchReloc(r)) continue;
    if (targetInRange(r)) continue;
    if (existingThunkInRange(r)) { reuse it; continue; }
    // Create new thunk and finalize it
    createThunk(r);
  }
}
```

Key differences from lld/ELF:

* **Single pass**: Addresses are assigned monotonically and never revisited
* **Slop reservation**: Reserves `slopScale * thunkSize` bytes (default: 256 × 12 = 3072 bytes on ARM64) to leave room for future thunks
* **Thunk naming**: `<function>.thunk.<sequence>` where sequence increments per target

[Thunk starvation problem](https://github.com/llvm/llvm-project/issues/50920): If many consecutive branches need thunks, each thunk (12 bytes) consumes slop faster than call sites (4 bytes apart) advance.
The test `lld/test/MachO/arm64-thunk-starvation.s` demonstrates this edge case.
Mitigation is increasing `--slop-scale`, but pathological cases with hundreds of consecutive out-of-range callees can still fail.

### mold's thunk creation algorithm

mold uses a two-pass approach:

* Pessimistically over-allocate thunks. Out-of-section relocations and relocations referencing to a section not assigned address yet pessimistically need thunks. (`requires_thunk(ctx, isec, rel, first_pass)` when `first_pass=true`)
* Then remove unnecessary ones.

Linker pass ordering:

* `compute_section_sizes()` calls `create_range_extension_thunks()` — final section addresses are NOT yet known
* `set_osec_offsets()` assigns section addresses
* `remove_redundant_thunks()` is called AFTER addresses are known — check unneeded thunks due to out-of-section relocations
* Rerun `set_osec_offsets()`

**Pass 1** (`create_range_extension_thunks`): Process sections in batches using a sliding window. The window tracks four positions:

```
  Sections:   [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] ...
               ^       ^       ^           ^
               A       B       C           D
               |       |_______|           |
               |         batch             |
               |                           |
               earliest                    thunk
               reachable                   placement
               from C
```

* **[B, C)** = current batch of sections to process (size ≤ branch_distance/5)
* **A** = earliest section still reachable from C (for thunk expiration)
* **D** = where to place the thunk (furthest point reachable from B)

```cpp
// Simplified from OutputSection<E>::create_range_extension_thunks
while (b < sections.size()) {
  // Advance D: find furthest point where thunk is reachable from B
  while (d < size && thunk_at_d_reachable_from_b)
    assign_address(sections[d++]);

  // Compute batch [B, C)
  c = b + 1;
  while (c < d && sections[c] < sections[b] + batch_size) c++;

  // Advance A: expire thunks no longer reachable
  while (a < b && sections[a] + branch_distance < sections[c]) a++;
  // Expire thunk groups before A: clear symbol flags.
  for (; t < thunks.size() && thunks[t].offset < sections[a]; t++)
    for (sym in thunks[t].symbols) sym->flags = 0;

  // Scan [B,C) relocations. If a symbol is not assigned to a thunk group yet,
  // assign it to the new thunk group at D.
  auto &thunk = thunks.emplace_back(new Thunk(offset));
  parallel_for(b, c, [&](i64 i) {
    for (rel in sections[i].relocs) {
      if (requires_thunk(rel)) {
        Symbol &sym = rel.symbol;
        if (!sym.flags.test_and_set()) {  // atomic: skip if already set
          lock_guard lock(mu);
          thunk.symbols.push_back(&sym);
        }
      }
    }
  });
  offset += thunk.size();
  b = c;  // Move to next batch
}
```

**Pass 2** (`remove_redundant_thunks`): After final addresses are known, remove thunk entries for symbols actually in range.

Key characteristics:

* **Pessimistic over-allocation**: Assumes all out-of-section calls need thunks; safe to shrink later
* **Batch size**: branch_distance/5 (25.6 MiB for AArch64, 3.2 MiB for AArch32)
* **Parallelism**: Uses TBB for parallel relocation scanning within each batch
* **Single branch range**: Uses one conservative `branch_distance` per architecture. For AArch32, uses ±16 MiB (Thumb limit) for all branches, whereas lld/ELF uses ±32 MiB for A32 branches.
* **Thunk size not accounted in D-advancement**: The actual thunk group size is unknown when advancing D, so the end of a large thunk group may be unreachable from the beginning of the batch.
* **No convergence loop**: Single forward pass for address assignment, no risk of non-convergence

### GNU ld's thunk creation algorithm

Each port implements the algorithm on their own. There is no code sharing.

GNU ld's AArch64 port (`bfd/elfnn-aarch64.c`) uses an iterative algorithm but with a single stub type and no lookup table.

**Main iteration loop** (`elfNN_aarch64_size_stubs()`):

```c
group_sections(htab, stub_group_size, ...);  // Default: 127 MiB
layout_sections_again();

for (;;) {
  stub_changed = false;
  _bfd_aarch64_add_call_stub_entries(&stub_changed, ...);
  if (!stub_changed)
    return true;
  _bfd_aarch64_resize_stubs(htab);
  layout_sections_again();
}
```


GNU ld's ppc64 port (`bfd/elf64-ppc.c`) uses an iterative multi-pass algorithm with a branch lookup table (`.branch_lt`) for long-range stubs.

**Section grouping**: Sections are grouped by `stub_group_size` (~28-30 MiB default); each group gets one stub section. For 14-bit conditional branches (`R_PPC64_REL14`, ±32KiB range), group size is reduced by 1024x.

**Main iteration loop** (`ppc64_elf_size_stubs()`):

```c
while (1) {
  // Scan all relocations in all input sections
  for (input_bfd; section; irela) {
    // Only process branch relocations (R_PPC64_REL24, R_PPC64_REL14, etc.)
    stub_type = ppc_type_of_stub(section, irela, ...);
    if (stub_type == ppc_stub_none)
      continue;
    // Create or merge stub entry
    stub_entry = ppc_add_stub(...);
  }

  // Size all stubs, potentially upgrading long_branch to plt_branch
  bfd_hash_traverse(&stub_hash_table, ppc_size_one_stub, ...);

  // Check for convergence
  if (!stub_changed && all_sizes_stable)
    break;

  // Re-layout sections
  layout_sections_again();
}
```

**Convergence control**:

- `STUB_SHRINK_ITER = 20` ([PR28827](https://sourceware.org/PR28827)): After 20 iterations, stub sections only grow (prevents oscillation)
- Convergence when: `!stub_changed && all section sizes stable`

**Stub type upgrade**: `ppc_type_of_stub()` initially returns `ppc_stub_long_branch` for out-of-range branches. Later, `ppc_size_one_stub()` checks if the stub's branch can reach; if not, it upgrades to `ppc_stub_plt_branch` and allocates an 8-byte entry in `.branch_lt`.

### Comparing linker thunk algorithms

| Aspect | lld/ELF | lld/MachO | mold | GNU ld ppc64 |
|--------|---------|-----------|------|--------------|
| Passes | Multi (max 30) | Single | Two | Multi (shrink after 20) |
| Strategy | Iterative refinement | Sliding window | Sliding window | Iterative refinement |
| Thunk placement | Pre-allocated intervals | Inline with slop | Batch intervals | Per stub-group |

## Linker relaxation

Some architectures take a different approach: instead of only expanding branches, the linker can also **shrink** instruction sequences when the target is close enough.
RISC-V and LoongArch both use this technique.
See [The dark side of RISC-V linker relaxation](/blog/2021-03-14-the-dark-side-of-riscv-linker-relaxation) for a deeper dive into the complexities and tradeoffs.

Consider a function call using the `call` pseudo-instruction, which expands to `auipc` + `jalr`:
```asm
# Before linking (8 bytes)
call ext
# Expands to:
#   auipc ra, %pcrel_hi(ext)
#   jalr ra, ra, %pcrel_lo(ext)
```

If `ext` is within ±1MiB, the linker can relax this to:
```asm
# After relaxation (4 bytes)
jal ext
```

This is enabled by `R_RISCV_RELAX` relocations that accompany `R_RISCV_CALL` relocations.
The `R_RISCV_RELAX` relocation signals to the linker that this instruction sequence is a candidate for shrinking.

Example object code before linking:
```
0000000000000006 <foo>:
       6: 97 00 00 00   auipc   ra, 0
                R_RISCV_CALL ext
                R_RISCV_RELAX *ABS*
       a: e7 80 00 00   jalr    ra
       e: 97 00 00 00   auipc   ra, 0
                R_RISCV_CALL ext
                R_RISCV_RELAX *ABS*
      12: e7 80 00 00   jalr    ra
```

After linking with relaxation enabled, the 8-byte `auipc`+`jalr` pairs become 4-byte `jal` instructions:
```
0000000000000244 <foo>:
     244: 41 11         addi    sp, sp, -16
     246: 06 e4         sd      ra, 8(sp)
     248: ef 00 80 01   jal     ext
     24c: ef 00 40 01   jal     ext
     250: ef 00 00 01   jal     ext
```

When the linker deletes instructions, it must also adjust:

* Subsequent instruction offsets within the section
* Symbol addresses
* Other relocations that reference affected locations
* Alignment directives (`R_RISCV_ALIGN`)

This makes RISC-V linker relaxation more complex than thunk insertion, but it provides code size benefits that other architectures cannot achieve at link time.

LoongArch uses a similar approach. A `pcaddu12i`+`jirl` sequence (`R_LARCH_CALL36`, ±128GiB range) can be relaxed to a single `bl` instruction (`R_LARCH_B26`, ±128MiB range) when the target is close enough.

## Diagnosing out-of-range errors

When you encounter a "relocation out of range" error, check the linker diagnostic and locate the relocatable file and function.
Determine how the function call is lowered in assembly.

## Summary

Handling long branches requires coordination across the toolchain:

| Stage | Technique | Example |
|-------|-----------|---------|
| Compiler | Branch relaxation pass | Invert condition + add unconditional jump |
| Assembler | Instruction relaxation | Invert condition + add unconditional jump |
| Linker | Range extension thunks | Generate trampolines |
| Linker | Linker relaxation | Shrink `auipc`+`jalr` to `jal` (RISC-V) |

The linker's thunk generation is particularly important for large programs where function calls may exceed branch ranges.
Different linkers use different algorithms with various tradeoffs between complexity, optimality, and robustness.

Linker relaxation approaches adopted by RISC-V and LoongArch is an alternative that avoids range extension thunks but introduces other complexities.

## Related

* [Relocation overflow and code models](/blog/2023-05-14-relocation-overflow-and-code-models)
* [Linker notes on AArch32](/blog/2023-04-23-linker-notes-on-aarch32)
* [Linker notes on AArch64](/blog/2023-03-05-linker-notes-on-aarch64)
* [Linker notes on Power ISA](/blog/2023-02-26-linker-notes-on-power-isa)
* [Linker notes on x86](/blog/2023-02-19-linker-notes-on-x86)
* [Toolchain notes on MIPS](/blog/2023-09-04-toolchain-notes-on-mips)
* [Toolchain notes on z/Architecture](/blog/2024-02-11-toolchain-notes-on-z-architecture)
