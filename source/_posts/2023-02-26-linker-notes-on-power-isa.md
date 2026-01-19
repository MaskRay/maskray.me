layout: post
title: Linker notes on Power ISA
author: MaskRay
tags: [llvm,linker]
---

This article describes target-specific details about Power ISA in ELF linkers.
Initially there was IBM POWER. The 1991 Apple–IBM–Motorola alliance created PowerPC.
In 2006, the architecture was rebranded as Power ISA.
According to the ISA manual, "In 2006, Freescale and IBM collaborated on the creation of the Power ISA Version 2.03, which represented the reunification of the architecture by combining Book E content with the more general purpose PowerPC Version 2.02."

The terms "PowerPC" and "powerpc" remain popular in numerous places, including the `powerpc-*-*-*` and `powerpc64-*-*-*` in official target triple names.
The abbreviation "PPC" ("ppc") is used in numerous places as well.
For simplicity, I will refer to the 32-bit architecture as "PPC32" and the 64-bit architecture as "PPC64".

We will see how the lack of PC-relative addressing before Power10 has caused great complexity to the ABI and linkers.

<!-- more -->

## ABI documents

* _Power Architecture™ 32-bit Application Binary Interface Supplement 1.0 - Linux® & Embedded_ revised in 2011.
* _64-bit PowerPC ELF Application Binary Interface Supplement 1.9_. This is commonly referred to as ELFv1 and is obsolete. Some 64-bit targets still use this ABI.
* _64-Bit ELF V2 ABI Specification: Power Architecture_

The 32-bit ELF ABI is more or less not cared for by maintainers and only remains relevant among some enthusiasts.
In 2019, I spent one week studying PPC32 ABI and added the PPC32 port to ld.lld.

For a 64-bit object file, the presence of a section `.opd` is a good indicator for ELFv1.
`e_flags` being 2 is a good indicator for ELFv2.
`e_flags` being 0 is either an ELFv1 object file, or an object file not using any feature affected by the differences.

_A new ABI for little-endian PowerPC64 Design & Implementation_ (2014) describes the motivation for introducing ELFv2.

## Global Offset Table

### PPC32 GOT

On PPC32, `_GLOBAL_OFFSET_TABLE_` is defined at the start of the section `.got`.
`.got` has 3 reserved entries.
`_GLOBAL_OFFSET_TABLE_[0]` stores the link-time address of `_DYNAMIC`, which is used by glibc `sysdeps/powerpc/powerpc32/dl-machine.h`.
`_GLOBAL_OFFSET_TABLE_[1]` and `_GLOBAL_OFFSET_TABLE_[2]` are for lazy binding PLT (`_dl_runtime_resolve` and link map in glibc).

`.plt` is like `.got.plt` for other architectures.
`.plt[n]` holds the address of a PLT entry (somewhere in `.glink`).

Like x86-32, PPC32 lacks memory load with PC-relative addressing.
As a poor man's replacement, PPC32 sets up r30 to hold a GOT base for position-independent code (PIC).
The GOT base is different for small PIC and large PIC.

* For `-fpic` and `-fpie`, r30 refers to `_GLOBAL_OFFSET_TABLE_` in the component.
* For `-fPIC` and `-fPIE`, r30 refers to `.got2` for the current translation unit. This has implications for PLT-generating relocations as we will see below.

```asm
.section        ".got2","aw"
.align 2
.LCTOC1 = .+32768
.LC0:
  .long var

  ...
  bcl 20,31,.L2
.L2:
  mflr 30                     # r30 = lr
  addis 30,30,.LCTOC1-.L2@ha
  addi 30,30,.LCTOC1-.L2@l    # finish setting up the GOT base
  lwz 9,.LC0-.LCTOC1(30)      # load the address of var relative to the GOT base
```

The component may have multiple translation units and each has a different `.got2`.
In the output file, `.got2` in one file may have an arbitrary offset relative to the output `.got2`.

### PPC64 GOT

On PPC64, `.got` has 1 reserved entry: the link-time address of `.TOC.`.
`.TOC.` is defined at the start of the section `.got` plus 0x8000.

`.plt` is like `.got.plt` for other architectures. `.plt` has the type `SHT_NOBITS` and an alignment of 4.

### PPC64 ELFv2 Table of Contents (TOC)

Before Power10, PPC64 uses `.toc` instead of `.got` to hold the addresses of global variables and address-taken functions.
This is different from most architectures.

```c
extern int var0, var1;
__attribute__((noinline)) int foo() { return var0 + var1; }
int bar() { return foo(); }
```

The above C program compiles to the following assembly:
```asm
foo:
.Lfunc_begin0:
.Lfunc_gep0:
  addis 2, 12, .TOC.-.Lfunc_gep0@ha
  addi 2, 2, .TOC.-.Lfunc_gep0@l
.Lfunc_lep0:
  .localentry     foo, .Lfunc_lep0-.Lfunc_gep0

  addis 3, 2, .LC0@toc@ha
  addis 4, 2, .LC1@toc@ha
  ld 3, .LC0@toc@l(3)
  ld 4, .LC1@toc@l(4)
  lwz 3, 0(3)
  lwz 4, 0(4)
  add 3, 4, 3
  extsw 3, 3
  blr

.section .toc,"aw",@progbits
.LC0:
  .tc var0[TC],var0
.LC1:
  .tc var1[TC],var1
```

`foo` has a global entry `foo`/`.Lfunc_gep0` and a local entry `.Lfunc_lep0`.
After the local entry, r2 holds the address of the TOC base of the current component.

If `foo` and a caller of `foo` are in the same component (e.g. `bar`), the caller may branch directly to the local entry, skipping a few instructions starting at the global entry (usually 2).
Otherwise, the caller needs to branch to the global entry so that `foo` will update r2 itself. This update requires that r12 points to the function entry address.
We will see that maintaining r2 and r12 causes a lot of trouble in sections diving into call stubs.

Another difference is the explicit mention of `.toc`. This scheme gives the compiler control within the translation unit.
With the traditional GOT scheme, input files do not mention `.got`. The compiler does not control how the linker will layout `.got`.
Well, I disagree with the presumed advantage of `.toc`: the compiler does not know the global information, and the translation unit local layout may not be ideal.
A linker is better placed to do such link-time optimization.

A `.tc` directive is a fancy way to produce a relocation of type `R_PPC64_ADDR64`.
If the linker decides to create a TOC entry, the entry will be a link-time constant (`-no-pie`) or be associated with a dynamic relocation (`-pie` or `-shared`).

Alan Modra

> A feature of ld.bfd is that input .toc (and .got) sections matching one linker input section statement may be sorted, to put entries used by small-model code first, near the toc base.  This is why scripts for powerpc64 normally use *(.got .toc) rather than *(.got) *(.toc), since the first form allows more freedom to sort.

#### Tail call

In the above example, `bar` tail calls `foo`, where `foo` is non-local and uses TOC (`st_other>0`).
Most other architectures just need one branch instruction. However, the TOC ABI needs 2 extra instructions (for setting up the TOC pointer) before the branch instruction.
```asm
bar:
.Lfunc_begin1:
.Lfunc_gep1:
  addis 2, 12, .TOC.-.Lfunc_gep1@ha
  addi 2, 2, .TOC.-.Lfunc_gep1@l
.Lfunc_lep1:
.localentry     bar, .Lfunc_lep1-.Lfunc_gep1
  b foo
  nop
```

**Case 1: `foo` is non-preemptible**

The linker will resolve `b foo` to the local entry of `foo`.
The extra instructions for `bar` ensures that: even if `bar` is called without setting up the TOC pointer, `foo` will still get the correct TOC pointer.

**Case 2: `foo` is preemptible**

A call stub will be needed and the `nop` instruction following `call foo` will be replaced with `ld 2,4(1)` by the linker.
We need a `nop` even if this is a tail call.

If want to preserve tail call semantics (e.g. [this cross-module musttail issue](https://github.com/llvm/llvm-project/issues/63214)), we need to avoid this `nop => ld 2,4(1)` rewrite by the linker.
This is doable: all callers of `bar` have to restore TOC even if `bar` is non-preemptible.
While this can be implemented by `.localentry bar, 1`, which sets bit 5 in `st_other` (indicating zero offset between GEP and LEP, and r2 is caller-saved for callers), this has poor toolchain implementation.

### TOC-indirect to TOC-relative optimization

See [All about Global Offset Table#GOT optimization](/blog/2021-08-29-all-about-global-offset-table#got-optimization).

## Procedure Linkage Table

### PPC32 PLT

_Power Architecture® 32-bit Application Binary Interface Supplement 1.0 - Linux® & Embedded_ specifies two PLT ABIs: BSS-PLT and Secure-PLT.

BSS-PLT is the older method, which is now obsolete.
While `.plt` on other architectures is created by the linker, BSS-PLT lets ld.so generate the PLT entries.
This has the advantage that the section can be made `SHT_NOBITS` and therefore not occupy file size.
However, the downside is the security concern of writable and executable memory pages.
Even worse, as an implementation issue, GNU ld places `.plt` in the text segment, making the whole text segment is writable and executable.
This renders `-z relro -z now` ineffective.

In the newer Secure-PLT ABI, `.plt` holds the table of function addresses.
`.plt` is like `.got.plt` for other architectures.

The linker synthesizes `.glink`, which is like `.plt` for other architectures. Unlike most architectures, `.glink` has a footer rather than a header.
Each PLT entry is either `b footer` or a nop falling through to the footer. In ld.lld, we only use `b footer` for simplicity.
See <https://reviews.llvm.org/D75394> for `PPC32GlinkSection` in ld.lld.

```asm
000102b4 <.glink>:
  b 0x102c0 <.glink+0xc>
  b 0x102c0 <.glink+0xc>
  b 0x102c0 <.glink+0xc>
  addis 11, 11, 0          # start of the resolver
  mflr 0
  bcl 20, 31, 0x102cc <.glink+0x18>
  addi 11, 11, 24
  mflr 12
  mtlr 0
  sub     11, 11, 12
  addis 12, 12, 1
  lwz 0, 184(12)
  lwz 12, 188(12)
  mtctr 0
  add 0, 11, 11
  add 11, 0, 11
  bctr
  nop
  nop
```

For non-PIC code, a possibly preemptible branch uses the relocation type `R_PPC_REL24`.
```asm
  bl foo  # R_PPC_REL24
  bl foo  # R_PPC_REL24
```

If the call target is preemptible, the linker creates a non-PIC call stub and redirects the caller's branch instruction to the call stub.
The non-PIC call stub will use absolute addressing to load `.plt[n]` into r11 (call-clobbered) and branch there.
This behavior is different from most other architectures where the caller can branch directly to the PLT entry.
```asm
  bl 00000000.plt_call32.f
  bl 00000000.plt_call32.f
  ...

00000000.plt_call32.f:
  lis 11, .plt[n]@ha
  lwz 11, .plt[n]@l(11)
  mtctr 11
  bctr
```

For PIC code, a branch to a possibly preemptible target uses `R_PPC_PLTREL24` as the PLT-generating relocation type.
The addend encodes r30 set up by the caller. Yes, this is unusual.

* For `-fpic` and `-fpie`, the addend is 0.
* For `-fPIC` and `-fPIE`, the addend is 0x8000. Linking this relocatable object file in `-r` mode may increase the addend.

When calling a function, if the target is preemptible, the linker creates a PIC call stub and redirects the caller's branch instruction to the call stub.
GNU ld names small PIC call stubs as `*.plt_pic32.*` and large PIC call stubs as `*.got2.plt_pic32.*`.
ld.lld follows the naming convention.

A call stub knows the value of r30 (GOT base) set up by the caller.
The distance from `.plt[n]` to r30 is a constant.
The call stub computes the address of `.plt[n]`, loads the entry, and branches there.
```asm
00000000.plt_pic32.f:
  ## If the GOT offset is beyond 64KiB
  addis 11, 30, .plt[n]-_GLOBAL_OFFSET_TABLE_@ha(30)
  lwz 11, .plt[n]-_GLOBAL_OFFSET_TABLE_@l(30)
  mtctr 11
  bctr

  ## If the GOT offset is within 64KiB
  # lwz 11, .plt[n]-_GLOBAL_OFFSET_TABLE_(30)
  # mtctr 11
  # bctr
  # nop

00000000.got2.plt_pic32.f:
  ## .got2 refers to the copy belonging to the current translation unit.
  ## Different translation units have to use different stubs.
  addis 11, 30, .plt[n]-(.got2+0x8000)(30)
  lwz 11, .plt[n]-(.got2+0x8000)@l(30)
  mtctr 11
  bctr

  ## The case when the GOT offset is within 64KiB is similar to plt_pic32.f.
```

While we have a working solution, if we revisit the scheme, we will find that setting up r30 is extremely expensive.
A trivial tail call example (`void foo() { bar(); }`) needs numerous instructions:

```asm
<foo>:
  stwu 1, -16(1)      # allocate stack
  mflr 0
  bcl 20, 31, 0x1bc   # set lr to PC
  stw 30, 8(1)        # save r30 which is used as the GOT base
  mflr 30
  addis 30, 30, 2     # high 16 bits of the GOT base (.got2+0x8000)
  stw 0, 20(1)        # save lr (copied to r0)
  addi 30, 30, 32140  # low 16 bits of the GOT base (.got2+0x8000)
  bl 0x1f0
  lwz 0, 20(1)
  lwz 30, 8(1)
  addi 1, 1, 16
  mtlr 0
  blr
```

### PPC64 ELFv2 PLT

`.glink` is like `.plt` for other architectures and has a header of 60 bytes.
Each PLT entry consists of one instruction `b .plt`. The PLT header subtracts the address of the first PLT entry from `r12` to compute the PLT index.

An unconditional branch instruction `b`/`bl` may produce a relocation of either `R_PPC64_REL24` or `R_PPC64_REL24_NOTOC`.
`R_PPC64_REL24` indicates that the caller uses TOC.
`R_PPC64_REL24_NOTOC` indicates that the caller does not use TOC or preserve r2 ([`DT_PPC64_OPT`](https://reviews.llvm.org/D150631) will be set to 2).

A conditional branch instruction may produce a relocation of type `R_PPC64_REL14`.

All of `R_PPC64_REL14`, `R_PPC64_REL24`, and `R_PPC64_REL24_NOTOC` are PLT-generating relocation types.
If a PLT entry is needed, the linker will create a traditional or PC-relative PLT call stub, and redirect the caller's branch instruction to the call stub.
This behavior is different from most other architectures where the caller can branch directly to the PLT entry.
The inefficiency comes from maintaining r2 and r12 for TOC.

There is no `R_PPC64_REL14_NOTIC`. `R_PPC64_REL14` used by conditional branches is generally not used for function calls.

Below I will describe call stubs for TOC/NOTOC interop and for range extension in detail.

## Thread Local Storage

Both PPC32 and PPC64 use a variant of TLS Variant I: the static TLS blocks are placed above the thread pointer. The thread pointer points to the end of the thread control block.

The linker performs TLS optimization.

See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage).

### Workaround for old IBM XL compilers

`R_PPC64_TLSGD` or `R_PPC64_TLSLD` is required to mark `bl __tls_get_addr` for General Dynamic/Local Dynamic code sequences.

```asm
addis r3, r2, x@got@tlsgd@ha # R_PPC64_GOT_TLSGD16_HA
addi r3, r3, x@got@tlsgd@l   # R_PPC64_GOT_TLSGD16_LO
bl __tls_get_addr(x@tlsgd)   # R_PPC64_TLSGD followed by R_PPC64_REL24
nop
```

However, there are two deviations from the above:

1. direct call to `__tls_get_addr`. This is essential to implement rtld in glibc/musl/FreeBSD.

```asm
bl __tls_get_addr
nop
```

This is only used in a `-shared` link, and thus not subject to the GD/LD to IE/LE relaxation issue below.

2. Missing `R_PPC64_TLSGD`/`R_PPC64_TLSGD` for compiler generated TLS references

According to Stefan Pintille, "In the early days of the transition from the ELFv1 ABI that is used for big endian PowerPC Linux distributions to the ELFv2 ABI that is used for little endian PowerPC Linux distributions, there was some ambiguity in the specification of the relocations for TLS.
The GNU linker has implemented support for correct handling of calls to `__tls_get_addr` with a missing relocation.
Unfortunately, we didn't notice that the IBM XL compiler did not handle TLS according to the updated ABI until we tried linking XL compiled libraries with LLD."

It is unfortunate but in short ld.lld needs to work around the old IBM XL compiler issue.
Otherwise, if the object file is linked in `-no-pie` or `-pie` mode, the result will be incorrect because the 4 instructions are partially rewritten (the latter 2 are not changed).

## PPC64 ELFv2 TOC caller

A caller using TOC marks its function calls with relocation type `R_PPC64_REL24`.

The caller expects that r2 does not change while the callee may alter r2.
To address the issue, the compiler and the linker collaborate to preserve r2.

For a call target which may resolve to a different translation unit (e.g. non-definition declaration, hidden visibility definition), the compiler inserts a NOP after the branch instruction.
A call target guaranteed to resolve to the current translation unit (e.g. internal linkage) does not need a NOP since r2 will not change.
```asm
caller:
  bl foo
  nop                # may become `ld 2, 24(1)`
  bl nonpreemptible
  blr
```

Note: An external linkage hidden visibility call target needs a NOP as well in case the callee clobbers r2 if it does not maintain the TOC pointer.

### TOC caller and preemptible callee

If the callee is preemptible, the caller and the callee may be in different components.

* If the callee uses TOC, it may change r2 to the TOC base of its component.
* If the callee uses PC-relative addressing, it may treat r2 as caller-saved and clobber r2.

The linker creates a PLT call stub to save r2 in the caller stack frame, and patches the `nop` to `ld 2, 24(1)` to restore r2.

```asm
<caller>:
  bl __plt_foo
  ld 2, 24(1)        # restore r2
  bl nonpreemptible
  bl nonpreemptible

<__plt_foo>:
  std 2, 24(1)       # save r2
  addis 12, 2, ...
  ld 12, ...(12)     # load .plt[n]
  mtctr 12
  bctr               # jump to the PLT entry
```

### TOC caller and non-preemptible callee with `localentry=1`

A non-TOC callee may or may not preserve r2.
Its `.localentry` value may be 0 or 1, where 1 indicates that r2 may be clobbered.

Similar to the preemptible callee case, the linker creates a call stub to save r2, and patches the `nop` to `ld 2, 24(1)` to restore r2.

```asm
<caller>:
  bl __toc_save_foo
  ld 2, 24(1)        # restore r2
  bl nonpreemptible
  blr

<__toc_save_foo>:
  std 2, 24(1)       # save r2
  b foo              # jump to the callee
```

If the call stub cannot reach the call target with a single `b` instruction, the linker will try computing the target address with `addis+addi`.
```asm
<__toc_save_far>:
  std 2, 24(1)       # save r2
  addis 12, 2, ...
  addi 12, 12, ...
  mtctr 12
  bctr               # jump to the callee
```

If `addis+addi` cannot reach the call target, the linker will store the target address in a `.branch_lt` entry and perform an indirect branch.
```asm
<__toc_save_farther>:
  std 2, 24(1)       # save r2
  addis 12, 2, ...
  ld 12, ...(12)     # load .branch_lt[n]
  mtctr 12
  bctr               # jump to the callee
```

## PPC64 ELFv2 non-TOC caller

A caller not using TOC marks its function calls with the relocation type `R_PPC64_REL24_NOTOC`.

```asm
caller:
  bl foo@notoc
  blr
```

Here is a test about a non-TOC caller and a TOC callee.
In `a0` and `a1`, the callee `foo` is non-preemtpbile while in `a2`, `foo` is preemptible.

```sh
echo 'int x = 42; void foo(); int main() { foo(); }' > a.c
printf '#include <stdio.h>\nextern int x; void foo() { printf("%%d\\n", x); }' > b.c
sed 's/^        /\t/' > Makefile <<'eof'
.MAKE.MODE := meta curDirOk=true
CC := /tmp/Rel/bin/clang --target=powerpc64le-linux-gnu
LDFLAGS := -fuse-ld=lld -Wl,--dynamic-linker=/usr/powerpc64le-linux-gnu/lib64/ld64.so.2,-rpath=/usr/powerpc64le-linux-gnu/lib -Wl,--no-power10-stubs

run: a0 a1 a2
        qemu-ppc64le-static -cpu power10 ./a0
        qemu-ppc64le-static -cpu power10 ./a1
        qemu-ppc64le-static -cpu power10 ./a2

a0: a.o b.o
        ${LINK.c} $> -o $@

a1: a.o b.o
        ${LINK.c} -r $> -o $@.ro
        ${LINK.c} $@.ro -o $@

a2: a.o b.so
        ${LINK.c} a.o ./b.so -o $@

a.o: a.c
        ${CC} -mcpu=power10 -c $>

b.so: b.o
        ${LINK.c} -shared $> -o $@
eof
```

Invoke `bmake` to run the test.

### Non-TOC caller and preemptible callee

The callee may or may not use TOC.
If the callee uses TOC and has a `.localentry` value larger than 1, its global entry point requires that r12 is set to the function entry address by the caller.

The linker creates a PC-relative PLT call stub to set r12 in case the callee needs r12.

```asm
<caller>:
  bl __plt_pcrel_foo
  blr

<__plt_pcrel_foo>:
  pld 12, .plt[n]@pcrel   # load .plt[n]
  mtctr 12
  bctr                    # jump to the PLT entry
```

If we don't use Power10 `pld` (`--power10-stubs=no`), we will need more instructions:
```asm
<__plt_pcrel_foo>:
  mflr 12                 # save lr
  bcl 20, 31, .+4
  mflr 11                 # r11 = current location
  mtlr 12                 # restore lr
  addis 12, 11, offset@ha
  ld 12, offset@l(12)     # load .plt[n]
  mtctr 12
  bctr                    # jump to the PLT entry
```

### Non-TOC caller and non-preemptible TOC callee

A non-preemptible callee may or may not use TOC.

* If the callee doesn't use TOC, the branch instruction can point to the target directly.
* If the callee uses TOC, use the preemptible callee case.

```asm
<caller>:
  bl __gep_setup_foo
  blr

<__gep_setup_foo>:
  paddi 12, 0, foo@pcrel  # compute target address
  mtctr 12
  bctr                    # jump to target
```

If we don't use Power10 `paddi` (`--power10-stubs=no`), we will need more instructions.
```asm
<__gep_setup_foo>:
  mflr 12
  bcl 20, 31, .+4
  mflr 11
  mtlr 12
  addis 12, 11, offset@ha
  addi 12, 12, offset@l
  mtctr 12
  bctr
```

### IPLT code sequence for non-preemptible IFUNC

Non-preemptible IFUNC are placed in `.glink` on PPC64.
If there is a non-GOT non-PLT relocation, for pointer equality, we change the type of the symbol from `STT_IFUNC` and `STT_FUNC` and bind it to the `.glink` entry.

On PPC64 ELFv2, every `bl` instruction in `.glink` is associated with a `.plt` entry relocated by `R_PPC64_JUMP_SLOT`.
An IPLT does not have an associated `R_PPC64_JUMP_SLOT`, so we cannot use `bl` in `.iplt`.
Instead, we create a regular TOC call stub.

A non-preemptible ifunc implementation may not save the TOC pointer, so if another DSO defines an ifunc resolver which resolves to this implementation, calling that ifunc will not set the TOC pointer correctly.
This is the restriction described by <https://sourceware.org/glibc/wiki/GNU_IFUNC> (though on many architectures it works in practice):

> Requirement (a): Resolver must be defined in the same translation unit as the implementations.

See <https://reviews.llvm.org/D71509>.

## Range extension thunks

On PPC32, an unconditional branch instruction `b`/`bl` has a range of +-32MiB and may use 3 relocation types: `R_PPC_LOCAL24PC`, `R_PPC_REL24`, and `R_PPC_PLTREL24`.
If the target is not reachable from the instruction location, a range extension thunk will be used.
`R_PPC_LOCAL24PC` is a useless relocation. All occurrences can be replaced with `R_PPC_REL24`.

On PPC64, an unconditional branch instruction `b`/`bl` has a range of +-32MiB and may use `R_PPC64_REL24` or `R_PPC64_REL24_NOTOC`.
The aforementioned call stubs for TOC/NOTOC interop have handled many long branches.
The cases which haven't been handled are:

* TOC caller and non-preemptible TOC callee
* non-TOC caller and non-preemptible non-TOC callee

ld.lld only has an implementation for the first case. After linking a caller may look like:

```asm
<caller>:
  bl __long_branch_nonpreemptible
  blr

<__long_branch_nonpreemptible>:
  addis 12, 2, offset@ha
  ld 12, offset@l(12)     # load .branch_lt[n]
  mtctr 12
  bctr                    # jump to the target
```

The branch target of a thunk may be a PLT entry.

## GPR Save and restore functions

GPR Save and Restore Functions defines some special functions which may be referenced by GCC produced assembly (LLVM does not reference them).

With GCC -Os, when the number of call-saved registers exceeds a certain threshold, GCC generates `_savegpr[01]_{14..31}` and `_restgpr[01]_{14..31}` calls and expects the linker to define them. See <https://sourceware.org/pipermail/binutils/2002-February/017444.html> and <https://sourceware.org/pipermail/binutils/2004-August/036765.html>.

This is weird because `libgcc.a` would be the natural place. However, the linker generation approach has the advantage that the linker can generate multiple copies to avoid long branch thunks.
I don't consider the advantage significant enough to complicate ld.lld's trunk implementation, so I take a simple approach.

* Check whether `_savegpr0_{14..31}` are used
* If yes, define needed symbols and add an InputSection with the code sequence.

## `--emit-relocs`

In GNU ld, unlike aarch64 and x86, the powerpc port converts relocation types.
For example, a TOC-indirect to TOC-relative optimization uses a pair of relocations `R_PPC64_TOC16_HA(.toc)+R_PPC64_TOC16_LO_DS(.toc)`.
After optimization, they will become `R_PPC64_TOC16_HA(sym)+R_PPC64_TOC16_LO(sym)`. The `R_PPC64_TOC16_HA` relocation is present even if the first instruction is converted to a NOP.

A general-dynamic TLS model code sequence may use relocations `R_PPC64_GOT_TLSGD16_HA+R_PPC64_GOT_TLSGD16_LO+R_PPC64_TLSGD+R_PPC64_REL24`.
After optimization, they will become:

* `R_PPC64_NONE+R_PPC64_TPREL16_HA+R_PPC64_TPREL16_LO+R_PPC64_NONE` after general-dynamic to local-exec TLS optimization.
* `R_PPC64_GOT_TPREL16_HA+R_PPC64_GOT_TPREL16_LO_DS+R_PPC64_NONE+R_PPC64_NONE` after general-dynamic to initial-exec TLS optimization.

```asm
addis 3,2,x@got@tlsgd@ha
addi 3,3,x@got@tlsgd@l
bl __tls_get_addr(x@tlsgd)
nop

=>

addis or nop                     # R_PPC64_GOT_TLSGD16_HA(x)
addi 3, 2, ...                   # R_PPC64_GOT_TLSGD16_LO(x)
bl thunk_for___tls_get_addr_opt  # R_PPC64_TLSGD(x), R_PPC64_REL24(__tls_get_addr_opt@GLIBC_2.22)
nop
```

GNU ld even annotates stubs with relocation types, e.g.
```
0000000000002000 <0000001b.plt_branch.1c:2>:
    2000: ff ff 82 3d   addis 12, 2, -1
                0000000000002000:  R_PPC64_TOC16_HA     *ABS*+0x20020d0
    2004: d0 7e 8c e9   ld 12, 32464(12)
                0000000000002004:  R_PPC64_TOC16_LO_DS  *ABS*+0x20020d0
    2008: a6 03 89 7d   mtctr 12
    200c: 20 04 80 4e   bctr
                ...

0000000000002030 <0000001b.long_branch.1c:2+8>:
    2030: f8 ff ff 49   b 0x2002028 <high_target+0x8>
                0000000000002030:  R_PPC64_REL24        *ABS*+0x2002028
                ...
```

## REL and RELA

`.plt` and `.branch_lt` are of type `SHT_NOBITS` and may need dynamic relocations with a non-zero addend.
This makes REL infeasible for the two sections. Therefore `-z rel` cannot be used when either `.plt` or `.branch_lt` is required.

## PPC64 ELFv1 function descriptors

In the ELFv1 ABI, the TOC pointer (r2) is set by the caller instead of the callee.
This means external function calls need to update both the program counter and the TOC pointer.

To achieve this, each externally visible function has a dedicated function descriptor stored in the `.opd` section.
This descriptor contains three doublewords:

* Function entry point address.
* TOC base address.
* Environment pointer: Used by languages such as Pascal and PL/1, but zero for C/C++.

The call site uses a BL instruction followed by a NOP instruction.
When the callee is preemptible, the linker creates a PLT call stub and patches the NOP to `ld r2, 40(r1)` to restore the TOC pointer, similar to ELFv2.
The PLT stub saves the current TOC pointer onto the stack, loads the function entry point and TOC pointer, and jumps to the function entry point.

```asm
  bl 0000001b.plt_call.foo
  ld r2,40(r1)               # restore TOC pointer
  bl 0000001b.plt_call.foo
  ld r2,40(r1)               # restore TOC pointer

0000001b.plt_call.foo:
  std     r2,40(r1)          # save r2
  ld      r12,val(r2)        # load the function entry point
  mtctr   r12
  ld      r2,val+8(r2)       # set TOC pointer
  bctr                       # jump
```

r2 is call-preserved, which requires TOC load/restore in PLT call stubs and therefore [suppresses preemptible tail calls](/blog/2021-09-19-all-about-procedure-linkage-table#got-setup-is-expensive-without-pc-relative-addressing). This is a design flaw.
Technically, non-preemptible tail calls are available, but GCC doesn't seem to implement that.

Calling a non-preemptible function uses a `R_PPC64_REL24` relocation, which should be resolved to the function entry point by the linker.

The value (`st_value`) of a function symbol is actually the address of the function descriptor in the `.opd` section.
For most relocation types unrelated to function calls, the linker can handle function and data symbols in the same way.
However, for `R_PPC_REL24`, which is used by function calls, the linker needs to resolve the relocation to reference the function entry point for a non-preemptible function.

The compiler generates `.opd` sections for each translation unit to be combined by the linker.

```asm
.section        ".opd","aw"
.align 3
foo:
  .quad   .L.foo,.TOC.@tocbase,0
bar:
  .quad   .L.bar,.TOC.@tocbase,0
```

There is another design flaw that the compiler generates `.opd` sections, which are combined by the linker.
The default garbage collection rules would retain the whole `.opd` if any function descriptor is live.
Instead, we should treat each function descriptor as a subsection and discardable on its own.
In addition, two relocations associated with each function descriptor are wasteful.

If we drop the advantage that a function symbol value points to the function descriptor and fix a few design mistakes:

* Make r2 call-clobbered.
* Remove the unused environment pointer.
* Introduce a relocation to identify a function descriptor and remove compiler-generated `.opd`.

We will essentially get a [FDPIC ABI](/blog/2024-02-20-mmu-less-systems-and-fdpic)!
