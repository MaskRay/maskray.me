---
layout: post
title: All about thread-local storage
author: MaskRay
tags: [binutils,elf,tls]
---

Updated in 2025-02.

Thread-local storage (TLS) provides a mechanism allocating distinct objects for different threads.
It is the usual implementation for GCC extension `__thread`, C11 `_Thread_local`, and C++11 `thread_local`, which allow the use of the declared name to refer to the entity associated with the current thread.
This article will describe thread-local storage on ELF platforms in detail, and touch on other related topics, such as: thread-specific data keys and Windows/macOS TLS.

<!-- more -->

An example usage of thread-local storage is POSIX `errno`:

> Each thread has its own thread ID, scheduling priority and policy, errno value, floating point environment, thread-specific key/value bindings, and the required system resources to support a flow of control.

Different threads have different `errno` copies. `errno` is typically defined as a function which returns a thread-local variable (e.g. `__errno_location`).

For each architecture, the authoritative ELF ABI document is the processor supplement (psABI) to the System V ABI (generic ABI).
These documents usually reference _The ELF Handling for Thread-Local Storage_ by Ulrich Drepper.
The document, however, mixes general specifications and glibc internals.

## Representation

### Assembler behavior

The compiler usually defines thread-local variables in `.tdata` and `.tbss` sections (which have the section flag `SHF_TLS`).
The symbols representing thread-local variables have type `STT_TLS` (representing thread-local storage entities).
In GNU as syntax, you can give `a` the type `STT_TLS` with `.type a, @tls_object`.
The `st_value` value of a TLS symbols is the offset relative to the defining section.

```asm
.section .tbss,"awT",@nobits
.globl a, b
.type a, @tls_object
.type b, @tls_object
a:
  .zero 4
  .size a, .-a
b:
  .zero 4
  .size b, .-b
```

In this example, `st_value(a)=0` while `st_value(b)=4`.

In Clang and GCC produced assembly, thread-local variables are annotated as `.type a, @object` (`STT_OBJECT`).
When the assembler sees that such symbols are defined in `SHF_TLS` sections or referenced by TLS relocations, `STT_NOTYPE`/`STT_OBJECT` will be upgraded to `STT_TLS`.

GNU as supports an directive `.tls_common` which defines `STT_TLS SHN_COMMON` symbols.
This is an obscure feature. It is not clear whether GCC still has a code path which emits `.tls_common` directives.
LLVM integrated assembler does not support `.tls_common`.

### Linker behavior

The linker combines `.tdata` input sections into a `.tdata` output section.
`.tbss` input sections are combined into a `.tbss` output section.
The two `SHF_TLS` output sections are placed into a `PT_TLS` program header.

* `p_offset`: the file offset of the TLS initialization image
* `p_vaddr`: the virtual address of the TLS initialization image
* `p_filesz`: the size of the TLS initialization image
* `p_memsz`: the total size of the thread-local storage. The last `p_memsz-p_filesz` bytes will be zeroed by the dynamic loader.
* `p_align`: alignment

The `PT_TLS` program header is contained in a `PT_LOAD` program header.
If `PT_GNU_RELRO` is used, `PT_TLS` is contained in a `PT_GNU_RELRO` and the `PT_GNU_RELRO` is contained in a `PT_LOAD`.
Conceptually `PT_TLS` and `STT_TLS` symbols are like in a separate address space.
The dynamic loader should copy the `[p_vaddr,p_vaddr+p_filesz)` of the TLS initialization image to the corresponding static TLS block.

In executable and shared object files, `st_value` normally holds a virtual address. For a `STT_TLS` symbol, `st_value` holds an offset relative to the virtual address of the `PT_TLS` program header.
The first byte of `PT_TLS` is referenced by the TLS symbol with `st_value==0`.

GNU ld treats `STT_TLS SHN_COMMON` symbols as defined in `.tcommon` sections.
Its internal linker script places such sections into the output section `.tdata`.
ld.lld does not support `STT_TLS SHN_COMMON` symbols.

### Dynamic loader behavior

The dynamic loader collects `PT_TLS` program headers from the main executable and immediately loaded shared objects (via transitive `DT_NEEDED`), and allocates static TLS blocks, one block for each `PT_TLS`.
For each `PT_TLS`, the dynamic loader copies `p_filesz` bytes from the TLS initialization image to the TLS block and sets the trailing `p_memsz-p_filesz` bytes to zeroes.

For the static TLS block of the main executable, the module ID is one and the TP offset of a TLS symbol is a link-time constant.
The linker and the dynamic loader share the same formula.

For a shared object loaded at program start, the offset from the thread pointer to its static TLS block is a fixed value at program start, albeit not a link-time constant.
The offset can be referenced by a GOT dynamic relocation used by the initial-exec TLS model.

_The ELF Handling for Thread-Local Storage_ describes two TLS variants and specifies their data structures.
However, only the TP offset of the static TLS block of the main executable is a hard requirement.
Nevertheless, libc implementations usually place static TLS blocks together, and allocate a space for both the thread control block and the static TLS blocks.

For a new thread created by `pthread_create`, the static TLS blocks are usually allocated as part of the thread stack.
Without a guard page between the largest address of the stack and the thread control block, this could be considered as vulnerable as stack overflow can overwrite the thread control block.

## Models

### Local exec TLS model (executable & non-preemptible)

This is the most efficient TLS model. It applies when the TLS symbol is defined in the executable.

The compiler picks this model in `-fno-pic/-fpie` modes if the variable is

* a definition
* or a declaration with a non-default visibility.

The first condition is obvious. The second condition is because a non-default visibility means the variable must be defined by another translation unit in the executable.

```c
_Thread_local int def;
__attribute__((visibility("hidden"))) extern thread_local int ref;
int foo() { return def + ref; }
```

```asm
# x86-64
# The segment register %fs holds the thread pointer. The instruction loads an offset relative to the thread pointer.
movl %fs:def@TPOFF, %eax
```

For the static TLS block of the main executable, the TP offset of a TLS symbol is a link-time constant.
Here is a list of common relocation types:

* arm: `R_ARM_TLS_LE32`
* aarch64:
  + `-mtls-size=12`: `R_AARCH64_TLSLE_ADD_TPREL_LO12`
  + [-65536,65536) (unavailable in GCC/Clang): `R_AARCH64_TLSLE_MOVW_TPREL_G0`
  + `-mtls-size=24` (default): `R_AARCH64_TLSLE_ADD_TPREL_HI12`, `R_AARCH64_TLSLE_ADD_TPREL_LO12_NC`
  + `-mtls-size=32`: `R_AARCH64_TLSLE_MOVW_TPREL_G1`, `R_AARCH64_TLSLE_MOVW_TPREL_G0_NC`
  + `-mtls-size=48`: `R_AARCH64_TLSLE_MOVW_TPREL_G2`, `R_AARCH64_TLSLE_MOVW_TPREL_G1_NC`, `R_AARCH64_TLSLE_MOVW_TPREL_G0_NC`
* i386: `R_386_TLS_LE`
* x86-64: `R_X86_64_TPOFF32`
* mips: `R_MIPS_TPREL_HI16`, `R_MIPS_TPREL_LO16`
* ppc32: `R_PPC_TPREL_HA`, `R_PPC_TPREL_LO`
* ppc64: `R_PPC64_TPREL_HA`, `R_PPC64_TPREL_LO`
* riscv: `R_RISCV_TPREL_HI20`, `R_RISCV_TPREL_LO12_I`, `R_RISCV_TPREL_LO12_S`

For RISC architectures, because an instruction typically has 4 bytes and cannot encode a 32-bit offset, it usually takes two instructions to materialize a TP offset.

In <https://reviews.llvm.org/D93331>, I patched ld.lld to reject local-exec TLS relocations in `-shared` mode.
In GNU ld, at least arm, riscv and x86's ports have the similar diagnostics, but aarch64 and ppc64 do not error.

### Initial exec TLS model (executable & preemptible)

This model is less efficient than local exec. It applies when the TLS symbol is defined in the executable or a shared object available at program start.
The shared object can be due to `DT_NEEDED` or `LD_PRELOAD`.

The compiler picks this model in `-fno-pic/-fpie` modes if the variable is a declaration with default visibility.
The idea is that a symbol referenced by the executable must be defined by an immediately loaded shared object, instead of a dlopen loaded shared object.
The linker enforces this as well by defaulting to `-z defs` for a `-no-pie/-pie` link.

```c
extern thread_local int ref;
int foo() { return ref; }
```

```asm
# x86-64
movq ref@GOTTPOFF(%rip), %rax
movl %fs:(%rax), %eax
```

Because the offset from the thread pointer to the start of a static block is fixed at program start, such an offset can be encoded by a GOT relocation.
Such relocation types typically have `GOT` and `TPREL/TPOFF` in their names.
Here is a list of common relocation types:

* arm: `R_ARM_TLS_IE32`
* aarch64: `R_AARCH64_TLSIE_ADR_GOTTPREL_PAGE21`, `R_AARCH64_TLSIE_LD64_GOTTPREL_LO12_NC` (`adrp x0, :gottprel:tls; ldr x0, [x0, #:gottprel_lo12:tls]`)
* i386: `R_386_TLS_IE`
* x86-64: `R_X86_64_GOTTPOFF`
* ppc32: `R_PPC_GOT_TPREL16`
* ppc64: `R_PPC64_GOT_TPREL16_HA`, `R_PPC64_GOT_TPREL16_LO_DS`
* riscv: `R_RISCV_TLS_GOT_HI20`, `R_RISCV_PCREL_LO12_I`

If the TLS symbol does not satisfy initial-exec to local-exec optimization requirements, the linker will allocate a GOT entry and emit a dynamic relocation.
Here is a list of dynamic relocation types:

* arm: `R_ARM_TLS_TPOFF32`
* aarch64: `R_AARCH64_TLS_TPREL64`
* mips32: `R_MIPS_TLS_TPREL32`
* mips64: `R_MIPS_TLS_TPREL64`
* i386: `R_386_TPOFF`
* x86-64: `R_X86_64_TPOFF64`
* ppc32: `R_PPC_TPREL32`
* ppc64: `R_PPC64_TPREL64`
* riscv: `R_RISCV_TLS_TPREL64`

While they have `TPREL` or `TPOFF` in their names, these dynamic relocations have the same bitwidth as the word size.
This is a good way to distinguish them from the local-exec relocation types used in object files.

If you add the `__attribute((tls_model("initial-exec")))` attribute, a thread-local variable can use this model in `-fpic` mode.
If the object file is linked into an executable, everything is fine.
If the object file is linked into a shared object, the shared object generally needs to be an immediately loaded shared object.
The linker sets the `DF_STATIC_TLS` flag to annotate a shared object with initial-exec TLS relocations.

glibc ld.so reserves some space in static TLS blocks and allows dlopen on such a shared object if its TLS size is small.
There could be an obscure reason for using such an attribute: general dynamic and local dynamic TLS models are not async-signal-safe in glibc.
However, other libc implementations may not reserve additional TLS space for dlopen'ed initial-exec shared objects, e.g. musl will error.

### General dynamic and local dynamic TLS models (DSO)

The two modes are generally used with `-fpic/-fPIC`, when the TLS symbol may be defined by a shared object.
They do not assume the TLS symbol is backed by a static TLS block.
Instead, they assume that the thread-local storage of the module may be dynamically allocated, making the models suitable for dlopen usage.
The dynamically allocated TLS storage is usually referred to as dynamic TLS.

Each TLS symbol is assigned a pair of (module ID, offset from dtv[m] to the symbol), which is usually referred to as a `tls_index` object.
The module ID m is assigned by the dynamic loader when the module (the executable or a shared object) is loaded, so it is unknown at link time.
dtv means the dynamic thread vector. Each thread has its own dynamic thread vector, which is a mapping from module ID to thread-local storage.
dtv[m] points to the storage allocated for the module with the ID m.

In the simplest form, once we have a pointer to the (module ID, offset from dtv[m] to the symbol) pair, we can get the address of the symbol with the following C program:

```c
// v is a pointer to the first element of the pair.
void *__tls_get_addr(size_t *v) {
  pthread_t self = __pthread_self();
  return (void *)(self->dtv[v[0]] + v[1]);
}
```

The code sequences are inefficient. If the generated relocatable file will not be linked into a dlopen shared object, consider `-ftls-model=initial-exec` to switch to the initial-exec TLS model.

#### General dynamic TLS model (DSO)

The general dynamic TLS model is the most flexible model. It assumes neither the module ID nor the offset from dtv[m] to the symbol is known at link time.
The model is used in `-fpic` mode when the local dynamic TLS model does not apply.
The compiler emits code to set up a pointer to the TLSGD entry of the symbol, then arranges for a call to `__tls_get_addr`.
The return value will contain the runtime address of the TLS symbol in the current thread.
On x86-64, you will notice that the leaq instruction has a data16 prefix and the call instruction has two data16 (0x66) prefixes and one rex64 prefix.
This is a deliberate choice to make the total size of leaq+call to be 16, suitable for link-time optimization.

```asm
data16 leaq def@tlsgd(%rip), %rdi  # R_X86_64_TLSGD
# GNU as does not allow duplicate data16 prefixes, so .value is used here.
.value 0x6666
rex64 call __tls_get_addr@PLT
movl (%rax), %eax
```

(There is an open issue that LLVM disassembler does not display data16 and rex64 prefixes.)

Here is a list of common relocation types. They are called "initial relocations" in _The ELF Handling for Thread-Local Storage_.

* arm: `R_ARM_TLS_GD32`
* aarch64: `R_AARCH64_TLSGD_ADR_PREL21`, `R_AARCH64_TLSGD_ADR_PAGE21`, `R_AARCH64_TLSGD_ADD_LO12_NC`, `R_AARCH64_TLSGD_MOVW_G1`, `R_AARCH64_TLSGD_MOVW_G0_NC` (rarely used because TLS descriptors are the default)
* i386: `R_386_TLS_GD`
* x86-64: `R_X86_64_TLSGD`
* mips: `R_MIPS_TLS_GD`, `R_MICROMIPS_TLS_GD`
* ppc32: `R_PPC_GOT_TLSGD16`
* ppc64: `R_PPC64_GOT_TLSGD16_HA`, `R_PPC64_GOT_TLSGD16_LO`
* riscv: `R_RISCV_TLS_GD_HI20`

When the linker scans such a relocation, it checks whether the referenced TLS symbol satisfy optimization requirements.
If not, the linker allocates two consecutive words in the `.got` section if not allocated yet.
The two entries are relocated by two dynamic relocations.
The dynamic loader will write the module ID to the first word and the offset from dtv[m] to the symbol to the second word.
The dynamic relocation types are:

* arm: `R_ARM_TLS_DTPMOD32` and `R_ARM_TLS_DTPOFF32`
* aarch64: `R_AARCH64_TLS_DTPMOD` and `R_AARCH64_TLS_DTPREL` (rarely used because TLS descriptors are the default)
* x86-32: `R_386_TLS_DTPMOD32` and `R_386_TLS_DTPOFF32`
* x86-64: `R_X86_64_DTPMOD64` and `R_X86_64_DTPOFF64`
* mips32: `R_MIPS_TLS_DTPMOD32` and `R_MIPS_TLS_DTPOFF32`
* mips64: `R_MIPS_TLS_DTPMOD64` and `R_MIPS_TLS_DTPOFF64`
* ppc32: `R_PPC_DTPMOD32` and `R_X86_64_DTPREL32`
* ppc64: `R_PPC64_DTPMOD64` and `R_X86_64_DTPREL64`
* riscv32: `R_RISCV_TLS_DTPMOD32` and `R_X86_64_TLS_DTPREL32`
* riscv64: `R_RISCV_TLS_DTPMOD64` and `R_X86_64_TLS_DTPREL64`
* s390/s390x: `R_390_TLS_DTPMOD` and `R_390_TLS_DTPOFF`

The are called "outstanding relocations" in _The ELF Handling for Thread-Local Storage_.

On x86-64, `-fno-plt` uses `call *__tls_get_addr@GOTPCREL(%rip)` instead of `call __tls_get_addr`.
If linked with GNU ld, a [2016-06 commit](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=e2cbcd9156d1606a9f2153aecd93a89fe6e29180) is needed.
For `-fpic -fno-plt -Wa,-mrelax-relocations=no` compiled relocatable object files, GNU ld cannot be used (<https://sourceware.org/bugzilla/show_bug.cgi?id=24784> is wontfix).

#### Local dynamic TLS model (DSO & non-preemptible)

The local-dynamic TLS model assumes that the offset from dtv[m] to the symbol is a link-time constant.
This case happens when the TLS symbol is non-preemptible.
The compiler emits code to set up a pointer to the TLSLD entry of the module, next arranges for a call to `__tls_get_addr`, then adds a link-time constant to the return value to get the address.

```c
static _Thread_local int def, def1;
int f0() { return ++def; }
int f1() { return ++def1 + def; }
```

The access of `def`, if uses the local-dynamic TLS model, may look like:
```asm
leaq def@tlsld(%rip), %rdi
call __tls_get_addr@PLT
movl def@dtpoff(%rax), %edx
```

I say "the TLSLD entry of the module" because while (on x86-64) `def@tlsld` looks like the TLSLD entry of the non-preemptible TLS symbol, it can really be shared by other non-preemptible TLS symbols.
So one module needs just one such entry.
Technically we can just use general dynamic relocation types to represent the local dynamic TLS model.
For example, GCC riscv does this:

```asm
la.tls.gd a0, .LANCHOR0
call __tls_get_addr@@plt

.section .tbss,"awT",@nobits
.align 2
.set .LANCHOR0, .+0
.type a, @object
.size a, 4
a:
  .zero 4
```

This is clever. However, I would prefer dedicated local-dynamic relocation types.
If we link multiple relocatable files together, the local symbols `.LANCHOR0` are separate and their GOT entries cannot be shared.
Architectures with dedicated local-dynamic relocation types can share the GOT entries.

Note that while the code sequence no longer contains data16 and rex64 prefixes, the code sequence is not shorter than the general-dynamic TLS model.
Actually on RISC architectures the code sequence is usually longer due to the addition of DTPREL.
Local-dynamic is beneficial if a function needs to access two or more non-preemptible TLS symbols, because the `__tls_get_addr` can be shared.

```asm
leaq def0@tlsld(%rip), %rdi
call __tls_get_addr@PLT
movl def0@dtpoff(%rax), %edx
movl def1@dtpoff(%rax), %eax
```

Here is a list of common relocation types.

* arm: `R_ARM_TLS_LDM32`
* i386: `R_386_TLS_LDM`
* x86-64: `R_X86_64_TLSLD`
* mips: `R_MIPS_TLS_LDM`, `R_MICROMIPS_TLS_LDM`
* ppc32: `R_PPC_GOT_TLSLD16`
* ppc64: `R_PPC64_GOT_TLSLD16_HA`, `R_PPC64_GOT_TLSLD16_LO`, `R_PPC64_GOT_TLSLD_PCREL34`

At the linker stage, if the TLS symbol does not satisfy local-dynamic to local-exec optimization requirements, the linker will allocate two consecutive words in the `.got` section for the TLSLD relocation.
The dynamic loader will write the module ID to the first word and the offset from dtv[m] to the symbol to the second word.

If the architecture does not define TLS optimization, the linker can still made an optimization: in `-no-pie/-pie` modes, set the first word to 1 (main executable) and omit the dynamic relocation for the module ID.

### TLS descriptors

Some architectures (arm, aarch64, i386, x86-64) have TLS descriptors as more efficient alternatives to the traditional general dynamic and local dynamic TLS models.
Such ABIs repurpose the first word of the (module ID, offset from dtv[m] to the symbol) pair to represent a function pointer.
The function pointer points to a very simple function in the static TLS case and a function similar to `__tls_get_addr` in the dynamic TLS case.
The caller does an indirection function call instead of calling `__tls_get_addr`.
There are two main points:

* The function call to `__tls_get_addr` uses the regular calling convention: the compiler has to make the pessimistic assumption that all volatile registers may be clobbered by `__tls_get_addr`.
* In glibc (which does lazy TLS allocation), `__tls_get_addr` is very complex. If the TLS of the module is backed by a static TLS block, the dynamic loader can simply place the TP offset into the second word and let the function pointer point to a function which simply returns the second word.

The first point is the prominent reason that TLS descriptors are generally more efficient.
Arguably traditional general dynamic and local dynamic TLS models could have a mechanism to use custom calling convention for `__tls_get_addr` as well.

For the following program, when TLS descriptors are used (`-fpic -mtls-dialect=desc`; `gnu2` for arm and x86), we can see that the resigers holding arguments are not spilled.
```c
__thread int x;
void ext(int a, int b, int c, int d, int e, int f);
int foo(int a, int b, int c, int d, int e, int f) {
  int ret = ++x;
  ext(a, b, c, d, e, f);
  return ret;
}
```

GCC's x86-64 port assumes that `FLAGS_REG` and `RAX` are changed while all other registers are preserved.
In 2024, glibc's x86-64 port fixed the issue that it [did not preserve vector registers in `_dl_tlsdesc_dynamic`](https://sourceware.org/bugzilla/show_bug.cgi?id=31372) (the slow code path).

In musl, in the static TLS case, the two words of the TLS descriptor will be set to `((size_t)__tlsdesc_static, tpoff)` where `__tlsdesc_static` is a function which returns the second word.
glibc's static TLS case is similar.

```asm
.globl __tlsdesc_static
.hidden __tlsdesc_static
__tlsdesc_static:
  # The second word stores the TP offset of the TLS symbol.
  movq 8(%rax), %rax
  ret
```

The scheme optimizes for static TLS but penalizes the case that requires dynamic TLS.
Remember that we have just two words in the GOT and by changing the first word to a function pointer, we have lost information about the module ID.
To retain the information, the dynamic loader has to set the second word to a pointer to a (module ID, offset) pair allocated by malloc.

aarch64 defaults to TLS descriptors. On arm, i386 and x86-64, you can select TLS descriptors via GCC `-mtls-dialect=gnu2`.
RISC-V psABI [specifies](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/373) TLS descriptors in 2023-09. https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/94

(I implemented TLS descriptors and optimization in ld.lld's [IA-32](https://reviews.llvm.org/D112582), x86-64, and [RISC-V](https://github.com/llvm/llvm-project/pull/79239) ports.)

Let's see an example. Loading an `int32_t` using x86-64 TLSDESC has a code sequence like the following:
```asm
leaq    x@TLSDESC(%rip), %rax
call    *x@TLSCALL(%rax)
movl    %fs:(%rax), %eax
```

Let's say the linker does not optimize this sequence into initial-exec or local-exec model, because we are building a shared object.
At run-time, musl rtld resolves the indirect call to load the GOT entry of `__tlsdesc_static`.
When the code executes, the call instruction will call the `__tlsdesc_static` function, loading the TP offset from the second word.
Then the third instruction `movl    %fs:(%rax), %eax` performs a %fs relative memory load.

In GNU ld aarch64/arm/x86-64, `R_*_TLSDESC` relocations are placed in `.rela.plt`.
glibc ld.so has to consider such lazy binding relocations.
However, due to data race, lazy binding is a bad idea so glibc eagerly resolves `R_*_TLSDESC` now.
I filed [`ld: Move R_*_TLSDESC to .rela.dyn`](https://sourceware.org/bugzilla/show_bug.cgi?id=28387).

For the dynamic case, rtld allocates an object with `dlopen` which holds the module ID and the offset from `dtv[m]` to the symbol.
The second GOT entry is rewritten to reference the object.
RISC-V TLS descriptors [explored](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/373) using static entries in place of a runtime `dlopen`, but the idea was turned down.


### PowerPC `__tls_get_addr_opt`

Alan Modra implemented a poor man's TLSDESC scheme for PowerPC in 2015.
<https://sourceware.org/legacy-ml/libc-alpha/2015-03/msg00626.html>

If glibc ld.so sees `DT_PPC64_OPT`, it sets the module ID in the `tls_index` object to zero and sets the TP offset (instead of the offset from dtv[m]).
glibc exports the symbol `__tls_get_addr_opt`.

If GNU ld sees a defined `__tls_get_addr_opt`, it converts a `__tls_get_addr` call to call `__tls_get_addr_opt` instead.
The PLT code sequence checks whether the module ID is zero, and if true, just adds the TP offset to TP and returns;
otherwise calls the `__tls_get_addr_opt` definition in ld.so.

In the common cases for immediately loaded shared objects, this can save a function call.
However, this scheme does not have the benefit of TLSDESC's custom calling convention.

### s390x `__tls_get_offset`

s390 and s390x use `__tls_get_offset` instead of `__tls_get_addr`.
See [Toolchain notes on z/Architecture](https://maskray.me/blog/2024-02-11-toolchain-notes-on-z-architecture) for detail.

### Which model does the compiler pick?

```c
if (executable) { // -fno-pic or -fpie
  if (preemptible)
    initial-exec;
  else
    local-exec;
} else { // -fpic
  if (preemptible || local-dynamic is not profitable)
    general-dynamic;
  else
    local-dynamic;
}
```

The linker uses a similar criterion to check whether TLS optimization apply.

## Link-time TLS optimization

Some psABIs define TLS optimization.
The idea is that the code sequences have fixed forms and are annotated with appropriate relocations, So the linker understands the compiler's intention and can perform 4 kinds of code sequence modification as optimizations.
There are 4 optimization schemes. I have annotated them with the respective condition.

* general-dynamic/TLSDESC to local-exec optimization: `-no-pie/-pie` && non-preemptible
* general-dynamic/TLSDESC to initial-exec optimization: `-no-pie/-pie` && preemptible
* local-dynamic to local-exec optimization: `-no-pie/-pie` (the symbol must be non-preemptible, otherwise it is an error to use local-dynamic)
* initial-exec to local-exec optimization: `-no-pie/-pie` && non-preemptible

I sometimes call the optimization schemes poor man's link-time optimization with nice ergonomics.
Intuitively general-dynamic/TLSDESC to initial-exec optimizations are rare, since it is uncommon to reference a TLS symbol defined in another module.

To make TLS optimization available, the compiler needs to communicate sufficient information to the linker.
So you may find marker relocations which don't relocate values.
Here is a general-dynamic code sequence for ppc64:

```asm
addis r3, r2, x@got@tlsgd@ha # R_PPC64_GOT_TLSGD16_HA
addi r3, r3, x@got@tlsgd@l   # R_PPC64_GOT_TLSGD16_LO
bl __tls_get_addr(x@tlsgd)   # R_PPC64_TLSGD followed by R_PPC64_REL24
```

`R_PPC64_TLSGD` does not relocate the location. It is there to indicate that it is the `__tls_get_addr` function call in the code sequence.

According to Stefan Pintilie, "In the early days of the transition from the ELFv1 ABI that is used for big endian PowerPC Linux distributions to the ELFv2 ABI that is used for little endian PowerPC Linux distributions, there was some ambiguity in the specification of the relocations for TLS."
The `bl __tls_get_addr` instruction was not relocated by `R_PPC64_TLSGD`.
Blindly converting the addis/addi instructions can make the code sequence malformed.
Therefore GNU ld detected the missing `R_PPC64_TLSGD/R_PPC64_TLSLD` and disabled optimization in 2009-03-03.

I was not fond of the fact that we still needed such a hack in 2020 but I implemented a scheme in ld.lld anyway because the request was so strong. <https://reviews.llvm.org/D92959>

The following example tests several optimization schemes.
`a.c` tests general-dynamic to initial-exec optimization.
`b.c:f1` tests local-dynamic to local-exec optimization.
If `f0` uses the general-dynamic TLS model, general-dynamic to local-exec optimization is tested as well.

```c
cat > b.c <<e
__attribute__((visibility("protected"))) _Thread_local int x, y;
int f0() { return ++x; }
int f1() { return ++y + x; }
e
cat > a.c <<e
#include <stdio.h>
extern _Thread_local int x, y;
int f0(); int f1();
int main() {
  f0();
  printf("%d\n", f1());
  printf("%d\n", x + y);
}
e
clang -fpic -O1 -shared b.c -o b.so
clang -fpic -O1 a.c ./b.so -o a
```

We can test general-dynamic to local-exec optimization with `clang -fpic -O1 a.c b.c -o a`.

## TLS variants

There are two variants.

In Variant I, the static TLS blocks are placed above (after) the thread pointer. The thread pointer points to the end of the thread control block.
The thread control block (TCB) is a per-thread data structure describing various attributes of the thread, as defined by the libc implementation.
arm, aarch64, alpha, ia64, m68k, mips, ppc, and riscv use schemes similar to this variant.
I say similar because some architectures (including m68k, mips, powerpc32, powerpc64) place the thread pointer at the end of the thread control block plus a displacement.

Let's say the main executable and two immediately loaded shared objects contain `PT_TLS` segments; then the placement of TCB and static TLS blocks is like the following.
```text
TCB TP_WITHOUT_DISPLACEMENT [GAP] tlsblock0 tlsblock1 tlsblock2

TP_WITHOUT_DISPLACEMENT % tls_align == 0
set TP to TP_WITHOUT_DISPLACEMENT + displacement
If displacement is 0 and GAP is absent, the TP offset of tlsblock0 is exe.tls.p_vaddr&(exe.tls.p_align-1).
```

`TP_WITHOUT_DISPLACEMENT` is aligned according to the maximum `p_align` of all `PT_TLS` segments.

The TP offset of `tlsblock0` is a value that is shared between the linker and the dynamic loader. We can compute the offset relative to `TP_WITHOUT_DISPLACEMENT` as follows:
```c
exe.tls_id = ++tls_cnt; // set tls_cnt to 1
// GAP_ABOVE_TP is 8 for aarch32 and 16 for aarch64
exe.tls.offset = GAP_ABOVE_TP + ((-GAP_ABOVE_TP+exe.tls.p_vaddr) & (exe.tls.p_align-1));
tls_offset = exe.tls.offset + exe.tls.p_memsz;
tls_align = max(exe.tls.p_align, MIN_TLS_ALIGN);
```

As an example, on powerpc64, the displacement is 0x7000, which means TP (the r13 register) is set to the end of the thread control block (`TP_WITHOUT_DISPLACEMENT`) plus 0x7000.
The space allocated for the TLS symbol with `st_value==0` in the executable is at `r13-0x7000 + exe.tls.p_vaddr%exe.tls.p_align`.
Since `exe.tls.p_vaddr%exe.tls.p_align` is normally 0, the code sequence accessing `st_value==0` may look like:
```asm
addis 3, 13, 0
lwz 3, -0x7000(3)
```

A single add instruction can access `[r13-0x8000, r13+0x8000)`, i.e. `[TP_WITHOUT_DISPLACEMENT-0x1000, TP_WITHOUT_DISPLACEMENT+0xf000)`.
If the thread control block is no larger than 0x1000, its member can be accessed with one single add instruction.

arm and aarch64 have a zero displacement, but they reserve two words at TP (a gap before `tlsblock0`). The TP offset of tlsblock0 is `sizeof(void*)*2 + ((p_vaddr-sizeof(void*)*2)&(p_align-1))`.

Dynamic loaders place `tlsblock1` and `tlsblock2` with the minimum alignment padding.
Their offsets are not known by the linker, so theoretically dynamic loaders can add an arbitrary amount of padding.
If we treat the TP without displacement as the anchor point, the offsets of `PT_TLS` segments of immediately loaded shared objects can be determined as follows:
```c
for (int i = 0; i < n_dso_with_tls; i++) {
  p = dso_with_tls[i];
  p->tls_id = ++tls_cnt;
  p->tls.offset = tls_offset + ((-tls_offset+p->tls.p_vaddr) & (p->tls.p_align-1)); // tls_offset = p_vaddr (mod p_align)
  tls_offset = p->tls.offset + p->tls.p_memsz;
  tls_align = max(tls_align, p->tls.p_align);
}
```

---

i386, x86-64, s390, and sparc use Variant II. In Variant II, the static TLS blocks are placed below (before) the thread pointer. The thread pointer points to the start of the thread control block.

Let's say the main executable and two immediately loaded shared objects contain `PT_TLS` segments; then the placement of TCB and static TLS blocks is like the following.
```text
tlsblock2 tlsblock1 tlsblock0 TP TCB

TP % tls_align == 0
The TP offset of tlsblock0 is -exe.tls.p_memsz - ((-exe.tls.p_memsz-exe.tls.p_vaddr)&(exe.tls.p_align-1)).
```

TP is aligned by the maximum `p_align` of all `PT_TLS` segments.

The TP offset of `tlsblock0` is a value that is shared between the linker and the dynamic loader. It can be computed as:
```c
exe.tls_id = ++tls_cnt; // set tls_cnt to 1
tls_offset = exe.tls.p_memsz + ((-(uintptr_t)exe.tls.p_vaddr-exe.tls.p_memsz) & (exe.tls.p_align-1));
exe.tls.offset = -tls_offset;
tls_align = max(exe.tls.p_align, MIN_TLS_ALIGN);
```

If you find the formula above confusing, it is;-) In normal cases, you can forget the alignment requirement and the TP offset of tlsblock0 is just `-p_memsz`.
glibc has a Variant II bug when `p_vaddr%p_align!=0`: [BZ24606](https://sourceware.org/bugzilla/show_bug.cgi?id=24606).
I reported the problem to FreeBSD rtld and fixed the formula for i386/amd64 in <https://github.com/freebsd/freebsd-src/commit/e6c76962031625d51fe4225ecfa15c85155eb13a>.

Dynamic loaders place `tlsblock1` and `tlsblock2` with the minimum alignment padding, though their placement can be more flexible as there offsets are not known by the linker.
```c
for (int i = 0; i < n_dso_with_tls; i++) {
  p = dso_with_tls[i];
  p->tls_id = ++tls_cnt;
  tls_offset += p->tls.p_memsz + ((-p->tls.p_memsz-p->tls.p_vaddr) & (p->tls.p_align-1));
  p->tls.offset = -tls_offset; // -tls_offset = p_vaddr (mod p_align)
  tls_align = max(tls_align, p->tls.p_align);
}
```

## Alignment

For a TLS variable, its alignment describes how its position in the TLS initialization image is aligned.
If ths `PT_TLS` program header satisfies `p_vaddr%p_align==0`, then `st_value` is aligned by the variable alignment as well.

## glibc's TLS implementation

```
// nptl/descr.h
struct pthread
{
  union
  {
#if !TLS_DTV_AT_TP
    tcbhead_t header;
...

// sysdeps/x86_64/nptl/tls.h
typedef struct
{
  void *tcb;
  dtv_t *dtv;
  ...
};
```

On x86-64, `TLS_DTV_AT_TP` is 0. `struct pthread` is at `fs:0`. `dtv` is at `fs:8`.

## Async-signal-safe TLS

C11 7.14.1 Specify signal handling says:

> If the signal occurs other than as the result of calling the abort or raise function, the behavior is undefined if the signal handler refers to any object with static or thread storage duration that is not a lock-free atomic object other than by assigning a value to an object declared as volatile sig_atomic_t, or the signal handler calls any function in the standard library other than the abort function, the _Exit function, the quick_exit function, or the signal function with the first argument equal to the signal number corresponding to the signal that caused the invocation of the handler. Furthermore, if such a call to the signal function results in a SIG_ERR return, the value of errno is indeterminate.

C++11 [support.signal] says:

> An evaluation is signal-safe unless it includes one of the following:
>
> an access to an object with thread storage duration;
>
> A signal handler invocation has undefined behavior if it includes an evaluation that is not signal-safe.

Despite that, accessing TLS from signal handlers can be useful (think of CPU and memory profilers), hence the accesses need to be async-signal safe.
Google reported the issue due to its usage of JVM and dlopen'ed JNI libraries ([Async-signal-safe access to __thread variables from dlopen()ed libraries?](https://sourceware.org/legacy-ml/libc-alpha/2012-06/msg00335.html)).
They eventually resorted to a non-upstream patch which used a custom allocator.

Let's discuss this topic in details.

Local-exec and initial-exec TLS models trivially satisfy the requirement since the size of static TLS blocks is fixed at program start and every thread has a pre-allocated copy.

For a dlopen'ed shared object which uses general-dynamic or local-dynamic TLS model, there are two cases.

* The dynamic loader allocates sufficient storage for all currently running threads at `dlopen` time, and allocates sufficient storage at `pthread_create` time. This is musl's choice. At dlopen time, the dynamic loader needs to block signal deliveray, take a thread list lock and install a new dynamic thread vector for each thread.
* Lazy TLS allocation. TLS allocation is done at the first time `__tls_get_addr` is called. This is glibc and many other libs implementation's choice. The allocation is typically done by malloc, which is not async-signal-safe.

Lazy TLS allocation has the nice property that it does not penalizes the threads which do not need to access TLS of the new shared object.
However, it is difficult to make `__tls_get_addr` async-signal-safe.
It is impossible to both allocate lazily and have dynamic TLS access that cannot fail ([TLS redux](https://sourceware.org/pipermail/libc-alpha/2014-January/047799.html)).
If `__tls_get_addr` cannot allocate memory, the ideal behavior is "fail safe" (e.g. abort), as opposed to the full range of undefined behaviors or deadlock.

One workaround is to let the shared object use the initial-exec TLS model. This will consume the static TLS space - a global resource.

If a dlopen implementing eager TLS allocation is developed, conceivably it may need a new symbol version because there can be programs expecting lazy TLS allocation.

## Large code model

Many 64-bit architectures have a small code model. Some have defined a large code model.
See [Relocation overflow and code models](/blog/2023-05-14-relocation-overflow-and-code-models) for detail.

A small code model usually restricts the addresses and sizes of sections to 4GiB or 2GiB, while a large code model generally makes no such assumption.
The TLS size is usually small and code models and impose some limitation even with a large code model.

For the local-exec TLS model, because a symbol is usually referenced via an offset adding to a register (thread pointer), it needs no distinction with a large code model.

For the initial-exec TLS model, because loading an GOT is needed, and GOT is part of the data sections, a large code model technically should implement a code sequence which is not restricted by the distance between code and data. GCC has not implemented such code sequences.

For the general-dynamic and local-dynamic TLS models, there is usually a GOT load and a `__tls_get_addr` call. As discussed previously, the GOT load needs to be free of 32-bit limitation.
For the `__tls_get_addr` call, on architectures which have implemented range extension thunks, since the linker can redirect the call to a thunk which arranges for the call, no special treatment is needed.

x86-64 has not implemented thunks. Compile a problem with x86-64 `gcc -S -fpic -mcmodel=large` and you can see that the `__tls_get_addr` call is indirect.
This is to prevent the +-2GiB range limitation imposed by the direct CALL instruction.

```asm
movabsq	$_GLOBAL_OFFSET_TABLE_-.L2, %r11
pushq	%rbx
leaq	.L2(%rip), %rbx
addq	%r11, %rbx
leaq	a@tlsgd(%rip), %rdi
movabsq	$__tls_get_addr@PLTOFF, %rax
addq	%rbx, %rax
call	*%rax
popq	%rbx
movl	(%rax), %eax
ret
```

The support for large code model TLS is fairly limited as of today. Most configurations don't lift the GOT load limitation.
On aarch64, `-fpic -mcmodel=large` has not been implemented on GCC and Clang.

## Thread-specific data keys

An alternative to ELF TLS is thread-specific data keys: `pthread_key_create`, `pthread_setspecific`, `pthread_getspecific` and `pthread_key_delete`.
This scheme can be seen as a simpler implementation of `__tls_get_addr` with key reuse feature.
There are C11 equivalents (`tss_create`, `tss_set`, `tss_get`, `tss_delete`) which are rarely used.
Windows provides similar API: `TlsAlloc`, `TlsSetValue`, `TlsGetValue`, `TlsFree`.

The maximum number of keys is usually limited. On glibc it is usually 1024. On musl it is 128.
So applications which potentially need many data keys typically create a wrapper on top of thread-specific data keys, e.g. chromium `base/threading/thread_local_storage.h`.

POSIX.1-2017 does not require `pthread_setspecific`/`pthread_getspecific` to be async-signal-safe.
Nevertheless, most implementations make `pthread_getspecific` async-signal-safe.
`pthread_setspecific` is not necessarily async-signal-safe.

## `-femulated-tls`

`-femulated-tls` uses thread-specific data keys to implement emulated TLS.
It is like using a general-dynamic TLS model for all modes.

```c
__thread int tls0;
extern __thread int tls1;
int foo() { return tls0 + tls1; }
```

```asm
  adrp    x0, :got:__emutls_v.tls0
  ldr     x0, [x0, :got_lo12:__emutls_v.tls0]  // load __emutls_control structure
  bl      __emutls_get_address                 // get load address
  ldr     w19, [x0]                            // load
  adrp    x0, :got:__emutls_v.tls1
  ldr     x0, [x0, :got_lo12:__emutls_v.tls1]  // load __emutls_control structure
  bl      __emutls_get_address                 // get load address
  ldr     w8, [x0]                             // load

  .type   __emutls_v.tls0,@object         // @__emutls_v.tls0
  .data
  .globl  __emutls_v.tls0
  .p2align        3, 0x0
__emutls_v.tls0:                          // A __emutls_control instance
  .xword  4                               // size
  .xword  4                               // alignment
  .xword  0                               // index, initialized to 0
  .xword  0                               // pointer to the initializer
  .size   __emutls_v.tls0, 32
```

Each thread-local variable definition is associated with a `__emutls_control` instance which records size/alignment/index/initializer.

The runtime implementation of `__emutls_get_address` is similar to a `__tls_get_addr` implementation in a lazy TLS allocation scheme.
Each thread has an object array (`emutls_address_array`), with each element being a pointer to the variable value.
Each thread-local variable is assigned an index into the array.

The inefficiency comes from these aspects:

* There is no linker optimization.
* Instead of geting the dynamic thread vector from the thread pointer (usually available in a register), the runtime needs to call `pthread_getspecific` to get the vector.
* The dynamic loader does not know emulated TLS, so the storage allocation is typically done in the access function via `pthread_once`.

libgcc has a mature runtime. In compiler-rt, the runtime was contributed by Android folks in 2015.

Currently Android and OpenBSD targets default to `-femulated-tls` in Clang. See `hasDefaultEmulatedTLS`.

## C++ thread_local

C++ thread_local adds additional features to `__thread`: dynamic initialization before first-use and destruction on thread exit.

> [basic.stc.thread]: A variable with thread storage duration shall be initialized before its first odr-use and, if constructed, shall be destroyed on thread exit.

The implementation chooses to initialize TLS lazily.
If a thread_local variable needs dynamic initialization or has a non-trivial destructor, the compiler calls the TLS wrapper function (`_ZTW*`, in a COMDAT group) instead of referencing the variable directly.
The TLS wrapper calls the TLS init function (`_ZTH*`, weak), which is an alias for `__tls_init`.
`__tls_init` calls the constructors and registers the destructors with `__cxa_thread_atexit`.

The `__cxa_thread_atexit` complexity is because a thread_local variabled defined in a dlopen'ed shared object needs to be destruct at dlclose time before thread exit.
libsupc++ and libc++abi define `__cxa_thread_atexit`. They call `__cxa_thread_atexit_impl` if the libc implementation provides it or use a generic implementation based on thread-specific data keys.

As an example, `x` needs a TLS wrapper function. The compiler may inline the TLS wrapper function and `__tls_init`.

```cpp
extern thread_local int x;
int foo() { return x; }
```

The assembly looks like the following. It uses undefined weak `_ZTH1x` to check whether the TLS init function is defined.
If yes, call the TLS init function. Then reference the variable via usual initial-exec or general dynamic TLS model or TLSDESC.

```asm
_Z3foov:
  pushq %rax
  cmpq $0, _ZTH1x@GOTPCREL(%rip)
  je .LBB0_2
  callq _ZTH1x@PLT
.LBB0_2:
  movq x@GOTTPOFF(%rip), %rax
  movl %fs:(%rax), %eax
  popq %rcx
  retq

.weak _ZTH1x
```

If you know `x` does not need dynamic initialization, C++20 constinit can make it as efficient as the plain old `__thread`.
`[[clang::require_constant_initialization]]` can be used with older language standards.

```cpp
extern thread_local constinit int x;
```

You could also use a pointer and register a thread destructor via another thread-local variable or `pthread_key_create`.
For example, instead of referencing a thread-local variable with destructor directly:
```
//--- a.cc
#include <assert.h>
#include <stdio.h>
#include <thread>
struct A { A(); ~A(); int x; };
static thread_local A a;
A &foo() { return a; }
void th() {
  printf("%d\n", foo().x++);
  printf("%d\n", foo().x++);
}
int main() {
  { std::jthread t(th); }
  th();
}

//--- b.cc
#include <stdio.h>
struct A { A(); ~A(); int x; };
A::A() : x(0) {}
A::~A() { puts("~A"); }
```

You could use a pointer, initialize it once, and use another variable to destroy it.

```cpp
//--- a.cc
#include <assert.h>
#include <stdio.h>
#include <thread>

struct A { A(); ~A(); int x; };
static thread_local A *a;
struct ADtor { ~ADtor() { delete a; }; };
static thread_local ADtor a_dtor;

A &foo() { return *a; }
void init_tls() {
  assert(!a);
  a = new A;
  // Use a_dtor to ensure its dtor will be called at thread exit. https://gcc.gnu.org/bugzilla/show_bug.cgi?id=61991
  (void)&a_dtor;
}

void th() {
  init_tls();
  printf("%d\n", foo().x++);
  printf("%d\n", foo().x++);
}

int main() {
  { std::jthread t(th); }
  th();
}

//--- b.cc
#include <stdio.h>
struct A { A(); ~A(); int x; };
A::A() : x(0) {}
A::~A() { puts("~A"); }
```

Here is an example that `__tls_init` needs to call `__cxa_thread_atexit`.

```cpp
struct S { S(); ~S(); };
thread_local S s;
S &foo() { return s; }
```

## Undefined weak TLS symbols

Regular unresolved weak symbols have a zero value.
TLS symbols are like in a separate address space and the rule doesn't apply.

## macOS TLS

The support was added very late. The scheme is similar to ELF's TLS descriptors, without the good custom calling convention promise.
In other words, the performance is likely worse than ELF's general dynamic TLS model.
To my surprise, thread-local variables of internal linkage need an indirect function call, too.

```cpp
thread_local int tls;
int f() { return tls; }
```

```asm
// x86-64
movq _tls@TLVP(%rip), %rdi     // X86_64_RELOC_TLV(_tls)
callq *(%rdi)                  // calls getAddrFunc in libdyld.dylib
movl (%rax), %eax
```

```asm
adrp x0, _tls@TLVPPAGE         // ARM64_RELOC_TLVP_LOAD_PAGEOFF12(_tls)
ldr x0, [x0, _tls@TLVPPAGEOFF] // ARM64_RELOC_TLVP_LOAD_PAGE21(_tls)
ldr x8, [x0]
blr x8                         // x0 = &tls
ldr w0, [x0]                   // x0 = tls
```

`_tls@TLVP` refers a TLV descriptor in `__DATA,__thread_vars`. Each descriptor consists of three words:

* `void* (*thunk)(struct TLVDescriptor*)`
* `unsigned long key`: thread-specific data key (unique per library) denoting the TLS block
* `unsigned long offset`: the offset in the block

[dyld](https://github.com/apple-oss-distributions/dyld) iterates the the descriptors and sets the first word (`thunk`) to `getAddrFunc`.

`__attribute__((tls_model(...)))` attributes are ignored in Clang.

## Windows TLS

The code sequence fetches `ThreadLocalStoragePointer` (offset 88) out of the Thread Environment Block and indexes it by `_tls_index`.
The return value is indexed with the offset of the variable from the start of the `.tls` section.
The scheme is similar to ELF's local-dynamic TLS model, replacing a `__tls_get_desc` call with an array index operation.

```asm
movl _tls_index(%rip), %eax
movq %gs:88, %rdx
movq (%rdx,%rax,8), %rax
movl %ecx, tls@SECREL32(%rax)
```

Referencing a TLS variable from another DLL is not supported.

```cpp
__declspec(dllimport) extern thread_local int tls;
// error C2492: 'tls': data with thread storage duration may not have dll interface
```

There are a lot of of details but my personal understanding of Windows does not allow me to say more ;-)
Interested readers can go to [Thread Local Storage, part 3: Compiler and linker support for implicit TLS](http://www.nynaeve.net/?p=183).


## libc API for TLS blocks

Sanitizers' runtime needs TLS blocks for a variety of use cases.
See <https://sourceware.org/bugzilla/show_bug.cgi?id=16291> for a glibc feature request.
Read on for a detailed description.

In LLVM, OrcJIT has a desire to register TLS blocks.
Lang Hames told me that he has got native TLS working by implementing dyld’s TLS support APIs in the Orc runtime.

Florian Weimer posted [Thread properties API](https://www.openwall.com/lists/libc-coord/2021/05/21/1) in 2021-05.

## Why does compiler-rt need to know TLS blocks?

### AddressSanitizer "asan" (`-fsanitize=address`)

The main task of AddressSanitizer is to detect addressability problems. If a regular memory byte is not addressable (i.e. accesses should be UB), it is said to be poisoned and the associated shadow encodes the addressability information (all unpoisoned/all poisoned/partly poisoned).

On thread creation, the runtime should unpoison the thread stack and static TLS blocks to allow accesses. (`test/asan/TestCases/Linux/unpoison_tls.cpp`; introduced in [`[asan] Make ASan report the correct thread address ranges to LSan.`](https://github.com/llvm/llvm-project/commit/09886cd17ab8e5e601fda0e2aa21ff28c1a8fa63))
The runtime additionally unpoisons the thread stack and TLS blocks on thread exit to allow accesses from later TSD destructors.

Note: if the allocation is rtld/libc internal and not intercepted, there is no need to unpoison the range. The associated shadow is supposed to be zeros.
However, if the allocation is intercepted, the runtime should unpoison the range in case the range reuses a previous allocation which happens to contain poisoned bytes.

In glibc, `_dl_allocate_tls` and `_dl_deallocate_tls` call malloc/free functions which are internal and not intercepted, so the allocations are opaque to the runtime and the shadow bytes are all zeroes.

### Hardware-assisted AddressSanitizer "hwasan" (`-fsanitize=hwaddress`)

Its `ClearShadowForThreadStackAndTLS` is similar to asan's.

### LeakSanitizer "lsan" (`-fsanitize=leak`)

LeakSanitizer detects memory leaks. On many targets, it is integrated (and enabled by default) in AddressSanitizer, but it can be used standalone.
The checker is triggered by an `atexit` hook (the default options are `LSAN_OPTIONS=detect_leaks=1:leak_check_at_exit=1`), but it can also be invoked via `__lsan_do_leak_check`.

Each supported platform provides an entry point: `StopTheWorld` (e.g. Linux [1]), which does the following:

* Invoke the clone syscall to create a new process which shared the address space with the calling process.
* In the new process, list threads by iterating over `/proc/$pid/task/`.
* In the new process, call `SuspendThread` (ptrace `PTRACE_ATTACH`) to suspend a thread.

`StopTheWorld` returns. The runtime performs mark-and-sweep, reports leaks, and then calls `ResumeAllThreads` (ptrace `PTRACE_DETACH`).

Note: the implementation cannot call libc functions. It does not perform code injection. The toot set includes static/dynamic TLS blocks for each thread.

(The `pthread_create` interceptor calls `AdjustStackSize` which computes a minimum stack size with `GetTlsSize`. <https://code.woboq.org/llvm/compiler-rt/lib/sanitizer_common/sanitizer_posix_libcdep.cpp.html#411> I am not sure musl needs this.)

Intercepting `__tls_get_addr` is useful to lsan but is not necessary.
First, the Linux `InitializePlatformSpecificModules` implementation ignores leaks from the dynamic loader.
Second, allocations called by `__tls_get_addr` are suppressed by a built-in rule `leak:*tls_get_addr` in `kStdSuppressions`.

The current lsan implementation has more requirement on `GetTls`: it does not intercept `pthread_setspecific`.
Instead, it expects `GetTls` returned range to include pointers to `pthread_setspecific` regions, otherwise there would be false positive leak reports.

In addition, lsan gets the static TLS boundaries at `pthread_create` time and expects the boundaries to include TLS blocks of dynamically loaded modules.
This means that `GetTls` returned range needs to include static TLS surplus.

(
You might ask that the thread control block has the dtv pointer, why can't lsan track the referenced allocations.
Well, for threads, rtld/libc implementations typically allocate the static TLS blocks as part of the thread stack, which are not seen by the runtime, so the runtime does not know the allocations.
)

On glibc, `GetTls` returned range includes `pthread::{specific_1stblock,specific}` for thread-specific data keys.
There is currently a hack to ignore allocations from ld.so allocated dynamic TLS blocks.
Note: if the `pthread::{specific_1stblock,specific}` pointers are encrypted, lsan cannot track the allocation.

[1]: https://code.woboq.org/llvm/compiler-rt/lib/sanitizer_common/sanitizer_stoptheworld_linux_libcdep.cpp.html#144

### MemorySanitizer "msan" (`-fsanitize=memory`)

MemorySanitizer detects uses of uninitialized memory. If a regular memory byte has uninitialized (poisoned) bits, its associated shadow byte has one bits.

Similar to asan.
On thread creation, the runtime should unpoison the thread stack and static TLS blocks to allow accesses. (`test/msan/tls_reuse.cpp`)
The runtime additionally unpoisons the thread stack and TLS blocks on thread exit to allow accesses from TSD destructors.

msan needs to do more than asan: the `__tls_get_addr` interceptor (`DTLS_on_tls_get_addr`) detects new dynamic TLS blocks and unpoisons the shadow.
ld.so calls a non-interposable `memset` to clear the blocks.
Otherwise, if a dynamic TLS block reuses a previous allocation with poison, there may be false positives.
One way to semi reliably trigger this is (`test/msan/dtls_test.cpp` <https://github.com/google/sanitizers/issues/547>):

* in a thread, write an uninitialized (poisoned) value to a dynamic TLS block
* destroy the thread
* create a new thread
* try making the new thread reuse the poisoned dynamic TLS block.

Note: aarch64 uses TLSDESC by default and there is no interposable symbol.

During the development of glibc 2.19, [commit 1f33d36a8a9e78c81bed59b47f260723f56bb7e6](https://sourceware.org/git/?p=glibc.git;a=commit;h=1f33d36a8a9e78c81bed59b47f260723f56bb7e6) ("Patch 2/4 of the effort to make TLS access async-signal-safe.") was checked in.
`DTLS_on_tls_get_addr` detects the `__signal_safe_memalign` header and considers it a dynamic TLS block if the block is not within the static TLS boundaries.
[commit dd654bf9ba1848bf9ed250f8ebaa5097c383dcf8](https://sourceware.org/git/?p=glibc.git;a=commit;h=dd654bf9ba1848bf9ed250f8ebaa5097c383dcf8) ("Revert "Patch 2/4 of the effort to make TLS access async-signal-safe.") reverted `__signal_safe_memalign`, but the implementation remains in grte branches.

See also [Re: glibc 2.19 - asyn-signal safe TLS and ASan.](https://groups.google.com/g/address-sanitizer/c/BfwYD8HMxTM)

Similar to lsan: the `pthread_create` interceptor calls `AdjustStackSize` which computes a minimum stack size with `GetTlsSize`.

### ThreadSanitizer "tsan" (`-fsanitize=thread`)

Similar to lsan: the `pthread_create` interceptor calls `AdjustStackSize` which computes a minimum stack size with `GetTlsSize`.

Similar to msan, the runtime unpoisons TLS blocks to avoid false positives.
Tested by `test/tsan/dtls.c` (D20927).
tsan also needs to intercept `__tls_get_addr`. The problem that aarch64 TLSDESC does not have an interposable symbol also applies.

I wrongly thought <https://reviews.llvm.org/D93866> was a workaround. <https://sourceware.org/pipermail/libc-alpha/2021-January/121352.html> explained that the code has not materialized changed since 2012.

For dynamic TLS blocks, older glibc (e.g. 2.23) calls `__libc_memalign`, which is intercepted (`tsan/rtl/tsan_interceptors_posix.cpp`); since BZ #17730, newer glibc (e.g. 2.32) calls `malloc`.

### NumericalSanitizer "nsan" (`-fsanitize=numerical`)

Similar to msan and dfsan, the runtime unpoisons TLS blocks to avoid false positives ([#102718](https://github.com/llvm/llvm-project/pull/102718)).

### glibc TLS allocation

For dynamic TLS blocks, `allocate_and_init` allocates the block.

For a new thread, glibc for Variant II ports allocates a memory region, places the static TLS block (which contains the pthread structure) at the end (`nptl/allocatestack.c`), and uses the remaining space as the thread stack.
The stack pointer may be just a few hundred bytes below the canary address (%fs:0x28 on x86-64) and a large oob write can potentially override it.

### Android bionic

Android bionic (API level 31) introduced some TLS APIs in `libc/include/sys/thread_properties.h`.
`__libc_get_static_tls_bounds` and `__libc_iterate_dynamic_tls` are used in compiler-rt.

```c
/**
 * Gets the bounds of static TLS for the current thread.
 *
 * Available since API level 31.
 */
void __libc_get_static_tls_bounds(void** __static_tls_begin,
                                  void** __static_tls_end) __INTRODUCED_IN(31);

/**
 * Iterates over all dynamic TLS chunks for the given thread.
 * The thread should have been suspended. It is undefined-behaviour if there is concurrent
 * modification of the target thread's dynamic TLS.
 *
 * Available since API level 31.
 */
void __libc_iterate_dynamic_tls(pid_t __tid,
                                void (*__cb)(void* __dynamic_tls_begin,
                                             void* __dynamic_tls_end,
                                             size_t __dso_id,
                                             void* __arg),
                                void* __arg) __INTRODUCED_IN(31);
```

dalias's notes

```
<@dalias> i think the api proposed there looks wrong
<@dalias> e.g. "static tls bounds" supposes a particular implementation where static is a single block range and static and dynamic are distinct
<@dalias> the interfaces proposed for dynamic are even worse
<@dalias> allowing interposition of individual dynamic tls area creation
<@dalias> supposing that they're created individually and ignoring that any interposition here would be extremely unsafe
<@dalias> the alternative prposed __libc_iterate_dynamic_tls is just a renamed dl_iterate_phdr without the glibc bug
<@dalias> and is pointless -- just fix the glibc bug
<@dalias> "When a thread (or dynamic TLS) is destroyed, the shadow for the stack (or dynamic TLS) should be unpoisoned"
<@dalias> this is backwards -- it should be poisoned because it's no longer valid. the stated desired behavior is based on bad glibc implementation internals (reuse of the stack/tls memory) and ignores that something should be done to unpoison it at the moment it's reused, not when it's freed for reuse
```

Test TLS:
```sh
cat > ./a.c <<eof
#include <assert.h>
int foo();
int bar();
int main() {
  assert(foo() == 2);
  assert(foo() == 4);
  assert(bar() == 2);
  assert(bar() == 4);
}
eof

cat > ./b.c <<eof
__thread int tls0;
extern __thread int tls1;
int foo() { return ++tls0 + ++tls1; }
static __thread int tls2, tls3;
int bar() { return ++tls2 + ++tls3; }
eof

echo '__thread int tls1;' > ./c.c

sed 's/        /\t/' > ./Makefile <<'eof'
.MAKE.MODE = meta curDirOk=true

CC := gcc -m32 -g -fpic -mtls-dialect=gnu2
LDFLAGS := -m32 -Wl,-rpath=.

all: a0 a1 a2

run: all
        ./a0 && ./a1 && ./a2

c.so: c.o; ${LINK.c} -shared $> -o $@
bc.so: b.o c.o; ${LINK.c} -shared $> -o $@
b.so: b.o c.so; ${LINK.c} -shared $> -o $@

a0: a.o b.o c.o; ${LINK.c} $> -o $@
a1: a.o b.so; ${LINK.c} $> -o $@
a2: a.o bc.so; ${LINK.c} $> -o $@
eof
```

```sh
bmake run && bmake CFLAGS=-O1 run
```

## Misc

On AArch32, many TLS code sequences need constant pool, making TLS incompatible with `-mexecute-only`.
