layout: post
title: Relocation overflow and code models
author: MaskRay
tags: [gcc,linker]
---

When linking an oversized executable, it is possible to encounter errors such as ``relocation truncated to fit: R_X86_64_PC32 against `.text'`` (GNU ld) or `relocation R_X86_64_PC32 out of range` (ld.lld).
These diagnostics are a result of the relocation overflow check, a feature in the linker.

```
% gcc -fuse-ld=bfd @response.txt
...
a.o: in function `_start':
(.text+0x0): relocation truncated to fit: R_X86_64_PC32 against `.text'
% gcc -fuse-ld=lld @response.txt
ld.lld: error: a.o:(.text+0x0): relocation R_X86_64_PC32 out of range: -2147483649 is not in [-2147483648, 2147483647]; references section '.text'
```

This article aims to explain why such issues can occur and provides insights on how to mitigate them.

<!-- more -->

## Static linking

In this section, we will deviate slightly from the main topic to discuss static linking.
By including all dependencies within the executable itself, it can run without relying on external shared objects.
This eliminates the potential risks associated with updating dependencies separately.

Certain users prefer static linking or mostly static linking for the sake of deployment convenience and performance aspects:

* Link-time optimization is more effective when all dependencies are known. Providing shared object information during executable optimization is possible, but it may not be a worthwhile engineering effort.
* Profiling techniques are more efficient dealing with one single executable.
* The traditional ELF dynamic linking approach incurs overhead to support [symbol interposition](/blog/2021-05-16-elf-interposition-and-bsymbolic).
* Dynamic linking involves PLT and GOT, which can introduce additional overhead. Static linking eliminates the overhead.
* Loading libraries in the dynamic loader has a time complexity `O(|libs|^2*|libname|)`. The existing implementations are designed to handle tens of shared objects, rather than a thousand or more.

Furthermore, the current lack of techniques to partition an executable into a few larger shared objects, as opposed to numerous smaller shared objects, exacerbates the overhead issue.

In scenarios where the distributed program contains a significant amount of code (related: software bloat), employing full or mostly static linking can result in very large executable files.
Consequently, certain relocations may be close to the distance limit, and even a minor disruption (e.g. add a function or introduce a dependency) can trigger relocation overflow linker errors.

## Relocation overflow

We will use the following C program to illustrate the concepts.
```c
int var0; // known non-preemptible if -fno-pic or -fpie
extern int var1; // possibly-preemptible
int callee();
int caller() { return callee() + var0 + var1; }
```

The generated x86-64 assembly may appear as follows, with comments indicating the relocation types associated with each instruction:
```asm
# gcc -S -O1 -fpie -mno-direct-extern-access -masm=intel a.c
.globl caller
caller:
  call callee@PLT                          # R_X86_64_PLT32
  add  eax, DWORD PTR [rip + var0]         # R_X86_64_PC32; load from &var0
  mov  rdx, QWORD PTR var1@GOTPCREL[rip]   # R_X86_64_REX_GOTPCRELX; rdx = .got[n] = &var1
  add  eax, DWORD PTR [rdx]                # load from &var1

.bss
.globl var0
var0: .long 0
```

You see that I specify `-mno-direct-extern-access` for some assembly output.
The option is to prefer GOT-generating code sequences for possibly-preemptible data symbols.
See `-fdirect-access-external-data` on [Copy relocations, canonical PLT entries and protected visibility](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected).

All of `R_X86_64_PLT32`, `R_X86_64_PC32`, `R_X86_64_REX_GOTPCRELX`, and `R_X86_64_CODE_4_GOTPCRELX` have a value range of `[-2**31,2**31)`. If the referenced symbol is too far away from the relocated location, we may get a relocation overflow.

In practice, relocation overflows due to code referencing code are not common.
The more frequent occurrences of overflows involve the following categories (where we use `.text` to represent code sections, `.rodata` for read-only data, and so on):

* `.text <-> .rodata`
* `.text <-> .eh_frame`: `.eh_frame` has 32-bit offsets. 64-bit code offsets are possible, but I don't know if an implementation exists.
* `.text <-> .data/.bss`
* `.rodata <-> .data/.bss`

In many programs, `.text <-> .data/.bss` relocations have the most stringent constraints.
Overflows due to `.text <-> .rodata` relocations are possible but rare (although I have encountered such issues in the past).

`.rodata <-> .data/.bss` overflows are generally infrequent. However, caution must be exercised when working with metadata `.quad label-.` instead of `.long label-.`.
Such issues can be easily addressed on the compiler side.

On x86-64, linkers optimize some GOT-indirect instructions (`R_X86_64_REX_GOTPCRELX`; e.g. `movq var@GOTPCREL(%rip), %rax`) to PC-relative instructions.
The distance between a code section and `.got` is usually smaller than the distance between a code section and `.data`/`.bss`.
ld.lld's one-pass relocation scanning scheme has a limitation: if it decides to suppress a GOT entry and it turns out that optimizing the instruction will lead to relocation overflow, the decision cannot be reverted.
It should be easy to work around the issue with `-Wl,--no-relax`.

The current section layout of ld.lld is as follows:
```
.rodata
.text
.plt
.data.rel.ro
.got
.data
.bss
```

One notable distinction from GNU ld is that `.rodata` precedes `.text`. This ordering decreases the distance between `.text` and `.data`/`.bss`, thereby alleviating relocation overflow pressure for references from `.text` to `.data`/`.bss`.

For handling `.eh_frame`, I suggest compiling with the [`-fno-asynchronous-unwind-tables`](/blog/2020-11-08-stack-unwinding) option.
`.eh_frame` is used by runtime support for C++ exceptions. Many programs don't utilitize exceptions, making `.eh_frame` non-mandatory.
For profiling purposes, the limitations of the `.eh_frame` format have become apparent, and it is not the most suitable unwinding format.

## x86-64 code models

The x86-64 psABI defines multiple code models. Here is a summary:

* Small: symbols are required to be located within the range `[0, 2**31 - 2**24)`. Use 32-bit PC-relative or absolute addressing
* Kernel: similar to the small code model, but symbols are within the high end range
* Medium: keep using 32-bit offsets for code and GOT, but split data sections into 2 parts: regular and large. Large data can be more than 2GiB away
* Large: all of code, GOT, and data can be more than 2GiB away

### x86-64 medium code model

The medium code model maintains the assumption that code and the GOT is within the ±2GiB range from the program counter, while allowing data to be located outside of that range.
Data that resides outside the range is placed in large data sections such as `.lrodata`, `.ldata`, and `.lbss`, as well as `.ldata`'s variants like `.ldata.rel`, `.ldata.rel.local`, `.ldata.rel.ro`, and `.ldata.rel.ro.local`.

These sections may have the `SHF_X86_64_LARGE` flag.

`-mlarge-data-threshold=` decides whether a data section should be treated as large.

Accessing code and GOT-indirect data has the same code sequence as the small code model.

To access data without GOT indirection (usually a known non-preemptible symbol, e.g. `var0`), GCC obtains the address of the GOT base symbol `_GLOBAL_OFFSET_TABLE_`, then adds the offset from `_GLOBAL_OFFSET_TABLE_` to the symbol.

```asm
# gcc -S -O1 -fpie -mcmodel=medium -mlarge-data-threshold=3 -masm=intel a.c
call    callee@PLT                        # R_X86_64_PLT32
lea     rdx, _GLOBAL_OFFSET_TABLE_[rip]   # rdx = &_GLOBAL_OFFSET_TABLE_
movabs  rcx, OFFSET FLAT:var0@GOTOFF      # R_X86_64_GOTOFF64; rcx = &var0 - &_GLOBAL_OFFSET_TABLE_
add     eax, DWORD PTR [rcx+rdx]          # load from &var0
mov     rdx, QWORD PTR var1@GOTPCREL[rip] # R_X86_64_REX_GOTPCRELX; rdx = .got[n] = &var1
add     eax, DWORD PTR [rdx]              # load from &var1
```

The relocation type `R_X86_64_GOTOFF64`, and another type that we will see below, `R_X86_64_PLTOFF64`, have confusing names.
If you haven't encountered these relocation types before, don't think too hard how they are related to the "GOT" concept.

For position-dependent code, accessing data without GOT indirection is simplified as we can just use abolute addressing.
```asm
# gcc -S -O1 -fno-pic -mcmodel=medium -mno-direct-extern-access -mlarge-data-threshold=3 -masm=intel a.c
call    callee                            # R_X86_64_PLT32
mov     edx, eax
movabs  eax, DWORD PTR [var0]             # R_X86_64_64; load from &var0
add     eax, edx
mov     rdx, QWORD PTR var1@GOTPCREL[rip] # R_X86_64_REX_GOTPCRELX; rdx = .got[n] = &var1
add     eax, DWORD PTR [rdx]              # load from &var1
```

Note that we use `-mno-direct-extern-access` in the above snippet, otherwise GCC's x86-64 port doesn't use GOT indirection for `var1`.
This will lead to a copy relocation if `var1` is defined in a shared object.

### x86-64 large code model

In the large code model, we no longer assume that GOT is within the ±2GiB range from the program counter, so `lea rdx, _GLOBAL_OFFSET_TABLE_[rip]` cannot be used.
An extra `movabs` instruction is needed to obtain the address of `_GLOBAL_OFFSET_TABLE_`.

Similarly, for a function call, we no longer assume that the address of the function or its PLT entry is within the ±2GiB range from the program counter, so `call callee` cannot be used.

Actually, `call callee` can still be used if we implement range extension thunks in the linker, unfortunately GCC/GNU ld did not pursue this direction.

```asm
# gcc -S -O1 -fpie -mcmodel=large -masm=intel
.L2:
lea     r15, .L2[rip]                     # r15 = &.L2
movabs  r11, OFFSET FLAT:_GLOBAL_OFFSET_TABLE_-.L2  # R_X86_64_GOTPC64; r11 = &_GLOBAL_OFFSET_TABLE_ - &.L2
add     r15, r11                          # r15 = &_GLOBAL_OFFSET_TABLE_
mov     eax, 0
movabs  rdx, OFFSET FLAT:callee@PLTOFF    # R_X86_64_PLTOFF64; rdx = (the address of callee or its PLT) - &_GLOBAL_OFFSET_TABLE_
add     rdx, r15                          # rdx = the address of callee or its PLT
call    rdx                               # indirectly call callee
movabs  rdx, OFFSET FLAT:var0@GOTOFF      # R_X86_64_GOTOFF64; rdx = &var0 - &_GLOBAL_OFFSET_TABLE_
add     eax, DWORD PTR [rdx+r15]          # load from &var0
movabs  rdx, OFFSET FLAT:var1@GOT         # R_X86_64_GOT64; rdx = &.got[n] - _GLOBAL_OFFSET_TABLE_
mov     rdx, QWORD PTR [r15+rdx]          # rdx = .got[n] = &var1
add     eax, DWORD PTR [rdx]              # load from &var1
```

For position-dependent code, GCC uses absolute addressing and does not rely on `_GLOBAL_OFFSET_TABLE_`.
The code sequence for accessing data symbols is the same as that used in the position-dependent medium code model.
```asm
# gcc -S -O1 -fno-pic -mcmodel=large -masm=intel a.c
movabs  rdx, OFFSET FLAT:callee           # R_X86_64_64; obstain the address of callee; canonical PLT entry if defined in a DSO
call    rdx
mov     edx, eax
movabs  eax, DWORD PTR [var0]             # R_X86_64_64; load from &var0
add     eax, edx
movabs  rdx, QWORD PTR [var1]             # R_X86_64_64; load from &var1; copy relocation if var1 is defined in a DSO
add     eax, edx
```

### x86-64 linker requirement

Linkers are expected to recognize these large data sections and place them in appropriate locations.
GNU ld uses the following section layout in its internal linker scripts:
```
.text
.rodata   # if -z separate-code, MAXPAGESIZE alignment
RELRO     # DATA_SEGMENT_ALIGN
.data     # DATA_SEGMENT_RELRO_END
.bss
.lbss
.lrodata  # MAXPAGESIZE alignment
.ldata    # MAXPAGESIZE alignment
```

GNU ld places `.lbss`, `.lrodata`, `.ldata` after `.bss` and inserts 2 MAXPAGESIZE alignments for `.lrodata` and `.ldata`.

`.lbss` is placed immediately after `.bss`, creating a single BSS, i.e. a read-write `PT_LOAD` program header with `p_filesz<p_memsz`.
The file image in the segment is smaller than the memory image.
When the dynamic loader creates the memory image for the `PT_LOAD` segment, it will set the byte range `[p_filesz,p_memsz)` to zeros.
However, there is a missing optimization that the `[p_filesz,p_memsz)` portion occupies zero bytes in the object file, preventing overlap between `.lrodata` and BSS in the file image.

ld.lld [from 17 onwards](https://reviews.llvm.org/D150510) uses the following section layout:
```
.lrodata
.rodata
.text     # if --ro-segment, MAXPAGESIZE alignment
RELRO     # MAXPAGESIZE alignment
.data     # MAXPAGESIZE alignment
.bss
.ldata    # MAXPAGESIZE alignment
.lbss
```

This layout saves one `PT_LOAD` segment than GNU ld's.
However, this layout is less ideal for `-fno-pic` code.
Small code model `-fno-pic` code has `R_X86_64_32S` relocations with a range of `[0,2**31)` (if we ignore the negative area).
Placing `.lrodata` earlier exerts relocation pressure on such code.
Non-x86 64-bit architectures generally have a similar `[0,2**31)` limitation if they use absolute addressing.
I added [`-z lrodata-after-bss`](https://github.com/llvm/llvm-project/pull/81224) to switch to the GNU ld layout.

In both section layouts, `.rodata`, `.text`, `.data`, and `.bss` are not interspersed with large data sections.

We have mentioned that when using `-mcmodel=medium`, GCC generates both regular and large data sections.
In practice, programs often include a mix of object files built with small and medium/large code models.
The small code model components may come from prebuilt object files (e.g. libc).
The large data sections do not exert relocation pressure on sections in object files built with `-mcmodel=small`.

I have changed GCC 14 to make `-mcmodel=large` respect `-mlarge-data-threshold` and generate large data sections as well.
Previously, the data sections built with `-mcmodel=large` may exert relocation pressure on sections in object files with `-mcmodel=small`.
Please refer to this x86-64-abi discussion: [Large data sections for the large code model](https://groups.google.com/g/x86-64-abi/c/jnQdJeabxiU).

I implemented <https://sourceware.org/PR30592> so with binutils 2.42 you can use `objcopy --set-section-flags .lrodata=alloc,readonly,large --set-section-flags .ldata=alloc,large a.o b.o` to set `SHF_X86_64_LARGE` flag for large data sections.
I filed a feature request [objcopy --set-section-flags: support toggling a flag](https://sourceware.org/PR30623).

The psABI mentions `.ltext`, `.lgot`, and `.lplt`, but GNU ld doesn't do anything with these sections and GCC doesn't generate `.ltext` sections.
I don't think that `.ltext` is necessary. We just need to implement range extension thunks in the rare case that the ±2GiB range of `call callee` becomes a problem.

Placing `.ltext` after `.ldata` may raise an interesting question of whether we should place the linker-defined symbols `_end` and `end`.
If `.lgot` is not implemented and `.got` is reused, placing `.ltext` at the end may end up increasing its distance to `.got`.

## AArch64 code models

Likewise, the psABI defines multiple code models. The small code model allows for a maximum text segment size of 2GiB and a maximum combined span of text and data segments of 4GiB.
For small position-independent code (pic), there is an additional restriction on the size of the Global Offset Table (GOT), which must be smaller than 32KiB.
The maximum combined span of text and data segments is larger than that of x86-64.

Linked object file sizes for AArch64 and x86-64 are comparable, but AArch64 linked object files are more resistant to relocation overflows.

For data references from code, x86-64 uses `R_X86_64_REX_GOTPCRELX`/`R_X86_64_PC32` relocations, which have a smaller range `[-2**31,2**31)`.
In contrast, AArch64 employs `R_AARCH64_ADR_PREL_PG_HI21` relocations, which has a doubled range of `[-2**32,2**32)`.
This larger range makes it unlikely for AArch64 to encounter relocation overflow issues before the binary becomes excessively oversized for x86-64.

```asm
bl      callee               // R_AARCH64_CALL26; [-2**27, 2**27)
adrp    x8, var0             // R_AARCH64_ADR_PREL_PG_HI21; [-2**32,2**32)
ldr     w8, [x8, :lo12:var0] // R_AARCH64_LDST32_ABS_LO12_NC
```

For function calls, the shorter range of `R_AARCH64_CALL26` doesn't matter. The linker will generate [range extension thunks](/blog/2023-03-05-linker-notes-on-aarch64) if `callee` is not directly reachable.
```asm
  b       __AArch64AbsLongThunk_callee
...
__AArch64AbsLongThunk_callee:
  ldr     x16, .+8
  br      x16
  .xword  callee
```

<https://github.com/ARM-software/abi-aa/blob/main/sysvabi64/sysvabi64.rst> summarizes compiler support.
GCC and Clang don't implement `-mcmodel=large` for PIC.
GCC and Clang don't implement `-mcmodel=medium`.
This makes sense as we haven't identified a use case for the unimplemented models yet.

Nevertheless, let's see the code sequence for the position-dependent large code model.
```asm
// aarch64-linux-gnu-gcc -S -O1 -mcmodel=large -fno-pic
bl      callee
adrp    x1, .LC0
ldr     x1, [x1, #:lo12:.LC0]
ldr     w1, [x1]
add     w0, w0, w1
adrp    x1, .LC1
ldr     x1, [x1, #:lo12:.LC1]
ldr     w1, [x1]
add     w0, w0, w1
...
        .align  3
.LC0:
        .xword  .LANCHOR0
        .align  3
.LC1:
        .xword  var1
        .global var0
```

`call callee` is still used since we can use linker-synthesized range extension thunks.
Accessing data symbols requires indirection through an 64-bit absolute symbol.

## Power architecture code models

Object file sizes are usually much larger than x86-64's.

GCC defaults to the medium code model, which is like x86-64 and AArch64's small code model.
Data symbols and TOC/GOT entries are assumed to be within the `[-0x80008000, 0x7fff8000)` range from the TOC.
```asm
# powerpc64le-linux-gnu-gcc -S -O1 -fpie -mcmodel=medium -mcpu=power10 a.c
bl callee@notoc            # R_PPC64_REL24_NOTOC
pld 9,var1@got@pcrel       # R_PPC64_GOT_PCREL34; r9 = &.got[n] for var1
lwz 10,0(9)                # load from &.got[n]
plwz 8,.LANCHOR0@pcrel     # R_PPC64_PCREL34; load from &var0
add 9,10,8
add 9,9,3
extsw 3,9
```

In the large code model, GCC simply uses GOT-indirect addressing to access data symbols, including the non-preemptible ones.
This maintains the assumption that the GOT entries are within the `[-0x80008000, 0x7fff8000)` range from the TOC, so the large code model is more limited than x86-64's.

```asm
addis 9,2,.LC0@toc@ha      # R_PPC64_TOC16_HA; [-0x80008000, 0x7fff8000)
ld 9,.LC0@toc@l(9)         # R_PPC64_TOC16_LO_DS
lwz 8,0(9)
addis 9,2,.LC1@toc@ha      # R_PPC64_TOC16_HA; [-0x80008000, 0x7fff8000)
ld 9,.LC1@toc@l(9)         # R_PPC64_TOC16_LO_DS
lwz 10,0(9)
add 9,10,8
add 9,9,3
extsw 3,9
```

## RISC-V code models

See <https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/388> for a proposal to add a large code model.

## Mitigation

There are several strategies to mitigate relocation overflow issues.

* Make the program smaller by reducing code and data size.
* Partition the large monolithic executable into the main executable and a few shared objects.
* Switch to the medium code model
* Use compiler options such as `-Os`, `-Oz` and link-time optimization that focuses on decreasing the code size.
* For compiler instrumentations (e.g. `-fsanitize=address`, `-fprofile-generate`), move some data to large data sections.
* Use linker script commands [`INSERT BEFORE` and `INSERT AFTER`](/blog/2021-07-04-sections-and-overwrite-sections#insert-before-and-insert-after) to reorder output sections.

## Debug information

For large executables, it is possible to encounter DWARF32 limitation (e.g. `relocation R_X86_64_32 out of range`). I will address this topic in another article.

## Just-in-time compilation

Just-in-time compilation systems generate code and data on the fly.
if the memory mapping backing code is far away from the caller/callee, a large code model may be needed as well.
Since the addresses are known, many uses case can choose the static relocation model and use the non-PIC large code model variant.
