layout: post
title: MMU-less systems and FDPIC
author: MaskRay
tags: [linux,linker,riscv,fdpic]
---

This article describes ABI and toolchain considerations about systems without a Memory Management Unit (MMU).
We will focus on FDPIC and the in-development FDPIC ABI for RISC-V, with updates as I delve deeper into the topic.

Embedded systems often lack MMUs, relying on real-time operating systems (RTOS) like VxWorks or special Linux configurations (`CONFIG_MMU=n`).
In these systems, the offset between the text and data segments is often not knwon at compile time.
Therefore, a dedicated register is typically set to somewhere in the data segment and writable data is accessed relative to this register.

Why is the offset not knwon at compile time?
There are primarily two reasons.

First, eXecute in Place (XIP), where code resides in ROM while the data segment is copied to RAM.
Therefore, the offset between the text and data segments is often not knwon at compile time.

Second, all processes share the same address space without MMU.
However, it is still desired for these processes to share text segments.
Therefore needs a mechanism for code to find its corresponding data.

<!-- more -->

## Compiler support for unknown text-data segment offset

### `-msep-data`

GCC's m68k port [added `-msep-data`](https://gcc.gnu.org/pipermail/gcc-patches/2003-September/113699.html) in 2003-10.

> Add -msep-data and -mid-shared-library support for uClinux. These are two special PIC variants that allow executing linux applications in ROM filesystems without loading an additional copy in memory (XIP).
>
> With -msep-data, references to global data are made through register A5 which is loaded with a pointer to the start of the data/bss segment allocated in RAM.
>
> The -mid-shared-library option allows using a special shared library flavour that allows allocationg a distinct data/bss section for each process without the need to relocate code in both library and application.

`-msep-data` is PIC only and updates `-fno-pic` to `-fPIE`. In this mode, a5 is read-only and holds the address of `_GLOBAL_OFFSET_TABLE_`.
When not used with `-mid-shared-library`, `-fPIC -msep-data` is unnecessary. Just stick with `-fPIE -msep-data`.

### `-mid-shared-library`

[This option](https://gcc.gnu.org/onlinedocs/gcc/M680x0-Options.html#:~:text=mid) was added to GCC's m68k port along with `-msep-data`. The documentation says:

> Generate code that supports shared libraries via the library ID method. This allows for execute-in-place and shared libraries in an environment without virtual memory management. This option implies -fPIC.

`-mid-shared-library` is PIC only and updates `-fno-pic` to `-fPIE`.
You compile a source file with `-mid-shared-library -mshared-library-id=n`, and the functions will be attached to library ID n.
At function entry a5 points to an array that maps a library ID to the corresponding GOT base address.
The compiler generates `move.l -(n+1)*4(%a5),%a5` to obtain the actual GOT base address.
The a5 will then be used to access the corresponding data segment.

`gcc/config/bfin` added `-msep-data` in 2006.

### `-mno-pic-data-is-text-relative`

This ARM option is similar to `-msep-data` and only makes sense with `-fpie`/`-fpic`. In 2013, `-mno-pic-data-is-text-relative`, generalized from the ARM [VxWorks RTP](https://gcc.gnu.org/pipermail/gcc-patches/2007-May/217111.html) port, was [added](https://inbox.sourceware.org/gcc-patches/000001cedf74$bd1bf710$3753e530$@arm.com/) to assume that text and data segments don't have a fixed displacement.
On non-VxWorks-RTP targets, `-mno-pic-data-is-text-relative` implies [`-msingle-pic-base`](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html#index-msingle-pic-base):

> Treat the register used for PIC addressing as read-only, rather than loading it in the prologue for each function. The runtime system is responsible for initializing this register with an appropriate value before execution begins.

r9 is used as the static base (`arm_pic_register`) in the position-independent data model to access the data segment.
Since r9 is not changed, dynamic linking seems unsupported as a DSO needs a different data segment.

GCC's s390x port added `-mno-pic-data-is-text-relative` in 2017 [for kpatch](https://github.com/dynup/kpatch/commit/10002f5aa671de2878252aaa48f585457d39638a) (live kernel patching).

### `-fropi` and `-frwpi`

* [[RFC][ARM] Add support for embedded position-independent code (ROPI/RWPI)](https://lists.llvm.org/pipermail/llvm-dev/2015-December/093022.html)
* [[ARM] Add support for embedded position-independent code](https://reviews.llvm.org/D23195)
* [[ARM] Command-line options for embedded position-independent code](https://reviews.llvm.org/D23196)

<!-- GlobalIsel abb088691b6830ebe73a24b1611569bcc2c5cf90 -->

Clang ARM's `-fropi` and `-frwpi` are special `-fno-pic` variants that only intended for static linking.
While regular `-fno-pic` assumes absolute addressing for both code data, `-fropi` and `-frwpi` add a twist by enforcing relative addressing based on specific assumptions about relocation.
Both options consider the text-data segment offset unknown at compile time.

* `-fropi` assumes code and read-only data will be relocated at runtime, making absolute addressing unsuitable. Instead, PC-relative addressing is used. The `.ARM.attributes` section contains `Tag_ABI_PCS_RO_data: 1` like `-fpic`.
* `-frwpi` assumes writable data will be relocated at runtime, making absolute addressing unsuitable. Instead, writable data is accessed relative to the static base register. The `.ARM.attributes` section contains `Tag_ABI_PCS_RW_data: 2`.

You can use `-fropi` and `-frwpi` together to require relative addressing for both code and data.
Compared with `-fno-pic -frwpi`, `-fno-pic -fropi -frwpi` needs one more instruction to retrieve a function address.

In terms of semantics, I think `-fno-pic -fropic -frwpic` is identical to `-fpie -mno-pic-data-is-text-relative` with hidden visibility declarations.
In practice, GCC `-fpie -mno-pic-data-is-text-relative` utilizes GOT-relative relocations (`R_ARM_GOT_BREL`), not MOVW/MOVT instructions.

### `-mfdpic`

We will discuss this in detal later.

### Compiler option summary

`-msep-data` and `-mno-pic-data-is-text-relative` are the same, relying on `-fpie/-fpic` semantics to enforce relative addressing for the text segment.
`-fropi` and `-frwpi` offer finer control. You can choose to use relative addressing for text segment only (`-fropi`), data segment only (using `-frwpi`), or both.

Neither `-msep-data` nor `-fropi -frwpi` supports shared libraries.
`-msep-data`'s variant `-mid-shared-library` provides a library ID based shared library, which works for some cases but is inflexible.

---

Now, let's review OS support.
While I'm not an RTOS expert, let's explore Linux's executable file loaders and see how they handle MMU-less scenarios.

## Linux binfmt loaders

`fs/Kconfig.binfmt` defines a few loaders.

* `BINFMT_ELF` defaults to y and depends on `MMU`.
* `BINFMT_ELF_FDPIC` defaults to y when `BINFMT_ELF` is not selected. A few architecture support `BINFMT_ELF_FDPIC` for NOMMU. ARM supports FDPIC even with a MMU.
* `BINFMT_FLAT` is provided for a few architectures.

Therefore, both `BINFMT_ELF_FDPIC` and `BINFMT_FLAT` can be used for MMU-less systems.
`BINFMT_FLAT` is a very old solution that does not allow dynamic linking while `BINFMT_ELF_FDPIC` supports dynamic linking.

BTW, `BINFMT_AOUT`, removed in 2022, had been supported for alpha/arm/x86-32.

## Binary flat format

Linux's `BINFMT_FLAT` refers to an object file format used by [μClinux](https://en.wikipedia.org/wiki/%CE%9CClinux): Binary Flat format (BFLT).
<https://myembeddeddev.blogspot.com/2010/02/uclinux-flat-file-format.html> has an introduction.
BFLT is an executable file only format, not used for relocatable files. An executable file is typically converted from ELF using [elf2flt](https://github.com/uclinux-dev/elf2flt).
`ld-elf2flt` is a ld wrapper that invokes `elf2flt` when the option `-elf2flt` is seen.

Linux's `BINFMT_FLAT` supports both version 2 (`OLD_FLAT_VERSION`) and version 4.
Version 4 supports eXecute in Place (XIP), where code resides in ROM while the data segment is copied to RAM.
Therefore, the offset between the text and data segments is often not knwon at compile time.

Greg added [ID-based shared library support](https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=3d97dc2d349e6630bced9ced2ca7d0c7b52e49bc) to be used with [`-mid-shared-library`](#mid-shared-library) in 2003, which was [removed](https://git.kernel.org/linus/70578ff3367dd4ad8f212a9b5c05cffadabf39a8) in April 2022.
The code supported one executable and at most 3 shared libraries.

The tooling for shared library support seems to be called eXtended FLAT (XFLAT). It is a limited shared library scheme that disallows global variable sharing. Quoting [XFLAT FAQ](https://xflat.sourceforge.net/XFlatFAQ.html):

> XFLAT provides an alternative mechanism to bind and relocate functions using a thunk layer that is inserted between each inter-module function call. However, without a GOT it is not possible to bind and relocate data. In short, with no GOT XFLAT cannot support sharing of global variables between program and shared library modules.

## FDPIC

FDPIC can be seen as an extended `-mno-pic-data-is-text-relative` mode that utilizes function descriptors to support PIC register changes for dynamic linking.
A FDPIC executable can be loaded using either the regular Linux ELF loader for MMU systems or `fs/binfmt_elf_fdpic.c` for MMU-less systems.
`fs/binfmt_elf_fdpic.c` has been [available](https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=be3e0ada7b264efa8714b1ff6203c2c29fcc61c5) since 2002.
It supports both MMU and NOMMU configurations but does not support `ET_EXEC` executables in NOMMU mode.
Each architecture that supports FDPIC defines an `EI_OSABI` value to be checked by the loader.

Several architectures define a FDPIC ABI.

* [Fujitsu FR-V](https://en.wikipedia.org/wiki/FR-V_(microprocessor)): [_The FR-V FDPIC ABI_](https://www.fsfla.org/~lxoliva/writeups/FR-V/FDPIC-ABI.txt), initial version in 2004
* [ADI Blackfin](https://en.wikipedia.org/wiki/Blackfin): _Blackfin FDPIC ABI_
* [SuperH](https://en.wikipedia.org/wiki/SuperH): [_The SH FDPIC ABI_](https://sourcery.sw.siemens.com/public/docs/sh-fdpic/sh-fdpic-abi.txt), initial version in 2008
* [_ARM FDPIC ABI_](https://github.com/mickael-guene/fdpic_doc), initial version in 2013. _ARM FDPIC Toolchain and ABI_ provides a good summary.

Here is a summary.

The read-only sections, which can be shared, are commonly referred to as the "text segment", whereas the writable sections are non-shared and commonly referred to as the "data segment".
Functions and certain data symbols (`.rodata`) reside in the text segment, while other data symbols and the GOT reside in the data segment.
Special entries called "canonical function descriptors" also reside in the GOT.

A call-clobbered register is reserved as the FDPIC register, used to access the data segment. Upon entry to a function, the FDPIC register holds the address of `_GLOBAL_OFFSET_TABLE_`.
The text segment can be referenced using PC-relative addressing.
The data segment including GOT is referenced using indirect FDPIC-register-relative addressing.
We will see later that sometimes it's unknown whether a non-preemptible symbol resides in the text segment or the data segment, in which case GOT-indirect addressing with the FDPIC register has to be used.

A function call is called external if the destination may reside in another module, which has a different data segment and therefore needs a different FDPIC register value.
Therefore, an external function call needs to update the FDPIC register as well as changing the program counter (PC).
The FDPIC register can be spilled into a stack slot or a call-saved register, if the caller needs to reference the data segment later.
The FDPIC register is call-clobbered to [allow external tail calls](/blog/2021-09-19-all-about-procedure-linkage-table#got-setup-is-expensive-without-pc-relative-addressing) and avoid PLT saving the register.

Calling a function pointer, including calling a PLT entry, also sets both the FDPIC register and PC.
When the address of a function is taken, the address of its canonical function descriptor is obtained, not that of the entry point.
The descriptor, resides in the GOT, contains pointers to both the function's entry point and its FDPIC register value.
The two GOT entries are relocated by a dynamic relocation of type `R_*_FUNCDESC_VALUE` (e.g. [`R_FRV_FUNCDESC_VALUE`](https://www.fsfla.org/~lxoliva/writeups/FR-V/FDPIC-ABI.txt#:~:text=The%20R_FRV_FUNCDESC_VALUE%20relocation%20is%20used%20to)).

If the symbol is preemptible, the code sequence loads a GOT entry.
When the symbol is a function, the GOT entry is relocated by a dynamic relocation `R_*_FUNCDESC` and will contain the address of the function descriptor address.

Let's checkout examples taking addresses of functions and variables.
```c
__attribute__((visibility("hidden"))) void hidden_fun();
void fun();
__attribute__((visibility("hidden"))) extern int hidden_var;
extern int var;
__attribute__((visibility("hidden"))) const int ro_hidden_var = 42;

void *addr_hidden_fun() { return hidden_fun; }
void *addr_fun() { return fun; }
void *addr_hidden_var() { return &hidden_var; }
void *addr_var() { return &var; }
const int *addr_ro_hidden_var() { return &ro_hidden_var; }
int read_hidden_var() { return hidden_var; }
int read_var() { return var; }
```

### Function access in FDPIC

Canonical function descriptors are stored in the GOT, and their access depends on whether the referenced function is preemptible or not.

* For non-preemptible functions: the address of the descriptor is directly computed by adding an offset to the FDPIC register.
* For preemptible functions: a GOT entry is loaded first. This entry, relocated by a `R_*_FUNCDESC` dynamic relocation, holds the final address of the function descriptor.

```asm
// arm-linux-gnueabihf-gcc -c -fpic -mfdpic -Wa,--fdpic
addr_hidden_fun: // non-preemptible function
  ldr     r0, .L3            // r0 = &.got[n] - FDPIC
  add     r0, r0, r9         // r0 = &.got[n]; the address of the canonical function descriptor
  ...
.L3:
// Linker resolves this to &.got[n] - FDPIC. .got[n], relocated by R_ARM_FUNCDESC_VALUE, is the canonical function descriptor.
  .word   hidden_fun(GOTOFFFUNCDESC) // R_ARM_GOTOFFFUNCDESC(hidden_fun)

addr_fun: // preemptible function
  ldr     r3, .L6            // r3 = &.got[n] - FDPIC
  ldr     r0, [r9, r3]       // r0 = &.got[n]; the address of the canonical function descriptor
  ...
.L6:
// Linker resolves this to &.got[n] - FDPIC. .got[n], relocated by R_ARM_FUNCDESC, will contain the address of the canonical function descriptor.
  .word   fun(GOTFUNCDESC)   // R_ARM_GOTFUNCDESC(fun)
```

Unfortunately, when linking a DSO, an `R_ARM_GOTOFFFUNCDESC` relocation referencing a hidden symbol results in a linker error.
This error likely arises because the generated `R_ARM_FUNCDESC_VALUE` dynamic relocation requires a dynamic symbol.
While this can be implemented using an `STB_LOCAL STT_SECTION` dynamic symbol, GNU ld currently lacks support for this approach.

```
% arm-linux-gnueabihf-gcc -fpic -mfdpic -O2 -Wa,--fdpic q.c -shared
/tmp/ccxpnij8.o: in function `addr_hidden_fun':
q.c:(.text+0x10): dangerous relocation: no dynamic index information available
collect2: error: ld returned 1 exit status
```

Let's try sh4. `sh4-linux-gnu-gcc -fpic -mfdpic -O2 q.c -shared -nostdlib` allows taking the address of a hidden function but not a protected function (my [pending fix](https://sourceware.org/pipermail/binutils/2024-February/132519.html)).

Then, let's see a global variable initialized by the address of a function and a C++ virtual table.
```cpp
struct A { virtual void foo(); };
void call(A *a) { a->foo(); }
auto *var_call = call;
```

```asm
// arm-linux-gnueabihf-g++ -c -fpic -mfdpic -Wa,--fdpic
  ldr     r3, [r0]           // load vtable
...
  ldr     r3, [r3]           // load vtable entry `.word _ZN1A3fooEv(FUNCDESC)`
  ldr     r9, [r3, #4]       // load FDPIC register value
  ldr     r3, [r3]           // load foo's entry point
  blx     r3

.section        .data.rel,"aw"
var_call:
// Function descriptor address, relocated by R_ARM_FUNCDESC dynamic relocation
  .word   _Z4callP1A(FUNCDESC) // R_ARM_FUNCDESC

.section        .data.rel.ro,"aw"
_ZTV1A:
  .word   0
  .word   _ZTI1A
// Function descriptor address, relocated by R_ARM_FUNCDESC dynamic relocation
  .word   _ZN1A3fooEv(FUNCDESC) // R_ARM_FUNCDESC
```

TODO: `-fexperimental-relative-c++-abi-vtables`

### Data access in FDPIC

GOT-indirect addressing is required for accessing data symbols under two conditions:

* Preemptible symbols: Traditional GOT requirement.
* Non-preemptible symbols with potential data segment placement: This includes
  + Writable data symbols: This covers both locally declared (`int var;`) and externally declared (`extern int var;`) non-const variables.
  + Potential dynamic initialization: `const A a; extern const int var;`
  + Certain guaranteed constant initialization: `extern constinit const int *const extern_const;`. Constant initialization may require a relocation, e.g. `constinit const int *const extern_const = &var;`

```asm
addr_hidden_var: // non-preemptible data with potential data segment placement
  ldr     r3, .L9             // r3 = &.got[n] - FDPIC
  ldr     r0, [r9, r3]
  ...
.L9:
// Linker resolves this to &.got[n] - FDPIC. .got[n], relocated by R_ARM_RELATIVE, will contain the address of hidden_var.
  .word   hidden_var(GOT)    // R_ARM_GOT_BREL

addr_var: // preemptible data
  ldr     r3, .L12
  ldr     r0, [r9, r3]
  ...
.L12:
// Linker resolves this to &.got[n] - FDPIC. .got[n], relocated by R_ARM_GLOB_DAT, will contain the address of var.
  .word   var(GOT)           // R_ARM_GOT_BREL
```

The dynamic relocations `R_*_RELATIVE` and `R_*_GLOB_DAT` do not use the standard `+ load_base` semantics.
It seems that musl fdpic doesn't support the special `R_*_RELATIVE`.

If the referenced data symbol is non-preemptible and guaranteed to be in the text segment, we can use PC-relative addressing.
However, this scenario is remarkably rare in practice.
The most likely use case is like the following:

```cpp
const int ro_array[] = {1, 2, 3, 4}; // text segment

int read_ro_array_elem(int i) { return ro_array[i]; }
```

GCC's arm port does not seem to utilize PC-relative addressing. We can try GCC's SuperH port:

```asm
// sh4-linux-gnu-gcc -S -fpic -mfdpic -O2 q.c
addr_hidden_var: // non-preemptible data
  mov.l   .L12,r0
  rts
  add     r12,r0
.L13:
  .align 2
.L12:
  .long   hidden_var@GOTOFF

addr_ro_hidden_var: // non-preemptible data
  mov.l   .L18,r0
  rts
  mov.l   @(r0,r12),r0
.L19:
  .align 2
.L18:
  .long   ro_hidden_var@GOT
```

It optimizes `addr_hidden_var` but not `addr_ro_hidden_var`.

### Thread-local storage in FDPIC

_ARM FDPIC ABI_ defines static TLS relocations `R_ARM_TLS_GD32_FDPIC, R_ARM_TLS_LDM32_FDPIC, R_ARM_TLS_IE32_FDPIC` to be relative to GOT, as opposed to their non-FDPIC counterpart relative to PC.

### PLT in FDPIC

The PLT entry needs to update the FDPIC register as well as changing the program counter (PC).
binutils' arm port uses the following code sequence.

```asm
foo@plt:
ldr r12, .L1
add r12, r12, r9
ldr r9, [r12, #4]
ldr pc, [r12]
.L1: .word foo(GOTOFFFUNCDESC)
```

Lazy binding could be implemented, but it is difficult if the architecture does not allow atomic updates of two words.
binutils' arm port just disable lazy binding.

Let's inspect an example involving consecutive function calls.
```c
void f0(void);
void f1(void);
void f2(void);
void g() { f0(); f1(); f2(); }
```

```asm
g:
  push    {r4, lr}
  mov     r4, r9
  bl      f(PLT)
  mov     r9, r4
  bl      f(PLT)
  mov     r9, r4
  pop     {r4, lr}
  b       f(PLT)
```

If GCC implements `-fno-plt`, it can use the following code sequence:
```asm
g:
  push    {r4, lr}
  mov     r4, r9
  // call f0
  ldr     r12, .L0
  add     r12, r12, r4
  ldr     r9, [r12, #4]
  ldr     pc, [r12]
  // call f1
  ldr     r12, .L1
  add     r12, r12, r4
  ldr     r9, [r12, #4]
  ldr     pc, [r12]
  // tail call f2
  ldr     r12, .L2
  add     r12, r12, r4
  ldr     r9, [r12, #4]
  pop     {r4, lr}
  ldr     pc, [r12]

.L0: .word f0(GOTOFFFUNCDESC)
.L1: .word f1(GOTOFFFUNCDESC)
.L2: .word f2(GOTOFFFUNCDESC)
```

### Relative relocations and `.rofixup` section

Unlike standard `R_*_RELATIVE` relocations that use "*loc += load_base" semantics, the load address in FDPIC mode is dependent on the containing segment.
The following code adapted fro musl demonstrates the behavior:
```c
static void *laddr(const struct dso *p, size_t v) {
  size_t j=0;
  for (; v-p->loadmap->segs[j].p_vaddr >= p->loadmap->segs[j].p_memsz; j++);
  return (void *)(v - p->loadmap->segs[j].p_vaddr + p->loadmap->segs[j].addr);
}
```

In `-pie` and `-shared` links, a dynamic section is present, and non-preemptible function and data pointers are relocated by `R_*_FUNCDESC_VALUE` and `R_*_RELATIVE` dynamic relocations.
For `-no-pie` links, the situation varies:

* Dynamic links: A dynamic section is present. We can still use dynamic relocations.
* Static links: There is no dynamic section. In the non-FDPIC, there is even no relocation (other than `R_*_IRELATIVE`, unsupported in musl/uclibc-ng).

FDPIC executables of type `ET_EXEC` present a unique challenge: while the text segment has a fixed address, the data segment has an unknown address at link time and require relocations.
To address this, a linker-created section named `.rofixup` was introduced in the first FDPIC ABI (FR-V), and later adopted by other FDPIC ABIs.

`.rofixup` holds non-preemptible function and data pointers, which have `R_*_RELATIVE` semantics.
The last entry of `.rofixup` is special and holds the address of `_GLOBAL_OFFSET_TABLE_`.
In a `-pie` or `-shared` link,  `.rofixup` has only one entry.
`__ROFIXUP_LIST__` and `__ROFIXUP_END__` are defined as encapsulation symbols of `.rofixup`.

At run time, the loader sets the FDPIC register to the relocated `_GLOBAL_OFFSET_TABLE_` value before traferring control to the entry point of the executable.

Here is an example:
```asm
.globl fun; fun: bx lr
.section .rodata,"a"
.globl var; var: .long 0

.section .data.rel.ro,"aw"
.long fun(FUNCDESC)          // R_ARM_FUNCDESC_VALUE or two .rofixup entries
.long var                    // R_ARM_RELATIVE or one .rofixup entry
```

## Reflections on FDPIC

FDPIC can be seen as:

* An extended `-msep-data`/`-mno-pic-data-is-text-relative` mode that utilizes function descriptors to support PIC register changes for dynamic linking.
* A fixed [PPC64 ELFv1 function descriptors ABI](/blog/2023-02-26-linker-notes-on-power-isa#ppc64-elfv1-function-descriptors). However, PPC64 ELFv1's trick of `st_value` referring to the function descriptor is better than the existing FDPIC ABIs (sh, arm).

FDPIC resembles PPC64 ELFv2 TOC where the FDPIC register is set by the caller instead of the callee, avoiding global/local entry and tail call complexity.

* `-fno-pic -mfdpic` with hidden visibility declarations can replace `-fno-pic -fropi -frwpi`, though clobbered r9 across function calls has slight overhead.
* `-fPIE -mfdpic` with hidden visibility declarations can replace `-fPIE -msep-data`, though setting the call-clobbered FDPIC register has slight overhead.

`-mfdpic` often generates smaller code than `-mno-fdpic` on architectures where PC-relative addressing is expensive. This includes:

* sh4: Lacks PC-relative addressing entirely.
* arm: Needs `LDR` with `.word _GLOBAL_OFFSET_TABLE_-(.LPIC0+4)`, which is expensive.

Since FDPIC works effectively even on systems with MMUs, it raises the intriguing possibility of replacing the standard calling ABI entirely.

## Toolchain notes

`-mfdpic` enables FDPIC code generation.
GCC'sh port got [FDPIC support](https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;h=1e44e857e05c165f6f01aeb56a7a43ee765bfc99) in 2015.
`-mfpic` implies `-fPIE`, so `-fno-pic -mfdpic` and `-fPIE -mfdpic` have the same codegen behavior.
`-fPIC -mfdpic` may have different generated code as it additionally sets `flag_shlib`.

The cfgexpand pass calls `sh_get_fdpic_reg_initial_val` to retrieve the FDPIC register value from a pseudo register, and register the pseudo register for the first invocation.
At the start of the ira (Integrated Register Allocator) pass, `allocate_initial_values` initializes the pseudo register to the hard register r12 at the function entry point.
sh is the only port that defines `TARGET_ALLOCATE_INITIAL_VALUE`.

In GCC's arm port, `-fno-pic -mfdpic` generated code does not work.

In addition, external function calls save and restore r9.

gas's arm port needs `--fdpic` to assemble FDPIC-related relocation types.
GCC configured with a `arm*-*-uclinuxfdpiceabi` target utilizes `arm/uclinuxfdpiceabi.h` and transforms `-mfdpic` to `--fdpic` when assembling a file.
For other targets, `-Wa,--fdpic` is needed to assemble the output. [[PATCH] arm: Support -mfdpic for more targets](https://gcc.gnu.org/pipermail/gcc-patches/2024-February/646436.html) will make `-Wa,--fdpic` unneeded.

`-mfdpic -mtls-dialect=gnu2` is not supported.
The ARM FDPIC ABI uses `ldr` to load a 32-bit constant embedded in the text segment.
The offset is used to materialize the address of a GOT entry (canonical function descriptor, address of the canonical function descriptor, or address of data).

You can configure binutils with `--target=arm-unknown-uclinuxfdpiceabi` to get a BFD linker that supports FDPIC emulations.
```
% ~/Dev/binutils-gdb/out/arm-fdpic/ld/ld-new -V
GNU ld (GNU Binutils) 2.42.50.20240222
  Supported emulations:
   armelf_linux_eabi
   armelfb_linux_eabi
   armelf_linux_fdpiceabi
   armelfb_linux_fdpiceabi
% ~/Dev/binutils-gdb/out/arm-fdpic/ld/ld-new -m armelf_linux_fdpiceabi -shared a.o -o a.so
```

GNU ld' arm port fails on `R_ARM_GOTOFFFUNCDESC` referencing a hidden function symbol ([PR31408](https://sourceware.org/bugzilla/show_bug.cgi?id=31408)).
```
% cat a.c
__attribute__((visibility("hidden"))) void fun_hidden();
void *fun_hidden_addr() { return fun_hidden; }
% ./bin/ld-new -m armelf_linux_fdpiceabi a.o
[1]    3819239 segmentation fault  ./bin/ld-new a.o
% ./bin/ld-new -m armelf_linux_fdpiceabi -shared a.o
./bin/ld-new: BFD (GNU Binutils) 2.42.50.20240224 internal error, aborting at ../../../bfd/elf32-arm.c:16466 in allocate_dynrelocs_for_symbol

./bin/ld-new: Please report this bug.
```

In `-no-pie` mode, certain non-function references that require a `.rofixup` entrie leads to a segfault ([PR31407](https://sourceware.org/bugzilla/show_bug.cgi?id=31407)).

Global/weak non-hidden symbols referenced by `R_ARM_FUNCDESC` are unnecessarily exported ([PR31409](https://sourceware.org/bugzilla/show_bug.cgi?id=31407)).

## RISC-V FDPIC

Several proposals exist for defining FDPIC-like ABIs to work for MMU-less systems.

* [[RFC] RISC-V ELF FDPIC psABI addendum](https://groups.google.com/a/groups.riscv.org/g/sw-dev/c/ZjYUJswknQ4/m/WYRRylTwAAAJ) and [RISC-V FDPIC/NOMMU toolchain/runtime support](https://lca2020.linux.org.au/schedule/presentation/203/): This offers a starting point, but needs further discussion.
* [lowRISC ePIC](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/343): While simple and interesting, ePIC lacks dynamic linking support.
* Stefan O'Rear's proposal: This proposal holds promise and deserves close attention.

Undoubtly, GP should be used as the FDPIC register.

Loading a constant near code (like ARM) is not efficient. Instead, consider a two-instruction sequence:

* Use hi20 and lo12 instructions to generate an offset relative to the GP register.
* Use `c.add a0, gp` to compute the address of the GOT entry.

Maciej's code sequence supports both function and data access through indirect GP-relative addressing.
We can easily enhance it by adding `R_RISCV_RELAX` to enable linker relaxation and improve performance.
Additionally, for consistency with similar notations on x86-64 and AArch64 ("gotpcrel"), let's adopt "gotgprel" notation.

```asm
.L0:
lui rX, %gotgprel_hi20(sym)  # R_RISCV_?(sym); R_RISCV_RELAX
c.add rX, gp                 # R_RISCV_?(.L0)
ld rY, %gotgprel_lo12(sym)   # R_RISCV_?(.L0); rY = address
```

For data access, the code sequence is followed by instructions like:
```asm
lb a0,0(rY)
sb a1,0(rY)
```

Function descriptors and data have different semantics, requiring two relocation types. Stefan O'Rear proposes:

* `R_RISCV_FUNCDESC_GOTGPREL_HI`: Find or create two GOT entries for the canonical function descriptor.
* `R_RISCV_GOTGPREL_HI`: For or create a GOT for the symbol, and return an offset from the FDPIC register.

Drawing inspiration from ARM FDPIC, two additional relocation types are needed for TLS. This results in a 4-type scheme.

### RISC-V FDPIC: optimization

Addressing performance concerns is crucial.
Stefan suggests an "indirect-to-relative optimization and relaxation scheme":

* `R_RISCV_PIC_ADD`: Tags `c.add rX, gp` to enable optimization
* `R_RISCV_INTERMEDIATE_LOAD`: Tags `ld rY, <gotgprel_lo12>(rX)` to enable optimization

Indirect GP-relative addressing can be optimized to direct GP-relative addressing under specific conditions:

* Non-preemptible functions
* Non-preemptible data in the data segment

```asm
# Indirect GP-relative to direct GP-relative
lui rX, <gprel_hi20>
c.add rX, gp
addi rY, rX, <gprel_lo12>
```

GOT-indirect addressing can be optimized to PC-relative for non-preemptible data in the text segment.

```asm
# Indirect GP-relative to PC-relative
auipc rX, %pcrel_hi20(sym)
c.nop                               # deletable
addi rY, rX, %pcrel_lo12(sym)
```

GOT-indirect addressing can be optimized to absolute addressing for non-preemptible data in the text segment.

```asm
# Indirect GP-relative to absolute
lui rX, %hi20(sym)
c.nop                               # deletable
addi rY, rX, %lo12(sym)
```

This can be used for `SHN_ABS` and unresolved undefined weak symbols.
With `-no-pie` linking, regular symbols are elligible for this optimization as well.
However, linkers may choose not to implement this since the added complexity might outweigh the benefits.

<!--
Remember, FDPIC aims for position-independent code, and a [static shared library](/blog/2021-09-19-all-about-procedure-linkage-table#appendix) scheme wouldn't be ideal.
Imagine multiple executables sharing the same system: each text segment needs a unique address for proper execution.
-->

### RISC-V FDPIC: thread-local storage

To handle TLSDESC, we introduce a new relocation type: `R_RISCV_TLSDESC_GPREL_HI`.
This type instructs the linker to find or create two GOT entries unless optimized to local-exec or init-exec.
The combined hi20 and lo12 offsets compute the GP-relative offset to the first GOT entry.

```asm
label:
lui rX, %tlsdesc_gprel_hi(sym)      # R_RISCV_TLSDESC_GPREL_HI(sym); R_RISCV_RELAX
c.add a0, gp                        # R_RISCV_PIC_ADD(label)
ld rY, rX, %tlsdesc_load_lo(label)  # R_RISCV_TLSDESC_LOAD_LO12(label)
addi a0, rX, %tlsdesc_add_lo(label) # R_RISCV_TLSDESC_ADD_LO12(label)
jalr t0, rY, %tlsdesc_call(label)   # R_RISCV_TLSDESC_CALL(label)
```

Existing relocation types, `R_RISCV_TLSDESC_LOAD_LO12` and `R_RISCV_TLSDESC_ADD_LO12`, are extended to work with `R_RISCV_TLSDESC_GPREL_HI`.

```asm
# TLSDESC to initial-exec optimization
lui a0, <gottpoff_gprel_hi20>
c.add a0, gp
ld a0, <gottpoff_gprel_lo12>(a0)

# TLSDESC to local-exec optimization
lui a0, <tpoff_hi20>
addi a0, a0, <tpoff_lo12>
```

For initial-exec TLS model, we need a new pseudoinstruction, say, `la.tls.ie.fd rX, sym`. It expands to:

```asm
lui rX, 0                    # R_RISCV_TLS_GOTGPREL_HI20(sym)
c.add rX, gp                 # R_RISCV_PIC_ADD(label)
ld rX, 0(rX)                 # R_RISCV_PIC_LO12_I(label)
```

Stefan's scheme defines `R_RISCV_PIC_LO12_I` as an alias for `R_RISCV_PCREL_LO12_I`.
Since the symbol is GP-relative instead of PC-relative, avoiding `PCREL` in the relocation type name makes sense.

Stefan's 11-type scheme adds `R_RISCV_PIC_ADDR_LO12_I` to be associated with `ld rX, 0(rX)` instead.
I have not yet figured out the reasoning.

### RISC-V FDPIC: `-fno-plt`

Regular [`-fno-plt`](/blog/2021-09-19-all-about-procedure-linkage-table#fno-plt) code loads the `.got.plt` entry using PC-relative addressing and performs an indirect branch.
The FDPIC `-fno-plt` variant needs to load both the FDPIC register and the destination address.

```asm
lui rX, 0
c.add rX, gp
ld gp, 8(rX)
ld rX, 0(rX)
c.jr rX
```

## libc implementations with FDPIC support

* uclibc-ng supports AArch32, Blackfin, and FR-V.
* musl supports SuperH.
