---
layout: post
title: Call relocation types
author: MaskRay
tags: [binutils,linker]
---

Most architectures encode direct branch/call instructions with a PC-relative displacement.
This post discusses a specific category of branch relocations: those used for direct function calls and tail calls.
Some architectures use two ELF relocation types for a call instruction:

```asm
# i386, x86-64
call foo              # R_386_PC32, R_X86_64_PC32
call foo@plt          # R_386_PLT32, R_X86_64_PLT32

# m68k
bsr.l foo             # R_68K_PC32
bsr.l foo@plt         # R_68K_PLT32

# s390/s390x
brasl %r14, foo       # R_390_PC32DBL
brasl %r14, foo@plt   # R_390_PLT32DBL

# sparc
call    foo, 0        # not PIC: R_SPARC_WDISP30
call    foo, 0        # gas -KPIC: R_SPARC_WPLT30
```

<!-- more -->

This post describes why I think this happened.

## Static linking: one type suffices

In the static linking model, all symbols are resolved at link time: every symbol is either defined in a relocatable object file or an undefined weak symbol.
A branch instruction with a PC-relative displacement—x86 `call`, m68k `bsr.l`, s390 `brasl`—can reuse the same PC-relative data relocation type used for data references.

* i386: `R_386_PC32` for both `call foo` and `.long foo - .`
* x86-64: `R_X86_64_PC32` for both `call foo` and `.long foo - .`
* m68k: `R_68K_PC32` for `bsr.l foo`, `move.l var,%d0`, and `.long foo - .`
* s390x: `R_390_PC32DBL` for `brasl %r14, foo`, `larl %r1, var`, and `.long foo - .`

No separate "call" relocation type is needed.
The linker simply patches the displacement to point to the symbol address.

## Dynamic linking changes the picture

With System V Release 4 style shared libraries, variable access and function calls diverge.

For **variables and function addresses**, a reference from one component to a symbol defined in another cannot use a plain PC-relative relocation, because the distance between the two components is not known at link time.
The [Global Offset Table](/blog/2021-08-29-all-about-global-offset-table) was introduced for this purpose, along with GOT-generating relocation types.
(Additionally, [copy relocations](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected) are a workaround for external data symbols from `-fno-pic` relocatable files.)
To satisfy the **pointer equality** requirement, a PC-relative data relocation in an executable must resolve to the same address as its counterpart in a shared object—this is why GOT indirection is used for symbols not known at compile time to be preemptible.

For **direct function calls**, the situation is different.
A call instruction has "transfer control there by any means" semantics - the caller usually doesn't care _how_ the callee is reached, only that it gets there.
This allows the linker to interpose a [PLT stub](/blog/2021-09-19-all-about-procedure-linkage-table) when the target is in another component, without any special code sequence at the call site.
Alternatively, some architectures support an indirect call sequence that bypasses PLT entirely: [`-fno-plt` on x86](/blog/2021-09-19-all-about-procedure-linkage-table#fno-plt) and [`-mno-plt` on MIPS](/blog/2023-09-04-toolchain-notes-on-mips#:~:text=mno%2dplt) (o32/n32 non-PIC).

Variable accesses do not have the same semantics - so the PC-relative data relocation type cannot be reused on a call instruction.

This is why separate branch relocation types were introduced: `R_386_PLT32`, `R_68K_PLT32`, `R_390_PLT32DBL`, and so on.
The relocation type carries the semantic information: "this is a function call that can use PLT indirection."

## Misleading names

The `@plt` notation in assembly and the `PLT32` relocation type names are misleading.
They suggest that a PLT entry is involved, but that is often not the case - when the callee is defined in the same component, the linker resolves the branch directly—no PLT entry is created.

`R_386_CALL32` and `R_X86_64_CALL32` would have been a better name.

In addition, the `@plt` notation itself is problematic as a [relocation specifier](/blog/2025-03-16-relocation-generation-in-assemblers#relocation-specifier-flavors).

## Architecture comparison

**Single type (clean design).**
Some architectures recognized from the start that one call relocation type is sufficient.
The linker can decide whether a PLT stub is needed based on the symbol's binding and visibility.

* AArch64: `R_AARCH64_CALL26` for `bl` and `R_AARCH64_JUMP26` for `b`.
* PowerPC64 ELFv2: `R_PPC64_REL24` for `bl`.

These architectures never had the naming confusion—there is no "PLT" in the relocation name, and no redundant pair.

**Redundant pairs (misguided).**
Some architectures introduced separate "PLT" and "non-PLT" call relocation types, creating a distinction without a real difference.

* SPARC: `R_SPARC_WPLT30` alongside `R_SPARC_WDISP30`. The assembler decides at assembly time based on PIC mode and symbol preemptivity, when ideally the linker should make these decisions.
* PPC32: `R_PPC_REL24` (non-PIC) and `R_PPC_PLTREL24` (PIC) have genuinely different semantics (the addend of `R_PPC_PLTREL24` encodes the r30 GOT pointer setup). However, `R_PPC_LOCAL24PC` is entirely useless—all occurrences can be replaced with `R_PPC_REL24`.
* RISC-V: `R_RISCV_CALL_PLT` alongside the now-removed `R_RISCV_CALL`. The community recognized that only one relocation is needed. `R_RISCV_CALL_PLT` is kept (despite the name, does not mandate a PLT entry).

x86-64 started with `R_X86_64_PC32` for `call foo` (inherited from the static-linking mindset) and `R_X86_64_PLT32` for `call foo@plt` (symbols not compile-time known to be non-preemptible).
In 2018, binutils <https://sourceware.org/bugzilla/show_bug.cgi?id=22791> switched to `R_X86_64_PLT32` for `call foo`.
LLVM integrated assembler followed suit.

This means `R_X86_64_PC32` is now effectively reserved for data references, and `R_X86_64_PLT32` marks all calls—a clean separation achieved by convention.

However, GNU Assembler still produces `R_X86_64_PC32` when `call foo` references an `STB_LOCAL` symbol.
I've sent a patch to fix this: [[PATCH v2] x86: keep PLT32 relocation for local symbols instead of converting to PC32](https://sourceware.org/pipermail/binutils/2026-February/148251.html).

GCC's s390 port seems to always generate `@plt` (even for hidden visibility functions), leading to `R_390_PLT32DBL` relocations.

## Range extension thunks

When a branch target is out of range, some architectures allow the linker to insert a [range extension thunks](/blog/2026-01-25-long-branches-in-compilers-assemblers-and-linkers#linker-range-extension-thunks):
On AArch64 and PowerPC64, this is well established.

On x86-64, the ±2GiB range of `call`/`jmp` has been sufficient so far, but as executables grow, [relocation overflow](/blog/2023-05-14-relocation-overflow-and-code-models) becomes a concern.
There are proposals to add range extension thunks to x86-64, which would require the linker to identify call sites-a PC-relative data relocation like `R_X86_64_PC32` would not be suitable (due to pointer equality requirement), making consistent use of `R_X86_64_PLT32` for calls all the more important.

## Recommendation for future architectures

For a specific instruction or pseudo instruction for function calls and tail calls, use a single call relocation type—no "PLT" vs. "non-PLT" distinction.
The assembler should emit the same relocation, and the linker, which knows whether the symbol is preemptible, decides whether a PLT stub is needed.
Optionally enable range extension thunks for the relocation type.
AArch64's `R_AARCH64_CALL26` and PowerPC64 ELFv2's `R_PPC64_REL24` demonstrate this approach well.

This discussion does not apply to intra-function branches, which target local labels.

## See also

* [All about Procedure Linkage Table](/blog/2021-09-19-all-about-procedure-linkage-table) for how PLT works
* [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table) for the data-access counterpart
* [Copy relocations, canonical PLT entries and protected visibility](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected) for the variable-access complications
* [Relocation generation in assemblers](/blog/2025-03-16-relocation-generation-in-assemblers) for how assemblers decide which relocations to emit
* [Long branches in compilers, assemblers, and linkers](/blog/2026-01-25-long-branches-in-compilers-assemblers-and-linkers) for range extension thunks
