---
layout: post
title: Linker notes on AArch32
author: MaskRay
tags: [linker,llvm]
---

UNDER CONSTRUCTION

This article describes target-specific details about AArch32 in ELF linkers.
I described AArch64 in a [previous article](/blog/2023-03-05-linker-notes-on-aarch64).

AArch32 is the 32-bit execution state for the Arm architecture and runs the A32 and T32 instruction sets.
A32 refers to the old ISA with a 32-bit fixed width, while T32 refers to the mixed 16-bit and 32-bit Thumb2 instructions.

"AArch32", "A32", and "T32" are new names.
Many projects use "ARM", "Arm", or "arm" as their port name.

<!-- more -->

## ABI documents

* [ELF for the Arm® Architecture](https://github.com/ARM-software/abi-aa/blob/main/aaelf32/aaelf32.rst)
* [Procedure Call Standard for the Arm® Architecture](https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst)
* [C++ ABI for the Arm® Architecture](https://github.com/ARM-software/abi-aa/blob/main/cppabi32/cppabi32.rst)
* [Exception Handling ABI for the Arm® Architecture](https://github.com/ARM-software/abi-aa/blob/main/ehabi32/ehabi32.rst)

## Global Offset Table

The Global Offset Table consists of two sections:

* `.got.plt` holds code addresses for PLT.
* `.got` holds other addresses and offsets.

The symbol `_GLOBAL_OFFSET_TABLE_` is defined at the beginning of the `.got` section.
GNU ld reserves a single entry for `.got` and `.got[0]` holds the link-time address of `_DYNAMIC` for a legacy reason
Versions of glibc prior to 2.35 have the `_DYNAMIC` requirement. See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table#global_offset_table_0).

## Procedure Linkage Table

The PLT header looks like:

```asm
L1: str lr, [sp, #-4]!
    add lr, pc,  #0x0NN00000 &(.got.plt - L1 - 4)
    add lr, lr,  #0x000NN000 &(.got.plt - L1 - 4)
    ldr pc, [lr, #0x00000NNN] &(.got.plt -L1 - 4)
```

If `.git.plt-.plt-4` exceeds the +-128MiB range, a long-form PLT header is needed.

In binutils, `bfd/elf32-arm.c:elf32_thumb2_plt_entry` supports a PLT scheme for Thumb2 (<https://sourceware.org/PR16017>).
For MMU-less microcontroller processors that only support Thumb instructions, the use cases are unclear since System V style dynamic linking is unavailable.
FDPIC is needed instead.

## Cortex-M Security Extensions

### `--cmse-implib`

This option is for linker support for the [Cortex-M Security Extensions (CMSE)](https://arm-software.github.io/acle/cmse/cmse.html). It does two jobs:

* synthesize secure gateway veneers
* write a CMSE import library when `--out-implib=` is specified

If a non-local symbol `__acle_se_$sym` is present, report an error if `$sym` is not defined.
Otherwise `$sym` is considered a secure gateway veneer.
Both `__acle_se_$sym` and `$sym` must be a non-absolute function with an odd `st_value`.

If the addresses of `__acle_se_$sym` and `$sym` are not equal, the linker considers that there is an inline secure gateway and doesn't do anything special; otherwise the linker synthesizes a secure gateway veneer in a special section `.gnu.sgstubs` with the following logic.

The linker allocates an input section in `.gnu.sgstubs` and defines `$sym` relative to it.
In the output file, `$sym` is moved to `.gnu.sgstubs`, a different text section.
```asm
<.gnu.sgstubs>:
  ...
$sym:
  sg
  b.w __acle_se_$sym
```

If `--in-implib` is specified and the library defines `$sym` (say the address is `$addr`), in the output `$sym` has a fixed address of `$addr`.
Otherwise, the linker assigns an address (larger than all synthesized secure gateway veneers with fixed addresses).

### `--out-implib=out.lib`

Used with `--cmse-implib`. Write the CMSE import library to `out.lib`.

`out.lib` will have 3 sections: `.symtab, .strtab, .shstrtab`.
For every synthesized Secure Gateway veneer, write a `SHN_ABS` symbol whose address is `$addr` (if specified by the `--in-implib` library) or the linker-assigned address.
(The CMSE import library does not contain text sections, so a defined symbol has to use `SHN_ABS`.)

## Thread Local Storage

AArch32 uses a variant of TLS Variant I: the static TLS blocks are placed above the thread pointer. The thread pointer points to the end of the thread control block.

The linker doesn't perform TLS optimization.

The traditional general dynamic and local dynamic TLS models are used by default.
There is a [TLSDESC ABI](https://www.fsfla.org/~lxoliva/writeups/TLS/RFC-TLSDESC-ARM.txt).

See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage).

## Thunks

A destination not reachable by the branch instruction needs a range extension thunk.
ARM and Thumb state changes need a thunk as well.

### v4, v4T

ARMv4T introduced the 16-bit Thumb instruction set.
These processors do not support BLX. ARM/Thumb state change must be done using BX.

In the ARM state, the relocation types `R_ARM_PC24/R_ARM_PLT32/R_ARM_JUMP24/R_ARM_CALL` may need range extension or state change.
lld 16.0.0 has added thunk support for Thumb.
We can build Game Boy Advance or Nintendo DS roms using Thumb code with ld.lld.

```asm
// ARM to ARM, absolute
ldr pc, [pc, #-4]
L1: .word S

// ARM to Thumb, absolute
ldr r12, [pc] ; L1
bx r12
L1: .word S

// ARM to ARM, position-independent
ldr ip, [pc]
L1: add pc, pc, ip
.word S - (L1 + 8)

// ARM to Thumb, position-independent
ldr ip, [pc]
L1: add ip, pc, ip
bx ip
.word S - (L1 + 8)
```

`R_ARM_THM_CALL`
```asm
// Thumb to ARM, absolute
bx
b #-6
ldr pc, [pc, #-4]
L1: .word S

// Thumb to Thumb, absolute
bx
b #-6
ldr ip, [pc]
bx ip
L1: .word S

// Thumb to ARM, position-independent
bx
b #-6
ldr ip, [pc, #]
L1: add ip, pc, ip
L2: .word S - (L1 + 8)

// Thumb to Thumb, position-independent
bx
b #-6
ldr ip, [pc, #-4]
L1: add ip, pc, ip
bx ip
L2: .word S - (L1 + 8)
```

### v5, v6, v6-K, v6-KZ

These are pre-Cortex processors that support BLX, but not Thumb branch range extension or MOVT/MOVW.

There is no Thumb branch instruction in ARMv5 that supports thunks.
LDR can switch processor states.

```asm
// absolute (LDR can switch processor states)
ldr pc, [pc, #-4]
.word S

// position-independent
ldr ip, [pc, #4]
L1: add ip, pc, ip
bx ip
.word S - (L1 + 8)
```

### v6-M

Only Thumb instructions are supported.
These processors support BLX and J1 J2 encodings (branch range extension), but not MOVT/MOVW.

```asm
// near
b.w S

// far, absolute
push {r0, r1}
ldr r0, [pc, #4]
str r0, [sp, #4]
pop {r0, pc}
.word S

// far, position-independent
push {r0}
ldr r0, [pc, #8]; L2
mov ip, r0
pop {r0}
L1: add pc, ip
nop
L2: .word S - (L1 + 4)
```

### v6T2, v7 (v7-A, v7-R, v7-M, v7E-M) and newer

ARMv6T2 introduced Thumb-2.
These processors support BLX/MOVT/MOVW and Thumb branch range extension.
(All architectures used in Cortex processors with the exception of v6-M and v6S-M have the MOVT/MOVW instructions.)

A pair of MOVT/MOVT instructions may materialize:

* a 32-bit immediate (e.g. absolute address) using `R_ARM_MOVT_ABS`/`R_ARM_MOVW_ABS_NC` relocations,
* a PC-relative address using `R_ARM_MOVT_PREL`/`R_ARM_MOVW_PREL_NC` relocations,
* or a static-base-relative address using `R_ARM_MOVT_BREL`/`R_ARM_MOVW_BREL_NC` relocations.

In ELF, `R_ARM_MOVT_PREL`/`R_ARM_MOVW_PREL_NC` are not utilized when materializing a local symbol address. <https://bugs.llvm.org/show_bug.cgi?id=28229>
However, this should be a concern as `R_ARM_MOVT_BREL`/`R_ARM_MOVW_BREL_NC` are used by ROPI/RWPI.

In the ARM state, these relocation types `R_ARM_PC24, R_ARM_PLT32, R_ARM_JUMP24, R_ARM_CALL` may need thunk for range extension or state change.

```asm
// near and the destination is in the ARM state
b S

// absolute
movw ip, :lower16:S
movt ip, :upper16:S
bx ip

// position-independent
movw ip, :lower16:S-(L1+8)
movt ip, :upper16:S-(L1+8)
L1: add ip, ip, pc
bx ip
```

In the Thumb state, these relocation types `R_ARM_THM_JUMP19, R_ARM_THM_JUMP24, R_ARM_THM_CALL` may need thunk for range extension or state change.

```asm
// near and the destination is in the Thumb state
b.w S

// absolute
movw ip, :lower16:S
movt ip, :upper16:S
bx ip

// position-independent
movw ip, :lower16:S-(L1+4)
movt ip, :upper16:S-(L1+4)
L1: add ip, ip, pc
bx ip
```

MOVT extracts the upper bits of a value and requires specific handling in both ELF and Mach-O formats, as they both utilize implicit addends.
In ELF, it utilizes the REL relocation format. When the assembler processes the MOVT fixup, it produces two possible results (`ARMAsmBackend::adjustFixupValue`):

* Resolved fixups: The assembler updates the relevant bits using `value >> 16`.
* Unresolved fixups: The assembler creates a relocation based on the original value.

## `--fix-cortex-a8`

This option enables a linker workaround for Arm Cortex-A8 Errata 657417.
Linkers scan a 4-byte Thumb-2 branch instruction (Bcc.w, B.w, BLX.w, BL.w) that spans two 4KiB pages, and the target address of the branch falls within the first region.
The branch instruction follows a 4-byte non-branch instruction.
This may result in an incorrect instruction fetch or processor deadlock.

Oncea erratum condition is detected, linkers try to rewrite it into an alternative code sequence.
See the comments in the implementations for detail.

## `SHT_ARM_ATTRIBUTES`

Usually named `.ARM.attributes`.

## `SHT_ARM_EXIDX` (`.ARM.exidx`) and `.ARM.extab`

See [Stack unwinding](https://maskray.me/blog/2020-11-08-stack-unwinding)

## `--be8`

When linking big-endian images there are the deprecated BE-32 mode (word-invariant addressing big-endian mode) and the new BE-8 mode (byte-invariant addressing big-endian mode).

BE-32 is used by older architectures like arm7tdmi and arm926ej-s.

For ARMv6-M, ARMv7, and later architectures the default is BE8.
The relocatable object files have big-endian code and data.
Compiler drivers pass `--be8` to the linker to to convert big-endian code to little-endian.

The linker finds `$a`/`$d`/`$t` mapping symbols to locate ARM and Thumb code and perform byte swapping.
ARM code is reversed as 4-byte units, Thumb code is reversed as 2-byte units, while data is unchanged.
The linker sets the `EF_ARM_BE8` flag in the ELF header.

```sh
cat > a.s <<e
.syntax unified
.cpu    arm1176jzf-s

.text
.globl _start
.type _start,%function
_start:
  bl thumbfunc
  bx lr
  .word   0x12345678

.thumb
.section .text.2, "ax", %progbits
.globl thumbfunc
.type thumbfunc,%function
thumbfunc:
  bx lr
e
clang --target=armeb-none-linux-gnueabi -c a.s
arm-linux-gnueabihf-ld -EB --be8 a.o -o a
```

`a.o` and `a` have data (`.word`) of the same endianness, but `a` has little-endian instructions.

```
% llvm-objdump -s -d -j .text -j .text.2 a.o

a.o:    file format elf32-bigarm
Contents of section .text:
 0000 ebfffffe e12fff1e 12345678           ...../...4Vx
Contents of section .text.2:
 0000 4770                                 Gp

Disassembly of section .text:

00000000 <_start>:
       0: ebfffffe      bl      0x0 <_start>            @ imm = #-0x8
       4: e12fff1e      bx      lr

00000008 <$d.1>:
       8: 12 34 56 78   .word   0x12345678

Disassembly of section .text.2:

00000000 <thumbfunc>:
       0: 4770          bx      lr
% llvm-objdump -s -d -j .text a

a:      file format elf32-bigarm
Contents of section .text:
 10058 020000eb 1eff2fe1 12345678 70470000  ....../..4VxpG..
 10068 00c09fe5 1cff2fe1 00010065 00000000  ....../....e....

Disassembly of section .text:

00010058 <_start>:
   10058: eb000002      bl      0x10068 <__thumbfunc_from_arm> @ imm = #0x8
   1005c: e12fff1e      bx      lr

00010060 <$d.1>:
   10060: 12 34 56 78   .word   0x12345678

00010064 <thumbfunc>:
   10064: 4770          bx      lr
   10066: 0000          movs    r0, r0

00010068 <__thumbfunc_from_arm>:
   10068: e59fc000      ldr     r12, [pc]               @ 0x10070 <__thumbfunc_from_arm+0x8>
   1006c: e12fff1c      bx      r12

00010070 <$d>:
   10070: 00 01 00 65   .word   0x00010065
   10074: 00 00 00 00   .word   0x00000000
% readelf -h a
...
 Flags:                             0x5800200, Version5 EABI, soft-float ABI, BE8
```

## Static base

Several relocation types encode an static base relative offset, e.g. `R_ARM_GOT_BREL`.

## FDPIC ABI

<https://github.com/mickael-guene/fdpic_doc/blob/master/abi.txt>
