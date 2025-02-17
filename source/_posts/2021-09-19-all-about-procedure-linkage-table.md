layout: post
title: All about Procedure Linkage Table
author: MaskRay
tags: [binutils,linker]
---

Updated in 2024-01.

## Branch target

Many architectures encode a branch/jump/call instruction with PC-relative addressing, i.e. the distance to the target is encoded in the instruction.
In an executable or shared object (called a component in ELF), if the target is bound to the same component, the instruction has a fixed encoding at link time; otherwise the target is unknown at link time and there are two choices:

* text relocation
* indirection

In [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table), I mentioned that linker/loader developers often frowned upon text relocations because the text segment will be unshareable.
In addition, the number of relocations would be dependent on the number of calls, which can be large.

<!-- more -->

```asm
call foo  # R_X86_64_PC32
call foo  # R_X86_64_PC32

=>

# The instructions are patched at runtime.
# On ELF x86-64, the R_X86_64_PC32 relocation type is used.
call ...
call ...
```

Therefore, the prevailing scheme is to add a level of indirection analogous to that provided by the Global Offset Table for data.

## Procedure Linkage Table

ELF uses Procedure Linkage Table to provide the indirection.
Here is an x86-64 example:

```asm
<.text>:
  call foo
  call bar

=>

<.text>:
  call foo
  call bar

<.plt>:
  pushq .got.plt+8(%rip)
  jmpq *.got.plt+16(%rip)
  nop

foo@plt:
  jmpq *.got.plt+24(%rip)
  pushq $0
  jmp .plt

bar@plt:
  jmpq *.got.plt+32(%rip)
  pushq $1
  jmp .plt

<.got.plt>:
  # First three entries are special.
  .quad link-time address of _DYNAMIC  # set by linker
  .quad Obj_Entry used to indicate the current component  # set by ld.so
  .quad _rtld_bind_start  # set by ld.so

  .quad ...  # relocated by R_X86_64_JUMP_SLOT referencing foo
  .quad ...  # relocated by R_X86_64_JUMP_SLOT referencing bar
```

In the relocatable object file, we have two call instructions.
The linker synthesizes stubs in `.plt` and redirects calls into the stubs.

In the assembly dump, `foo@plt` is such a stub.
Note that `foo@plt` is not a symbol table entry. It is just that objdump displays the PLT entry as `foo@plt`.
We use the notation to describe a stub.
The stub consists of 3 instructions.
The first jmpq instruction loads an entry from `.got.plt` and performs an indirect jump.
The remaining two instructions will be described when introducing lazy binding.

`.got.plt` holds an array of word size entries.
On some architectures (x86-32, x86-64) `.got.plt[0]` is the link time address of `_DYNAMIC`.
`.got.plt[1]` and `.got.plt[2]` are reserved by ld.so.
`.got.plt[1]` is a descriptor of the current component while `.got.plt[2]` is the address of the PLT resolver.

The subsequent entries are for resolved function addresses.
Each entry is relocated by an `R_*_JUMP_SLOT` dynamic relocation describing the target function symbol.
Resolving a function address is also called a binding. There are two binding schemes.

Note: some architectures use `R_*_JMP_SLOT` instead of `R_*_JUMP_SLOT` for the dynamic relocation type.

### Eager binding (immediate binding)

The simpler scheme is eager binding.
Before ld.so transfers control to the component, during relocation resolving, ld.so resolves the address of `foo` and writes it into the `.got.plt` entry.
Therefore, when the program runs, the `.got.plt` entry is guaranteed to hold the address of the target function.

This is the model used by musl and Android bionic.
In other ld.so implementations, lazy binding is usually the default.
However, with the rise of security hardening on Linux distributions, many have switched to full RELRO linking scheme.
If the component is linked with `ld -z now`, the linker will set the `DF_1_NOW` dynamic tag, and ld.so will use eager binding.

Eager binding can also be enabled with the environment variable `LD_BIND_NOW=1`.

For better understanding, we can ignore many instructions in the assembly output above.
```asm
<.text>:
  call foo
  call bar

<.plt>:
  ...

foo@plt:
  jmpq *.got.plt+24(%rip)
  ...

bar@plt:
  jmpq *.got.plt+32(%rip)
  ...

<.got.plt>:
  ...
  .quad ...  # relocated by R_X86_64_JUMP_SLOT referencing foo
  .quad ...  # relocated by R_X86_64_JUMP_SLOT referencing bar
```

Eager binding has a bonus: it can detect underlinking problems.
See [Dependency related linker options](/static/2021-06-13-dependency-related-linker-options) for details.

### Lazy binding (deferred binding)

The extra instructions and extra entries in `.got.plt` are all for lazy binding.

The ELF dynamic shared library scheme postpones binding from the link time to the load time.
There is some slowdown due to work before the program starts.
In addition, a shared object has more code than the portion linked into the executable with static linking.
This is because shared objects usually need to export all symbols which can defeat the archive member extraction trick and linker garbage collection.

Anyhow, eagerly resolving all `R_*_JUMP_SLOT` relocations had a significant overhead.

In the lazy binding scheme, ld.so does minimum work to ensure the jmp instruction lands in a trampoline which will tail call the PLT resolver in ld.so.
On x86-64, the associated `.got.plt` entry refers to the pushq instruction immediately after the jmpq.
The pushq instructions pushes a descriptor (for the `.got.plt` entry) and jumps to the PLT header.
The PLT header is a stub calling the function address stored at `.got.plt+16` (PLT resolver in ld.so) with an extra argument (descriptor for the current component).

```asm
<.plt>:
  pushq .got.plt+8(%rip)   # Push the descriptor
  jmpq *.got.plt+16(%rip)  # Jump to the PLT resolver in ld.so
  nop

foo@plt:
  jmpq *.got.plt+24(%rip)  # The first call
  pushq $0                 #   jumps here
  jmp .plt
```

The PLT resolver in ld.so is an assembly trampoline that invokes `elf/dl-runtime.c:_dl_fixup` to resolve the `.got.plt` entry and then perform a tail call to the resolved address.
`_dl_fixup` reads `r_offset` and `r_info` from the JUMP_SLOT entry (`l_info[DT_JMPREL]` plus an offset), looks up the symbol with the `.got.plt` descriptor, then writes the resolved address to the `.got.plt` entry (through `elf_machine_fixup_plt`).
Now that the `.got.plt` entry has been updated to the actual function address, subsequent calls to `foo` will bypass the PLT resolver.

Why is lazy binding slow? I can think of two factors.

First, binding granularity. The shared library scheme we use today is SunOS'. A shared library is a single executable file.
There is no distinction of different modules within a shared library.
The linker and the loader are not able to track which module within a shared library imports which symbols from other modules.
Therefore the linker is forced to import a shared library as a whole and the loader has to resolve all bindings as a while.
(In Multics the shared library is implemented as a collection of object files, like an archive.)

In the absence of `LD_PRELOAD`, ld.so first looks at the symbol table of the executable itself, then at the symbol tables of the `DT_NEEDED` entries (in order), and then at the second level `DT_NEEDED` entries, and so on.
You may think that symbol search can be optimized if all defined symbols are added in one global hash table.
However, a global hash table also costs some memory.
In addition, dlopen, dlmopen, symbol versioning, and probably some features I don't know can all introduce complexity in the process and make a global hash table complex.
Anyway, none of glibc/musl/FreeBSD rtld implements it.

## `-fno-plt`

After the `.got.plt` entry is resolved, a function call requires the following 3 instructions:
```asm
In .text: call foo  # 5 bytes
At foo@plt: jmpq *.got.plt+offset(%rip)
In another component: the first instruction of the target function
```

An optimization idea is that we can just use one instruction (note that a GOT entry is eager binding), essentially inlining the PLT entry:
```asm
In .text: call *.got.plt+offset(%rip)  # 6 bytes
```

GCC 6.0 added `-fno-plt` to AArch64 and x86 to perform this transformation.
The call instruction is one byte longer than the `R_X86_64_PLT32` form.

If the target symbol is preemptible, there is a `R_X86_64_GLOB_DAT` relocation, and the optimization works as intended.

However, if the target is non-preemptible, i.e. the target is resolved to the same component, this is actually a pessimization.
The linker (GNU ld, gold, and ld.lld) can mitigate the loss by rewriting the 6 bytes to `addr32 call foo`. The downside is the one-byte-longer code sequence.

By specifying `-fno-plt`, the user is making a trade-off on a function call which is not known to be bound locally at compile time: whether it resolves to the same component.
For some benchmarks, especially those where libc functions are a bottleneck, there may be a performance win.
This is however deceiving, as statically linking some libc functions will hide the performance win.

On RISC architectures, there is typically no single instruction which can load a GOT entry and perform an indirect call.
`-fno-plt` has a longer code quence. For example, AArch64 needs 3 instructions:
```asm
adrp    x0, _GLOBAL_OFFSET_TABLE_
ldr     x0, [x0, #:gotpage_lo15:bar]
br      x0
```

The trade-off leans more towards the "useless" side.

Instead of using a global `-fno-plt`, the function attribute `__attribute__((noplt))` can be added to individual declarations for fine-grained control.
The compiler generates GOT indirection for calls to annotated functions.
The effect is similar to Windows `__dllspec(dllimport)`.
The attribute is only useful when a function is guaranteed to be in a shared object.
However, for many use cases static libraries may be used and therefore there is no such guarantee.

GCC's x86 port [supports](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=e7c77c4f12100602e078cee454b8d0e7433c2660) `-mforce-indirect-call` to force register indirect branches for function calls.
This mode is designed for Intel Processor Trace (PT), but can also avoid PLT entries like `-fno-plt`.
```asm
movq preemptible@GOTPCREL(%rip), %rax
call *%rax
leaq nonpreemptible(%rip), %rax
call *%rax
```

I added AArch64 `-fno-plt` support to LLVM.

## GOT setup is expensive without PC-relative addressing

On some older architectures, PLT may introduce overhead in function prologue code.
The fundamental problem is that they do not support memory load with PC-relative addressing (e.g. x86-64 PC-relative instructions).
Such architectures typically make a distinction between non-PIC PLT and PIC PLT.

x86-32 is a notorious example.
```c
void bar(void);
void foo(void) { bar(); }
```

```asm
foo:
  pushl   %ebx
  call    __x86.get_pc_thunk.bx
  addl    $_GLOBAL_OFFSET_TABLE_, %ebx
  subl    $8, %esp
  call    bar@PLT
  addl    $8, %esp
  popl    %ebx
  ret

.section        .text.__x86.get_pc_thunk.bx,"axG",@progbits,__x86.get_pc_thunk.bx,comdat
.globl  __x86.get_pc_thunk.bx
.hidden __x86.get_pc_thunk.bx
.type   __x86.get_pc_thunk.bx, @function
__x86.get_pc_thunk.bx:
  movl    (%esp), %ebx
  ret
```

What went wrong?

To work around the lack of PC-relative addressing, the i386 psABI reserves the callee-saved register `ebx` to refer to the GOT base (`_GLOBAL_OFFSET_TABLE_`).
Then the PLT entry can load the `.got.plt` entry (by adding an offset to the GOT base) and do an indirect jump.

Using a callee-saved register has pros and cons.
On the upside, a function just needs to compute `_GLOBAL_OFFSET_TABLE_` once, because callees cannot clobber the register.
On the downside, one general-purpose register is wasted (significant on x86-32 due to the lack of general-purpose registers.
Prologue/epilogue code needs to save and restore the register, so tail calls are inhibited.

[32-bit x86 Position Independent Code - It's that bad](https://ewontfix.com/18/) mentions an alternative scheme which allows a tail call:
```asm
foo:
  call __x86.get_pc_thunk.cx
  jmp  *bar@GOT+_GLOBAL_OFFSET_TABLE_(%ecx)
```

`jmp  *bar@GOT+_GLOBAL_OFFSET_TABLE_(%ecx)` requires a relocation type representing the distance from PC to the GOT entry.
Unfortunately `R_386_GOT32` and `R_386_GOT32X` relocation types are misdesigned and do not support the usage.
That said, adding an extra `addl` instruction is not too bad. GCC since 6.0 can produce the following with `-fno-plt`:

```asm
foo:
  call __x86.get_pc_thunk.ax
  addl $_GLOBAL_OFFSET_TABLE_, %eax
  jmp  *bar@GOT(%eax)
```

This code sequence uses a GOT-generating relocation which may defeat lazy binding, so it is not the default.
Thankfully x86-64 supports PC-relative addressing and does not have the aforementioned problem.

## Life of a `.plt`/`.got.plt` entry

For a function call, the compiler emits a branch instruction.

If the branch target is within the same section and the distance between the call site and the target is a known constant at assembly time, the assembler resolves it;
otherwise the assembler needs to produce a relocation for the linker to process.
The relocation type indicates that the linker may create a PLT entry.
Such a relocation type is sometimes called PLT-generating.

When the linker sees a PLT-generating relocation, it checks whether the referenced symbol is _preemptible_ (_interposable_), i.e., provided by a definition of the same name in another component.
If the symbol is not preemptible, the relocation can be resolved at link time and ld.so will not need to anything.

If the symbol is preemptible, the linker creates a PLT entry and a `.got.plt` entry relocated by an `R_*_JUMP_SLOT` relocation.
For subsequent PLT-generating relocations referencing the same symbol, the linker just reuses the entry.
The address of the PLT entry is insignificant: it can be called, but its address cannot be taken by the program.

Technically the linker can use multiple entries for one symbol.
It just wastes space for the majority of cases.

The second case for PLT generation is a non-preemptible ifunc.
See [GNU indirect functions](/blog/2021-01-18-gnu-indirect-function) for details.

### Canonical PLT entries

This is the third case a linker needs to create a PLT entry.

For position dependent code (`-fno-pic`), traditionally the compiler optimizes for statically linked executables and uses direct addressing (usually absolute relocations).
Taking the address of a function does not use GOT indirection.
If the symbol turns out to be external, the linker has to employ a trick called "canonical PLT entry" (`st_shndx=0, st_value!=0`).
The term is a parlance within a few ld.lld developers, but not broadly adopted.

Every PLT entry has an associated `R_*_JUMP_SLOT` entry and has a dynamic symbol (`st_shndx=0`, display as `UND` in `readelf` output).
If we assign a non-zero value to the symbol, we can consider the address of the PLT entry as canonical and bind references from executable/shared objects to it.

```c
// a.c
extern void foo(void);
int main() { foo(); }

// b.c
void foo(void) {}
```

```text
% gcc -m32 -shared -fuse-ld=bfd -fpic b.c -o b.so
% gcc -m32 -fno-pic -no-pie -fuse-ld=lld a.c ./b.so
% readelf -W --dyn-syms a.out

Symbol table '.dynsym' contains 7 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
...
     6: 00401670     0 FUNC    GLOBAL DEFAULT  UND foo
```

If `b.so` binds `foo` locally, e.g. if it is linked with `-Bsymbolic-functions`, taking the address from the executable and from the shared object will get different results.

To fix this (C and C++ require address uniqueness), compilers need to use GOT indirection.
See [Copy relocations, canonical PLT entries and protected visibility](2021-01-09-copy-relocations-canonical-plt-entries-and-protected), sadly it doesn't seem that GCC folks want to take action.

x86-32 has an extra problem that it uses `R_386_PC32` to represent a function call from non-PIC code.
See my article for details.

## Stack unwinding

As linker synthesized code, the PLT entries may not have unwind tables information.
Unwinders using `.eh_frame` needs to recognize the `_PROCEDURE_LINKAGE_TABLE_` region.
See [Explain GNU style linker options](/blog/2020-11-15-explain-gnu-linker-options) `--no-ld-generated-unwind-info` (the positive option is unnecessary in my opinion) for details.

## Mach-O stub

Mach-O's stub scheme is similar to Procedure Linkage Table. We can make the conceptual mapping:

* `__TEXT,__stubs`: `.plt` entries
* `__TEXT,__stubs_helper`: `.plt` header and PowerPC's glink entries
* `__DATA,__la_symbol_ptr`: `.got.plt`

For an external function call, the text section calls a stub in `__TEXT,__stubs`.
The stub loads a `__la_symbol_ptr` entry which initially points in the middle of `__TEXT,__stubs_helper`.
The code pushes on the stack the offset into `__la_symbol_ptr`, then branches to the start of `__TEXT,__stubs_helper`.
The start of `__TEXT,__stubs_helper` pushes the address of `_dyld_private` and jumps to ``libdyld.dylib`dyld_stub_binder``.

## Case study

### AArch64

AArch64 has the best PLT design in my opinion.

`R_AARCH64_CALL26`, `R_AARCH64_JUMP26`, and `R_AARCH64_PLT32` are PLT-generating relocation types.

PLT header

```asm
stp    x16, x30, [sp,#-16]!
adrp   x16, Page(&(.got.plt[2]))
ldr    x17, [x16, Offset(&(.got.plt[2]))]
add    x16, x16, Offset(&(.got.plt[2]))
br     x17
nop
nop
nop
```

.plt[n]

```asm
adrp x16, Page(&(.got.plt[n]))
ldr  x17, [x16, Offset(&(.got.plt[n]))]
add  x16, x16, Offset(&(.got.plt[n]))
br   x17
```

`x16` holds the address of the `.got.plt` entry and `x17` holds the target address.
The PLT resolver (e.g. `_rtld_bind_start` in FreeBSD rtld) can compute the `R_AARCH64_JUMP_SLOT` index from `x16` and `x17`.

When Arm v8.5 Branch Target Enablement is enabled, all indirect branches must land on a `bti` instruction.
A PLT entry may indirectly branch to the PLT header, so the PLT header needs a `bti` instruction.
```txt
% aarch64-linux-gnu-gcc -fpic -mbranch-protection=standard -shared a.c -o a.so
```

In glibc, the PLT resolver does not preserve all the registers required for AdvSIMD and SVE vector calls.
Saving and restoring all registers would use more stack space and break existing binaries.
To support such a variant procedure call standard with different register usage convention, the following scheme is used.

Function symbols with the variant procedure call standard have the `STO_AARCH64_VARIANT_PCS` bit.
The linker adds the `DT_AARCH64_VARIANT_PCS` dynamic tag if there is at least one PLT referencing an `STO_AARCH64_VARIANT_PCS` symbol.
In glibc ld.so, when `DT_AARCH64_VARIANT_PCS` is set, the relocation resolver scans all `R_*_JUMP_SLOT` relocations and eagerly resolves `STO_AARCH64_VARIANT_PCS` entries.

For Mach-O, the stub helper and a stub look like:
```asm
__stub_helper:
  adrp  x17, _dyld_private@page
  add   x17, x17, _dyld_private@pageoff
  stp   x16/x17, [sp, #-16]!
  adrp  x16, dyld_stub_binder@page
  ldr   x16, [x16, dyld_stub_binder@pageoff]
  br    x16

  ldr   w16, 1f
  b     __stub_helper
  1: .long lazy_bind_offset_0

  ldr   w16, 1f
  b     __stub_helper
  1: .long lazy_bind_offset_1


one stub:
  adrp  x16, (__la_symbol_ptr+offset)@page
  ldr   x16, [x16, (__la_symbol_ptr+offset)@pageoff]
  br x16
```

The content of `__stub_helper` is similar to PowerPC64 `__glink_PLTresolve` and the subsequent `b` instructions.
The advantage compared with ELF is one fewer instruction in the call stub.

### MIPS

I know really little about MIPS and cannot add much description.

The normal code generation is like `-fno-plt`.
MIPS did not use PLT at all until ~2008-2009.

`-mno-abicalls`, which is only available for `-fno-pic`, can generate `j` and `jal` instructions.
However, GNU ld has a warning `linking abicalls files with non-abicalls files`.

### PA-RISC

In _The Hewlett-Packard Journal, June 1992_, _Shared libraries for HP-UX_ says:

> Dedicated linkage Table Pointer. Since code and data could not be mapped contiguously, the linkage tables could not be accessed with a pc-relative code sequence generated at compile time. Therefore, we chose a general register (gr19) as a place for holding the pointer for shared library linkage.

gr19 appears to be a callee-saved register.
This gives a clue why older architectures do not use PC-relative addressing.
Today for non-FDPIC ABI, the distance from code to data should be a link-time constant for performance reasons.
(Well, secruity researchers may want to proceed this road for more randomness.)

Cary Coutant kindly shared with me the answer. There were two reasons.

First, PA-RISC could not maintain a link-time constant delta between code and data segments.
PA-RISC (and, later, Itanium) used a global virtual addressing model, with virtually-addressed caches, and segment (or region) registers that extended the virtual addresses from a per-process address space into a global address space.
Code, which can be shared across processes, must be placed in one quadrant of the address space (octant for Itanium), private data in another, and shared data in yet another.

Second, PC-relative instructions would be expensive.
Because of the deep instruction pipeline and out-of-order execution, routing the PC over to an adder for the purpose of computing an effective address would have forced PA-RISC to increase the basic cycle time of the CPU.

The reason appears to be shared with many processors at that time.
With improvement of chip design, PC-relative instructions are no longer expensive.

### PowerPC32

_Power Architecture® 32-bit Application Binary Interface Supplement 1.0 - Linux® & Embedded_ specifies two PLT ABIs: BSS-PLT and Secure-PLT.

BSS-PLT is the older one.
While `.plt` on other architectures are created by the linker, BSS-PLT let ld.so generate the PLT entries.
This has the advantage that the section can be made `SHT_NOBITS` and therefore not occupy file size.
The downside is the security concern of writable and executable memory pages.
Even worse, as an implementation issue, GNU ld places `.plt` in the text segment and therefore the whole text segment is writable and executable.
`-z relro -z now` has no effect.

In the newer Secure-PLT ABI, `.plt` holds the table of function addresses.
This usage is weird, because on other architectures such a section is called `.got.plt`.

Like x86-32, PowerPC32 lacks memory load with relative addressing and has a distinction between non-PIC PLT and PIC PLT.

```asm
00000000.plt_call32.f:
  lis 11, .plt[n]@ha
  lwz 11, .plt[n]@l(11)
  mtctr 11
  bctr

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

The call-clobbered register r11 is used to compute the `.plt` entry address.
For a non-PIC PLT, just use absolute addressing.

For a PIC PLT, the callee-saved register r30 holds the GOT base.
PowerPC32 is one of the few old architectures where GCC `-fpic` and `-fPIC` are different.
For `-fpic`, the GOT base is at `_GLOBAL_OFFSET_TABLE_` in the component.
For `-fPIC`, the GOT base is at `.got2` for the current translation unit. The component may have multiple translation units and each has a different `.got2`.

Unlike x86 (which just places the lazy binding code after the initial `jmpq` instruction), the Secure-PLT ABI places the lazy binding code in `__glink_PLTresolve` (like PLT header on x86) and following entries.
When I added PowerPC32 port to ld.lld, I picked the glink code in a separate section `.glink` for clarity.

For PIC function prologue code, setting up r30 is expensive. The previous foo tail calling bar C code requires these many instructions.

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

### PowerPC64 ELFv1

The latest ABI is _64-bit PowerPC ELF Application Binary Interface Supplement 1.9_.
It defines the TOC ("Table of Contents") section which combines the functions of the GOT and the small data section.
To optimize for inter-component calls, the ABI tries to avoid the per-function TOC base setup code.
The TOC base address is stored in r2 which is not clobbered by inter-component calls.
We will see later that as a result intra-component calls are slower.

The single `.got.plt` entry on other architectures is replaced by a function descriptor which contains 3 entries:

* The first doubleword contains the address of the entry point of the function.
* The second doubleword contains the TOC base address for the function.
* The third doubleword contains the environment pointer for languages such as Pascal and PL/1.

```asm
00000000000002c0 <0000001a.plt_call.bar>:
  std     r2,40(r1)       # Save the TOC base address
  ld      r12,-32488(r2)  # Load the function address from the function descriptor
  mtctr   r12
  ld      r2,-32480(r2)   # Load the TOC base address from the function descriptor
  cmpldi  r2,0            # Non-zero indicates that the function descriptor has been resolved
  bnectr+                 # Just branch there
  b       344 <bar@plt>   # otherwise, call the glink entry to resolve the function descriptor
  .long 0x0

00000000000002e0 <.foo>:
  mflr    r0
  std     r0,16(r1)
  stdu    r1,-112(r1)
  bl      2c0 <0000001a.plt_call.bar>  # Branch to the call stub
  ld      r2,40(r1)                    # Restore the TOC base address
  addi    r1,r1,112
  ld      r0,16(r1)
  mtlr    r0
  blr
  .long 0x0
  .long 0x1000000
  .long 0x80
  .long 0x1fce0
  .long 0x0

0000000000000318 <__glink_PLTresolve>:
  mflr    r12
  bcl     20,4*cr7+so,320 <__glink_PLTresolve+0x8>
  mflr    r11
  ld      r2,-16(r11)
  mtlr    r12
  add     r11,r2,r11
  ld      r12,0(r11)
  ld      r2,8(r11)
  mtctr   r12
  ld      r11,16(r11)
  bctr

0000000000000344 <bar@plt>:
  li      r0,0            # Pass an argument to describe the entry index
  b       318 <__glink_PLTresolve>
```

`bl 0x2c0` jumps to the call stub.
The call stub loads the function address and the TOC base address from the function descriptor.
If the TOC base address is zero, which means the function descriptor hasn't been resolved, branch to the glink entry to resolve the function descriptor;
otherwise branch to the resolved function address.
In both cases, r2 has been updated to the new TOC base address.

After returning from the callee, the linker synthesized `ld r2,40(r1)` is executed to restore the TOC base address.

When the target function is bound locally, the instruction after `bl` is a `nop` (instead of `ld r2,40(r1)`).
This scheme is better than the compiler unconditionally generating a code sequence to load the TOC base address.

When the target function is external, the scheme pays the cost in the call stub.
This scheme is worse than the compiler unconditionally generating a code sequence to load the TOC base address.

### PowerPC64 ELFv2

`R_PPC64_REL14`, `R_PPC64_REL24`, and `R_PPC64_REL24_NOTOC` are PLT-generating relocation types.

Here are the main pain points with ELFv1:

* Function descriptors waste space.
* Call stubs are too long (7 instructions padded by one nop).

The idea is to move the TOC base address setup code from call stubs to the start of functions.

```asm
00000000000002c0 <0000001a.plt_call.bar>:
  std     r2,24(r1)                    # Save the TOC base address
  ld      r12,-32496(r2)               # Load function address from .plt entry
  mtctr   r12
  bctr

00000000000002e0 <foo>:
# globalentry: functions with a different TOC jump here.
  addis   r2,r12,2                     # Compute the TOC base address
  addi    r2,r2,31776
# localentry: functions with the same TOC jump here.
  mflr    r0
  std     r0,16(r1)
  stdu    r1,-32(r1)
  bl      2c0 <0000001a.plt_call.bar>  # Branch to the call stub
  ld      r2,24(r1)                    # Restore the TOC base address
  addi    r1,r1,32
  ld      r0,16(r1)
  mtlr    r0
  blr

0000000000000348 <__glink_PLTresolve>:
  mflr    r0
  bcl     20,4*cr7+so,350 <__glink_PLTresolve+0x8>
  mflr    r11
  mtlr    r0
  ld      r0,-16(r11)
  subf    r12,r11,r12
  add     r11,r0,r11
  addi    r0,r12,-44
  ld      r12,0(r11)
  rldicl  r0,r0,62,2
  mtctr   r12
  ld      r11,8(r11)
  bctr

000000000000037c <bar@plt>:
  b       348 <__glink_PLTresolve>
```

The distance between the global entry and the local entry is normally 8 bytes (2 instructions), but there can be more in the large code model.
I have seen BoringSSL using more instructions to [instrument assembly files](https://github.com/google/boringssl/blob/master/crypto/fipsmodule/FIPS.md) to avoid relocations from text to data.

The globalentry/localentry scheme requires caution in many parts of the toolchain: taking addresses, [copying symbols](https://reviews.llvm.org/D56586), [computing branch distance for range extension thunks](https://reviews.llvm.org/D61058), etc.

In POWER10, IBM finally decided to add PC-relative instructions (Prefixed Load).

```asm
0000000000000300 <0000001a.plt_call.bar>:
  pld     r12,130320      # 20010 [bar@plt]
  mtctr   r12
  bctr

0000000000000340 <foo>:
  mflr    r0
  std     r0,16(r1)
  stdu    r1,-32(r1)
  bl      300 <0000001a.plt_call.bar>
  addi    r1,r1,32
  ld      r0,16(r1)
  mtlr    r0
  blr

# __glink_PLTresolve and bar@plt are the same.
```

So why do we still need glink entries?
Well, the `pld` instruction loads the `.plt` entry (`.got.plt` on other architectures), but the address of the `.plt` entry is not communicated to the PLT resolver.
This means that for the small code model the `pld` scheme is not optimal.

Compatibility is a thing.
How to make the PC-relative instructions work with TOC and globalentry/localentry need a lot of thought and some complexity in the linker.
I am sure that if you read the GNU ld or gold code, you will be amused:)

```c
if (plt) {
  if (is(R_PPC64_REL24_NOTOC))
    use PC-relative PLT call stub (`__plt_pcrel_${name}`)
  else
    use TOC PLT call stub
} else if ((is(R_PPC64_REL14) || is(R_PPC64_REL24)) && (st_other>>5) == 1) {
  // The caller requires valid r2 but the callee may not preserve r2.
  use r2 save stub
} else if (is(R_PPC64_REL24_NOTOC)) {
  if ((st_other>>5) > 1)
    use r12 setup stub (`__gep_setup_${name}`)
  else
    use PC-relative long branch thunk (`__long_branch_pcrel_${name}`)
} else {
  if (shared || pie)
    use TOC position-independent long branch thunk (`__long_branch_${name}`)
  else
    use TOC position-dependent long branch thunk (`__long_branch_${name}`)
}
```

### RISC-V

`R_RISCV_CALL` and `R_RISCV_CALL_PLT` are the only two PLT-generating relocation types.
The GNU toolchain use them in non-PIC and PIC code, respectively.

However, RISC-V supports memory load with PC-relative addressing, so it doesn't need non-PIC PLT vs PIC PLT distinction.
I filed [`R_RISCV_CALL vs R_RISCV_CALL_PLT`](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/98) about deprecation for the former.

Otherwise, its PLT scheme is quite good.
```asm
Disassembly of section .plt:

0000000000000270 <.plt>:
 270:   00002397                auipc   t2,0x2
 274:   41c30333                sub     t1,t1,t3
 278:   d903be03                ld      t3,-624(t2) # 2000 <.got>
 27c:   fd430313                addi    t1,t1,-44
 280:   d9038293                addi    t0,t2,-624
 284:   00135313                srli    t1,t1,0x1
 288:   0082b283                ld      t0,8(t0)
 28c:   000e0067                jr      t3

0000000000000290 <bar@plt>:
 290:   00002e17                auipc   t3,0x2
 294:   d80e3e03                ld      t3,-640(t3) # 2010 <bar>
 298:   000e0367                jalr    t1,t3
 29c:   00000013                nop
```

The address of an anchor in the PLT entry is available in t1, so the `R_*_JUMP_SLOT` index can be computed in the PLT header.

### x86

For x86-32, `R_386_PLT32` is the only PLT-generating relocation type.
Unfortunately non-PIC code uses `R_386_PC32` which can trigger caninical PLT entries.

For x86-64, `R_X86_64_PLT32` is the only PLT-generating relocation type.

ld.lld supports `-z retpolineplt` for Spectre v2 mitigation.
The indirection branch needs to be protected by a special code sequence.

```asm
000000000000142c <foo>:
    142c: 55                            pushq   %rbp
    142d: 48 89 e5                      movq    %rsp, %rbp
    1430: e8 5b 00 00 00                callq   0x1490 <bar+0x1490>
    1435: 90                            nop
    1436: 5d                            popq    %rbp
    1437: c3                            retq

0000000000001460 <.plt>:
    1460: ff 35 4a 21 00 00             pushq   8522(%rip)              # 0x35b0 <_GLOBAL_OFFSET_TABLE_+0x8>
    1466: 4c 8b 1d 4b 21 00 00          movq    8523(%rip), %r11        # 0x35b8 <_GLOBAL_OFFSET_TABLE_+0x10>
    146d: e8 0e 00 00 00                callq   0x1480 <.plt+0x20>
    1472: f3 90                         pause
    1474: 0f ae e8                      lfence
    1477: eb f9                         jmp     0x1472 <.plt+0x12>
    1479: cc                            int3
    147a: cc                            int3
    147b: cc                            int3
    147c: cc                            int3
    147d: cc                            int3
    147e: cc                            int3
    147f: cc                            int3
    1480: 4c 89 1c 24                   movq    %r11, (%rsp)
    1484: c3                            retq
    1485: cc                            int3
    1486: cc                            int3
    1487: cc                            int3
    1488: cc                            int3
    1489: cc                            int3
    148a: cc                            int3
    148b: cc                            int3
    148c: cc                            int3
    148d: cc                            int3
    148e: cc                            int3
    148f: cc                            int3
    1490: 4c 8b 1d 29 21 00 00          movq    8489(%rip), %r11        # 0x35c0 <_GLOBAL_OFFSET_TABLE_+0x18>
    1497: e8 e4 ff ff ff                callq   0x1480 <.plt+0x20>
    149c: e9 d1 ff ff ff                jmp     0x1472 <.plt+0x12>
    14a1: 68 00 00 00 00                pushq   $0
    14a6: e9 b5 ff ff ff                jmp     0x1460 <.plt>
    14ab: cc                            int3
    14ac: cc                            int3
    14ad: cc                            int3
    14ae: cc                            int3
    14af: cc                            int3
```

With `-z now`, the code sequence can be simplified a bit.

As part of Intel's Control-flow Enforcement Technology, Indirect Branch Tracking requires that indirect jumps land on an `endbr` instruction.

```txt
% gcc -fpic -fcf-protection=full -shared a.c -o a.so
```

An `endbr64` instruction takes 4 bytes and does not fit in the current 16-byte PLT entry scheme.
They somehow decided to add a new section `.plt.sec`.
The linker uses `endbr64` if all object files have the `GNU_PROPERTY_X86_FEATURE_1_IBT` GNU property.

```asm
Disassembly of section .plt:

0000000000001000 <.plt>:
    1000:       ff 35 02 30 00 00       push   0x3002(%rip)          # 4008 <_GLOBAL_OFFSET_TABLE_+0x8>
    1006:       f2 ff 25 03 30 00 00    bnd jmp *0x3003(%rip)        # 4010 <_GLOBAL_OFFSET_TABLE_+0x10>
    100d:       0f 1f 00                nopl   (%rax)
    1010:       f3 0f 1e fa             endbr64                      # Jumps here if not resolved yet
    1014:       68 00 00 00 00          push   $0x0
    1019:       f2 e9 e1 ff ff ff       bnd jmp 1000 <bar@plt-0x20>
    101f:       90                      nop

Disassembly of section .plt.sec:

0000000000001020 <bar@plt>:
    1020:       f3 0f 1e fa             endbr64
    1024:       f2 ff 25 ed 2f 00 00    bnd jmp *0x2fed(%rip)        # 4018 <bar+0x2fd8>
    102b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

Disassembly of section .text:

0000000000001030 <foo>:
    1030:       f3 0f 1e fa             endbr64
    1034:       e9 e7 ff ff ff          jmp    1020 <bar@plt>
    1039:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
```

`foo` will jump to `bar@plt` in `.plt.sec`, then either jump to the corresponding `.plt` entry or the target function directly.

Let me register a complaint here.
I think `.plt.sec` is unneeded complexity and I [proposed](https://groups.google.com/g/x86-64-abi/c/Z6GqQDy_NaI/m/_ok9VVyPAgAJ):

```asm
endbr64
jmpq *xxx(%rip)  # jump to the next endbr64 for lazy binding
endbr64
pushq relocaton_index
jmpq *xxx(%rip)  # jump to .plt
```

<https://groups.google.com/g/x86-64-abi/c/sQcX3__r4c0> has more discussions.
Most folks agreed that the 2-PLT scheme was over-engineered.
After this event, I subscribed to x86-64-abi in case I missed such over-engineering designs in the future.

If we can clobber r11, we can use the following [scheme](https://groups.google.com/g/x86-64-abi/c/oi3i85b1uI0/m/MAOvvMMSBAAJ):
```asm
plt_header:
f3 0f 1e fa        endbr64
41 53              push %r11
ff 35 00 00 00 00  push GOT[1]
ff 25 00 00 00 00  jmp *GOT[2]
0f 1f 40 00        nop
0f 1f 40 00        nop
0f 1f 40 00        nop
66 90              nop

plt[n]:
f3 0f 1e fa        endbr64
41 bb 00 00 00 00  mov $index, %r11d
ff 25 00 00 00 00  jmp *.got.plt[index]
```

We can optimize out the unconditional `mov` instruction by computing the PLT index with the return address:
```asm
plt_header:
  pop %r11
  sub %r11d, plt_header
  shr $0x5, %r11
  push %r11
  push GOT[1]
  jmp *GOT[2]

plt[n]:
  endbr64
  jmp GOT[foo]
  call plt_header
```

#### x86 PLT rewriting

In 2023-09, GNU ld's x86-64 port [introduced `-z mark-plt`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=832ca732b8a96ff9a3e7c4abf24098bf2a59a96d) to communicate information to rtld to rewrite eligible indirect jump instructions to direct jump instructions.

* The linker adds dynamic tags `DT_X86_64_PLT/DT_X86_64_PLTSZ/DT_X86_64_PLTENT`.
* The addend in `R_X86_64_JUMP_SLOT` relocations are adjusted to indicate the offset of the indirect jump instruction.

Since 2024-01, if binutils is configured with `--enable-mark-plt`, `-z mark-plt` will be the default.

In glibc, when `GLIBC_TUNABLES=glibc.cpu.plt_rewrite=1` or 2 is specified, lazy PLT binding is disabled, and an object file enables `DT_X86_64_PLT/DT_X86_64_PLTSZ/DT_X86_64_PLTENT` tags, glibc [rewrites eligible PLT entries](https://sourceware.org/git/?p=glibc.git;a=commit;h=848746e88ec2aa22e8dea25f2110e2b2c59c712e).
After relocating an object file, `x86_64_dynamic_after_reloc` (due to the `ELF_DYNAMIC_AFTER_RELOC` hook) calls `x86_64_rewrite_plt_in_place`, which changes the permission of the `.plt` memory page (`MAP_PRIVATE`) to `PROT_READ|PROT_WRITE`, rewrites eligible PLT entries, then changes the page to `PROT_READ|PROT_EXEC`.

For each `R_X86_64_JUMP_SLOT` relocation, `x86_64_rewrite_plt_in_place` reads the target address from the `.got.plt` entry, computes the indirect jump address using the addend, then checks whether the jump target is reachable with a direct JMP instruction.
If so, the indirect jump instruction is rewritten to `jmp $target`; otherwise, when `GLIBC_TUNABLES=glibc.cpu.plt_rewrite=2` is specified on APX processors, the indirect jump instruction is rewritten to `jmpabs $target` (64-bit absolute jump).

The mprotect operations increase private data uses and are incompatible with [memory-deny-write-execute](https://git.kernel.org/linus/b507808ebce23561d4ff8c2aa1fb949fe402bc61).
One primary cost of PLT is the use of an extra instruction cache line. Rewriting jump instruction does not eliminate this overhead.

Since the main executable is far away from shared objects in the address space, in the absence of APX `jmpabs`, the PLT rewriting will very likely not occur.

PowerPC32 has a BSS-PLT ABI that generates PLT entries on the fly, which shares some similarity. BSS-PLT is obsoleted primarily due to security concerns.
glibc also implements runtime PLT generation for alpha and sparc32/sparc64.

As a minor note, this optimization will [nullify](https://groups.google.com/g/x86-64-abi/c/vbuHVMK_RIA/m/zi0qi_0pBQAJ) the [`.plt.got` optimization (little benefit, but clever)](/blog/2021-08-29-all-about-global-offset-table#combining-.got-and-.got.plt).

`-fno-plt` may also do better than this PLT optimization.

## Summary

```c
if (supports memory load with PC-relative addressing) {
  You made the right design choice!
  Non-PIC PLT and PIC PLT can be the same.
  See AArch64, RISC-V, x86-64.
  if (letting one register hold the .got.plt address is efficient)
    See AArch64 PLT.
  else
    See PowerPC64 call stub and Mach-O AArch64 stub.
} else if (-fno-plt) {
  Use a call-clobbered register to store _GLOBAL_OFFSET_TABLE_.
  Call sites are longer.
  Every call site needs re-computing _GLOBAL_OFFSET_TABLE_.
  The computation is unneeded for inter-component calls.
  No lazy binding.
  if (optimize inter-component calls)
    Add linker optimization/relaxation.
} else if (optimize inter-component calls) {
  Use the PowerPC64 style TOC.
  if (Don't mind having two entry points at function start)
    See ELFv2 global/local entries.
  else
    See ELFv1 function descriptors.
} else {
  Use the x86-32 scheme.
}
```

# Appendix

## The history of *NIX shared library support

... according to my archaeology.

The a.out format was designed for PDP-11.
The 16-bit design could be naturally extended to 32-bit, which was suitable for many 32-bit machines and operating systems.

Outside the UNIX world, VAX/VMS had dynamic linking earlier than Unix.

In _1984 USENIX UniForum Conference Proceedings_, _Transparent Implementation of Shared Libraries_ described a library stub and link table scheme which is similar to `.plt` plus `.got.plt` used today.

System V release 3 switched to the COFF format.
In _1986 Summer USENIX Technical Conference & Exhibition Proceedings_, _Shared Libraries on UNIX System V_ from AT&T described a shared library design.
Its shared library must have a fixed virtual address, which is called "static shared library" in _Linkers and Loaders_'s term.

In 1988, SunOS 4.0 was released with an extended a.out binary format with dynamic shared library support.
Unlike previous static shared library schemes, the a.out shared libraries are position independent and can be loaded at different addresses.
The GOT and PLT schemes are what we see today.
In 1992, SunOS 5.0 (Solaris 2.0) switched to ELF.

The first System V release 4 ABI (using ELF) is from around that time.
The work was a cooperative effort between AT&T and Sun.
AT&T contributed the ELF object format.
Sun contributed all of the dynamic linking implementation from SunOS 4.x.

In January 1991, HP-UX 8.0 (developed since around 1989) added dynamic shared libraries to Spectrum Object Module.
In 1996, they switched to ELF for 64-bit runtime on PA-RISC 2.0.
SOM was kept for 32-bit runtime.
HP-UX on Itanium used ELF exclusively.

IRIX 5.0, released in 1993, included ELF support.

In 1993, on NetBSD,
<https://github.com/NetBSD/src/commit/97ca10e37476fb84a20a8ec4b0be3188db703670> (`A linker supporting shared libraries.`)
and
<https://github.com/NetBSD/src/commit/3d68d0acaed0a32f929b2c174146c62940005a18> (`A linker supporting shared libraries (run-time part).`)
added shared library support similar to the SunOS scheme.
In 2000, NetBSD 1.5 switched to ELF by default.

In 1994, GNU ld added ELF shared library support.

The Linux was using a static shared library variant of a.out.
In 1995, glibc added ELF shared library support.

FreeBSD ported shared library code from NetBSD since 1994.
In 1998, FreeBSD 3.0 switched to ELF by default.

Dynamic shared library support on Mach-O was later.
In a NeXTSTEP manual released in 1995, I can find `MH_FVMLIB` (fixed virtual memory library, which appears to be a static shared library scheme) but not `MH_DYLIB` (used by modern macOS for .dylib files).
