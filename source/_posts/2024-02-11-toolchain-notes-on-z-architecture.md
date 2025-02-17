layout: post
title: Toolchain notes on z/Architecture
author: MaskRay
tags: [llvm,linker,s390x]
---

This article describes some notes about [z/Architecture](https://en.wikipedia.org/wiki/Z/Architecture) with a focus on the ELF ABI and ELF linkers.
An [lld/ELF patch](https://github.com/llvm/llvm-project/pull/75643) sparked my motivation to study the architecture and write this post.

[z/Architecture](https://en.wikipedia.org/wiki/Z/Architecture) is a big-endian mainframe computer architecture supporting 24-bit, 31-bit, and 64-bit addressing modes.
It is the latest generation in a lineage stretching back to the 1964 with IBM System/360 (32-bit general-purpose registers and 24-bit addressing).
This lineage includes System/370 (1970), System/370 Extended Architecture (1983), Enterprise Systems Architecture/370 (1988), and Enterprise Systems Architecture/390 (1990).
For a deeper dive into the design choices behind z/Architecture's extension from ESA/390, you can refer to _"Development and attributes of z/Architecture."_
![IBM System/360 at Computer History Museum](/static/2024-02-11-toolchain-notes-on-z-architecture/system360.webp){ width=70% }

<!-- more -->

[Linux on IBM Z](https://en.wikipedia.org/wiki/Linux_on_IBM_Z) is a 64-bit operating system on z/Architecture, related to an older effort porting Linux to ESA/390.
As the Wikipedia page clarifies:

> Historically the Linux kernel architecture designations were "s390" and "s390x" to distinguish between the 32-bit and 64-bit Linux on IBM Z kernels respectively, but "s390" now also refers generally to the one Linux on IBM Z kernel architecture.

## Documents

* [_z/Architecture Principles of Operation_](https://linux.mainframe.blog/zarchitecture-principles-of-operation/): This is the instruction set manual with an unusual name inheirted from _IBM System/360 Principles of Operation_.
  ![IBM System/360 Principles of Operation and the Mythical Man-Month](/static/2024-02-11-toolchain-notes-on-z-architecture/system360-doc.webp){ width=70% }
* _Assembler Language Programming for IBM System z_: This book is more readable than _Principles of Operation_.
* _z/Architecture Reference Summary_: A concise reference of instructions.
* [_zSeries ELF Application Binary Interface Supplement_](https://refspecs.linuxbase.org/ELF/zSeries/lzsabi0_zSeries.html) (v1.0.2), 2002: This ABI document has been superseded by s390x-abi.
* <https://github.com/IBM/s390x-abi>: The latest version of the psABI (processor supplement to the System V ABI) resides here. While the absence of updates between 2002 and 2021 might seem odd, rest assured the documentation is actively maintained.

## Instruction notes

Each instruction has a length of two, four or six bytes (one to three halfwords), and must be located at a halfword boundary.
Six-byte instructions have been available since S/360.
The two most significant bits of the first halfword determines the length of instruction.

There are more than 1000 basic instructions.

There are 16 64-bit general-purpose registers, each treated as two independent 32-bit parts.
Certain instructions operate on the high 32-bit part. E.g. `aih %r2, 1` add 1 to the high 32-bit part.
I suspect that using these instructions to overcome register scarcity wouldn't be a good idea due to the overhead of register synchronization.

PC-relative addressing is supported with the general-instructions-extension facility (February 2008).
For example, only one instruction is needed to load `_GLOBAL_OFFSET_TABLE_` (see "Global Offset Table" below) into a register (usually r12).
```asm
larl    %r12, _GLOBAL_OFFSET_TABLE_ # r12 = _GLOBAL_OFFSET_TABLE_
```

The RIL instruction format, consisting of 6 bytes, encodes a register and a 32-bit immediate operand.
This enables it to implement valuable instructions like BRASL (Branch Relative And Save Long, like x86's CALL) and LARL (Load Address Relative Long, like x86's MOV with RIP-relative addressing).

```c
int var;
void fun();
int foo() { fun(); return var; }
```
```asm
// s390x
brasl   %r14, fun@PLT
lgfrl   %r2, var

// x86-64
callq   fun@PLT
movl    var(%rip), %eax
```

Note: RISC-V's JALR is akin to BRASL, as you can specify which register to save the return address.
z/Architecture's BRASL has a length of 6 bytes, so encoding the register, while wasting the encoding space, is more affordable.

LGF (Load, RXY-type) performs a load with a register offset and a 20-bit displacement (base+offset+disp20), but does not support a scaled index operation.
Two instructions, consisting of 12 bytes, are required to perform a simple index operation:
```c
int foo(int *a, long i) { return a[i+3]; }
// sllg    %r3,%r3,2
// lgf     %r2,12(%r3,%r2)
```

`-march=arch9` introduces some conditional instructions like LOCR/LOCGR (Load On Condition, RRF-c-type), `if (cond) r1 =  r2`.
In contrast, 4 bytes on other architectures can generally implement a more powerful three-register conditional move.

While I haven't had extensive time to study the instruction set architecture (ISA), I do see some clear limitations:

* Many instructions have different behaviors in 24-bit addressing, 31-bit addressing, and 64-bit addressing. This is needless complication for a CPU when 24-bit and 31-bit addressing become irrelevant for modern programs.
* The legacy instruction formats inherited from S/360, S/370, and S/390 restrict the design flexibility of z/Architecture.
* Instructions supporting 24-bit addressing become redundant in a 64-bit environment but must be retained for compatibility reasons.

This raises the question: when would IBM prefer designing a completely new architecture and implementing a dynamic binary translator for existing programs?

## ABI notes

r14 is used as the link register while r15 is the stack pointer.
In s390x-abi, registers r6 to r13, and r15 are designated as designated as non-volatile (not clobbered by a function call).
Registers r2 to r6 are used for integer arguments.

* r6 being non-volatile for argument storage seems uncommon compared to other architectures.
* Only 4 registers are used for integer argument storage, which is inadequate. It is unclear why r1 and r7 are not used.

The stack alignment is 8 bytes. Most 64-bit architectures employ 16-byte alignment.

Symbols representing a section offset must be halfword aligned. Compilers assume that an external symbol (e.g. `extern char a;`) to be halfword aligned.
[`-munaligned-symbols`](https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;h=6cb2f2c7f36c999590a949f663d6057cbc67271f) removes the assumption.

## Compilers

LLVM supports IBM z10 and newer models. 31-bit addressing is not supported.

## Global Offset Table

The `.got` section has 3 reserved entries.
The linker defines `_GLOBAL_OFFSET_TABLE_` at the start of `.got`.
`_GLOBAL_OFFSET_TABLE_[0]` stores the link-time address of `_DYNAMIC`, which is used by glibc.
`_GLOBAL_OFFSET_TABLE_[1]` and `_GLOBAL_OFFSET_TABLE_[2]` are for lazy binding PLT (`_dl_runtime_resolve` and link map in glibc).

The assembler modifier `@GOTENT` designates a 32-bit immediate operand.
The assembler modifier `@GOT` designates an immediate operand of either 16-bit or 32-bit.

Compilers generate a LGRL (Load Relative Long) instruction to load the GOT entry of a symbol.
When the symbol is non-preemptible and not an [ifunc](/blog/2021-01-18-gnu-indirect-function#non-preemptible-ifunc), the [GOT indirection can be optimized](/blog/2021-08-29-all-about-global-offset-table#got-optimization) to LARL (Load Address Relative Long).
This is similar to x86-64's GOTPCRELX optimization.

```asm
lgrl %r1, var@GOT            # R_390_GOTENT(var)

=>

larl %r1, var
```

## Procedure Linkage Table

At 32 bytes per entry, PLTs are notably larger than other architectures.
Only the first 14 bytes (encompassing three instructions) are strictly necessary for eager binding.

```asm
larl %r1, .got.plt[n]
lg %r1, 0(%r1)
br %r1
basr %r1, %r0
lgf %r1, 12(%r1)
jg .plt[0]
.long relocation offset
```

For lazy PLT binding, the `.got.plt` entry refers to `basr %r1, %r0` (14 bytes relative to the PLT entry), which stores the next instruction address into `r1`.
PLT0 is called with `r1` set to the relocation offset.
PLT0 sets up arguments and calls `.got[2]`, the PLT resolver in glibc.

mold utilizes a 16-byte PLT entry scheme that uses `basr %r0, %r1` instead of `br %r1` so that PLT0 can compute the relocation offset using `r0`.

## Relocations

There are 5 absolute relocation types: `R_390_{8,16,20,32,64}`.
They can be used as data relocations (`.byte`, `.short`, etc) as well as code relocations.

* `R_390_8` is used by instruction formats with a 8-bit immediate operand (e.g. SI).
* `R_390_16` is used by instruction formats with a 16-bit immediate operand (e.g. RI).
* `R_390_20` is used by instruction formats with a 20-bit immediate operand (e.g. RSY, RXY).
* `R_390_32` is used by instruction formats with a 32-bit immediate operand (e.g. RIL).

The assembler modifier `@PLTOFF` designates `R_390_PLTOFF16`, `R_390_PLTOFF32`, and `R_390_PLTOFF64`.
Their computation `&.plt[n] - .got + A` is similar to [`R_X86_64_PLTOFF64`](https://reviews.llvm.org/D112386) used by x86-64's large code model.
However, `-mcmodel=large` is unsupported, so these relocations seem not useful.

Relocation types `R_390_GOTPLT*` (`.got.plt[n]` relative to `.got` or PC) seem unused. GCC never emits the assembler modifier `@GOTPLT`.
I believe these relocations are not useful in the presence of PC-relative adddressing.

## Thread Local Storage

Refer to [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage) for TLS.
s390x utilizes TLS Variant II, [implemented by glibc](https://sourceware.org/pipermail/libc-alpha/2003-January/012611.html) in 2003.
The 64-bit TLS ABI closely mirrors the 32-bit TLS ABI, which itself is inspired by x86-32.
Unlike other architectures that revamped their TLS ABI during the 64-bit transition, s390x's predates modern instructions like LGFI and LGRL, resulting in a less efficient implementation compared to newer architectures.

First, let's look at thread pointer accessing.

* s390: 32-bit thread pointer stored in 32-bit access register `a0`.
* s390x: 64-bit thread pointer split across `a0` and `a1`, both still 32-bit.

This necessitates three instructions (14 bytes) to retrieve the full thread pointer, while 64-bit access registers would simplify this:
```asm
ear     %r0, %a0             # r0 = hi(r0) | a0
sllg    %r1, %r0, 32         # r1 = r0<<32
ear     %r1, %a1             # r1 = hi(r1) | a1 = a0<<32 | a1
```

Access registers holds 32-bit access-list-entry tokens (ALET), which are not used on Linux.

### General dynamic TLS model

In the general dynamic TLS model, a key difference compared to other architectures is the use of `__tls_get_offset` instead of `__tls_get_addr`.
The process involves several steps, illustrated by the provided assembly code:

```asm
ear     %r0, %a0
sllg    %r1, %r0, 32
ear     %r1, %a1             # r1 = TP
larl    %r12, _GLOBAL_OFFSET_TABLE_ # r12 = _GLOBAL_OFFSET_TABLE_

lgrl    %r2, .LCPI0_0        # r2 = *(.LCPI0_0) = an offset into .got
# r2 = __tls_get_offset(r2) = dtv[m]+a@DTPOFF - TP
brasl   %r14, __tls_get_offset@PLT:tls_gdcall:a # R_390_PLT32DBL(__tls_get_offset+0x2) at offset+2, R_390_TLS_GDCALL(a) at offset
lgf     %r2, 0(%r2,%r1)      # r2 = *(r2+r1) = *(dtv[m]+a@DTPOFF) = a

.section        .data.rel.ro,"aw",@progbits
.LCPI0_0:
  .quad   a@TLSGD            # R_390_TLS_GD64(a); linker resolves this to an offset into .got
```

* Retrieving the thread pointer and `_GLOBAL_OFFSET_TABLE_`: Four instructions are required but can be shared by subsequent TLS accesses. This step can be reordered.
* Obtaining the GOT offset: The offset (`a@TLSGD`) is stored in the `.data.rel.ro` section. The offset refers to two GOT entries (a `tls_index` structure), relocated by dynamic relocations `R_390_TLS_DTPMOD` and `R_390_TLS_DTPOFF`. The dynamic loader will set the values to `(m, a@DTPOFF)`, the module ID and an offset of the symbol relative to the dynamic TLS block.
* Finding the offset relative to the current dynamic TLS block (`DTPOFF`): `__tls_get_offset(r2)` returns `dtv[m] + a@DTPOFF - TP`. `__tls_get_addr` in other architectures just return `dtv[m] + a@DTPOFF`.
* Adding the thread pointer to get the symbol address in the current thread

In glibc `sysdeps/s390/dl-tls.h`, `__tls_get_offset` is defined as:
```asm
// unsigned long __tls_get_offset(unsigned long offset);

__tls_get_offset:
la      %r2,0(%r2,%r12)
jg      __tls_get_addr

// __tls_get_addr is not exported, satisfying linkers' requirement
```

While this approach works, it's considered the least efficient implementation of general dynamic TLS among the architectures I have analyzed.
Here is why:

* Ineffecient `tls_index` argument (similar to AArch32): This requires an extra lookup in `.data.rel.ro`.
* Unnecessary use of `_GLOBAL_OFFSET_TABLE_` (similar to x86-32): Instead of loading `a@TLSGD`, and then adding `_GLOBAL_OFFSET_TABLE_`, it is easier to just load the GOT entry address using LGRL.
* Redundant argument: `__tls_get_offset` takes the GOT offset instead of the direct GOT entry address.
* Indirect return value: Instead of returning the final TLS symbol address directly, `__tls_get_offset` only provides an offset, requiring an extra instruction for addition with the TP.

The 64-bit TLS ABI, modeled closely after the 32-bit ABI, was codified before nice instructions like LGRL (general-instructions-extension facility, February 2008) were available.
It clearly comes at the cost of performance.

The marker relocation `R_390_TLS_GDCALL` comes after `R_390_PLT32DBL`, different from other architectures.

The general-dynamic code sequence can be optimized to initial-exec or local-exec.

```asm
// general-dynamic to initial-exec
lgrl    %r2, .LCPIC0_0       # r2 = *(.LCPI0_0) = &.got[n]-_GLOBAL_OFFSET_TABLE_
lg      %r2, 0(%r2,%r12)     # r2 = TP offset
lgf     %r2, 0(%r2,%r1)      # r2 = *(r2+r1) = TLS value in the current thread

.section .data.rel.ro,"aw",@progbits
.LCPI0_0:
  .quad &.got[n]-_GLOBAL_OFFSET_TABLE_ # .got[n], relocated by R_PPC64_TPREL64, holds the TP offset

// general-dynamic to local-exec
lgrl    %r2, .LCPIC0_0       # r2 = *(.LCPI0_0) = TP offset
brasl   0, .+0               # nop
lgf     %r2, 0(%r2,%r1)      # r2 = *(r2+r1) = TLS value in the current thread

.section .data.rel.ro,"aw",@progbits
.LCPI0_0:
  .quad a@NTPOFF
```

In both cases, the linker only needs to patch one instruction, instead of four for PPC64.

### Local dynamic TLS model

The process involves several steps, illustrated by the provided assembly code:

```asm
lgrl    %r2,.LC0             # r2 = *(.LC0) = GOT offset of a tls_index object holding {module_ID, 0}
# r2 = __tls_get_offset(r2) = dtv[m]-TP
brasl   %r14,__tls_get_offset@PLT:tls_ldcall:a # R_390_PLT32DBL(__tls_get_offset+0x2) at offset+2, R_390_TLS_LDCALL(a) at offset

ear     %r3, %a0
sllg    %r4, %r3, 32
ear     %r4, %a1             # r4 = TP
la      %r2,0(%r2,%r4)       # r2 = r2+r4 = dtv[m]

lgrl    %r1, .LC1            # r1 = a@DTPOFF
lgf     %r1,0(%r1,%r2)       # r1 = *(a@DTPOFF + dtv[m]) = a

lgrl    %r1, .LC2            # r1 = b@DTPOFF
lgf     %r1,0(%r1,%r2)       # r1 = *(b@DTPOFF + dtv[m]) = b

.section .data.rel.ro,"aw"
.align 8
.LC0: .quad a@TLSLDM         # R_390_TLS_LDM64(a); linker resolves this to a GOT offset of tls_index{m, 0}
.LC1: .quad a@DTPOFF         # R_390_TLS_LDO64(a); linker resolves this to a's offset relative to dtv[m]
.LC2: .quad b@DTPOFF         # R_390_TLS_LDO64(b); linker resolves this to b's offset relative to dtv[m]
```

* Retrieving the thread pointer and `_GLOBAL_OFFSET_TABLE_`
* Obtaining the GOT offset: The offset (`a@TLSLDM`) is stored in the `.data.rel.ro` section. The offset refers to two GOT entries (a `tls_index` structure): the module ID and a zero. The module ID entry is relocated by a dynamic relocation `R_390_TLS_DTPMOD`.
* Finding the dynamic TLS block address: `__tls_get_offset(r2)` returns `dtv[m] - TP`. It is not `dtv[m] + XXX - TP` because the second GOT entry is zero.
* Adding DTPOFF to get the symbol address in the current thread

The first three steps can be shared among TLS symbols.

Similar to general-dynamic, the marker relocation `R_390_TLS_LDCALL` comes after `R_390_PLT32DBL`, different from other architectures.
This makes lld implementation awkward.

The local-dynamic code sequence can be optimized to local-exec.

```asm
lgrl    %r2,.LC0             # r2 = 0
brcl    0, .                 # nop

ear     %r3, %a0
sllg    %r4, %r3, 32
ear     %r4, %a1             # r4 = TP
la      %r2,0(%r2,%r4)       # r2 = r2+r4 = TP

lgrl    %r1, .LC1            # r1 = a@NTPOFF
lgf     %r1,0(%r1,%r2)       # r1 = *(a@NTPOFF + TP) = a

lgrl    %r1, .LC2            # r1 = b@NTPOFF
lgf     %r1,0(%r1,%r2)       # r1 = *(b@NTPOFF + TP) = b

.section .data.rel.ro,"aw"
.align 8
.LC0: .quad 0
.LC1: .quad a@NTPOFF         # a's TP offset
.LC2: .quad b@NTPOFF         # b's TP offset
```

### Initial Exec TLS model

```asm
lgrl    %r1, a@INDNTPOFF     # R_390_TLS_IEENT(a); linker resolves this to a GOT holding the TP offset
lgf     %r1, 0(%r1,%r7)      # r1 = *(a@NTPOFF + TP) = a
```

Optimizing the code sequence to local-exec is straightforward: changing the first instruction to `lgfi %r1, a@NTPOFF`.
However, LGFI (Load Immediate) is part of the extended-immediate facility (September 2005), introduced with System z9 109, unavailable when the ABI was defined.

Relocation types `R_390_TLS_IE32`/`R_390_TLS_IE64` for the initial-exec TLS model seem not useful.

### Local Exec TLS model

The code sequence loads the TP offset indirectly in a manner similar to AArch32.
```asm
lgrl    %r1, .LC0            # r1 = a@NTPOFF
lgf     %r1, 0(%r1,%r7)      # r1 = *(a@NTPOFF + TP) = a

.section .data.rel.ro,"aw"
.LC0: .quad a@NTPOFF         # R_390_TLS_LE64(a); linker resolves this to the TP offset, a negative integer
```

The indirection is unfortunate. Similarly, LGFI (Load Immediate) can be used instead.

## Linux distributions

* <https://almalinux.org/blog/how-we-built-almalinux-86-for-s390x/>
* <https://wiki.alpinelinux.org/wiki/S390x/Installation>
* <https://wiki.debian.org/SupportedArchitectures>
* <https://alt.fedoraproject.org/alt/>
* <https://wiki.gentoo.org/wiki/Project:S390>
* <https://en.opensuse.org/ZSystems>
