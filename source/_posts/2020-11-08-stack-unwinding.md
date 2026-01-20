---
layout: post
title: Stack unwinding
author: MaskRay
tags: [gcc,llvm]
---

Update in 2025-11.

The main usage of stack unwinding is:

* To obtain a stack trace for debuggers, crash reporters, profilers, garbage collectors, etc.
* To implement C++ exceptions (Itanium C++ ABI) using personality routines and language-specific data area. See [C++ exception handling ABI](/blog/2020-12-12-c++-exception-handling-abi)

Some people use "stack walking" to refer to obtaining a stack trace, while "stack unwinding" refers to an enhanced operation that also recovers callee-saved registers.
In this post, we don't make the distinction.

<!-- more -->

Stack unwinding tasks can be divided into two categories:

- synchronous: triggered by the program itself, C++ throw, get its own stack trace, etc. This type of stack unwinding only occurs at the function call (in the function body, it will not appear in the prologue/epilogue)
- asynchronous: triggered by a garbage collector, signals or an external program, this kind of stack unwinding can happen in function prologue/epilogue

Note, GCC supports `-fnon-call-exceptions`, but its behavior is unclear. ([PR70387](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=70387))
The feature seems used heavily by the defunct GNU Compiler for Java.

```
% cat a.cc
#include <stdio.h>
void nop() {}
int main() {
  int i = 0;
  int* volatile p = &i;
  try {
    nop();
    printf("%d\n", 1 / *p);
    nop();
  } catch (...) { puts("oops"); }
}
% g++ a.cc -o a -fnon-call-exceptions -fasynchronous-unwind-tables
% ./a
[1]    16959 floating point exception (core dumped)  ./a
```

## Mechanisms

- Frame pointers
- DWARF `.eh_frame`
- Other unwind formats
- Hardware-assisted stack walking features

## Frame pointer

The most classic and simplest stack unwinding is based on the frame pointer: fix a register as the frame pointer (RBP on x86-64), put the frame pointer in the stack frame at the function prologue, and update the frame pointer to the address of the saved frame pointer.
The frame pointer and its saved values in the stack form a singly linked list.

```asm
pushq %rbp
movq %rsp, %rbp # after this, RBP references the current frame
...
popq %rbp
retq  # RBP references the previous frame
```

After obtaining the initial frame pointer value (`__builtin_frame_address`), dereference the frame pointer continuously to get the frame pointer values of all stack frames.
This method is not applicable to some instructions in the prologue/epilogue.

Note: on RISC-V and LoongArch, the stack slot for the previous frame pointer is stored at `fp[-2]` instead of `fp[0]`.
See [Consider standardising which stack slot fp points to](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/18) for the RISC-V discussion.

The following code works on many architectures. You can enable `-rdynamic` (`g++ a.cc -rdynamic`) to symbolize the global function names.
```c
#include <dlfcn.h>
#include <stdio.h>
[[gnu::noinline]] void qux() {
  void **fp = (void **)__builtin_frame_address(0);
  for (;;) {
#if defined(__riscv) || defined(__loongarch__)
    void **next_fp = fp[-2], *pc = fp[-1];
#elif defined(__powerpc__)
    void **next_fp = fp[0];
    void *pc = next_fp <= fp ? 0 : next_fp[2];
#else
    void **next_fp = (void **)*fp, *pc = fp[1];
#endif
    printf("%p %p", next_fp, pc);
    Dl_info info;
    if (dladdr(pc, &info) && info.dli_sname)
      printf(" %s + 0x%tx", info.dli_sname, (char *)pc - (char *)info.dli_saddr);
    puts("");
    if (next_fp <= fp)
      break;
    fp = next_fp;
  }
}
[[gnu::noinline]] void bar() { qux(); }
[[gnu::noinline]] void foo() { bar(); }
int main() { foo(); }
```

The frame pointer-based method is simple, but has several drawbacks.

When the above code is compiled with `-O1` or above and `[[gnu::noinline]]` attributes are removed, `foo` and `bar` will have tail calls, and the program output will not include the stack frame for `foo` and `bar`.
`-fno-omit-frame-pointer` does not suppress the tail call optimization.

The compiler default of `-fomit-frame-pointer` has played an important role. Many targets default to `-fomit-frame-pointer` with `-O1` or above.
Therefore, in practice, it is not guaranteed that all libraries contain frame pointers.
When unwinding a thread, it is necessary to check whether `next_fp` is like a stack address before dereferencing it to prevent segfaults.

If we can inject code to the target thread, `pthread_attr_getstack` gets the stack bounds and is an efficient way to check page accessibility.

```c
pthread_attr_t attr;
void *addr = 0;
size_t size = 0;
pthread_getattr_np(pthread_self(), &attr);
pthread_attr_getstack(&attr, &addr, &size);
pthread_attr_destroy(&attr);
```

Another way is to parse `/proc/*/maps` to determine whether the address is readable (slow). There is a smart trick:
```c
#include <fcntl.h>
#include <unistd.h>

// Or use the write end of a pipe.
int fd = open("/dev/random", O_WRONLY);
assert(fd >= 0);
if (write(fd, address, 1) < 0)
  // not readable
close(fd);
```

On Linux, `rt_sigprocmask` is better:
```c
#include <errno.h>
#include <fcntl.h>
#include <syscall.h>
#include <unistd.h>

errno = 0;
syscall(SYS_rt_sigprocmask, ~0, address, (void *)0, /*sizeof(kernel_sigset_t)=*/8);
if (errno == EFAULT)
  // not readable
```


In addition, reserving a register for the frame pointer will increase text size and have negative performance impact (prologue, epilogue additional instruction overhead and register pressure caused by one fewer register), which may be quite significant on x86-32 which lack registers.
On an architecture with relatively sufficient registers, e.g. x86-64, the performance loss can be very small, say, 1%.

### Compiler behavior

- Don't maintain FP chain: `-fomit-frame-pointer -momit-leaf-frame-pointer` (smallest overhead)
  + On some targets FP is not reserved, i.e. FP may be allocated.
  + On other targets FP is reserved, e.g. Windows AArch64 since Clang 21, AArch32 with `-mframe-chain=aapcs` and `-mframe-chain=aapcs+laf`
- Maintain FP for non-leaf functions, reserve but don't push/pop FP for leaf functions: `-fno-omit-frame-pointer -momit-leaf-frame-pointer`
- Maintain FP for all functions: `-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer` (largest overhead)

In many targets supported by Clang, `-O` implies `-fomit-frame-pointer`.

For leaf functions (those that don't call other functions), while the frame pointer register should still be reserved for consistency, the push/pop operations are often unnecessary.
Compilers provide `-momit-leaf-frame-pointer` (with target-specific defaults) to reduce code size.

The viability of this optimization depends on the target architecture:

- On AArch64, the return address is available in the link register (X30). The immediate caller can be retrieved by inspecting X30, so `-momit-leaf-frame-pointer` does not compromise unwinding.
- On x86-64, after the prologue instructions execute, the return address is stored at RSP plus an offset. An unwinder needs to know the stack frame size to retrieve the return address, or it must utilize DWARF information for the leaf frame and then switch to the FP chain for parent frames.

Beyond this architectural consideration, there are additional practical reasons to use `-momit-leaf-frame-pointer` on x86-64:

- Many hand-written assembly implementations (including numerous glibc functions) don't establish frame pointers, creating gaps in the frame pointer chain anyway.
- In the prologue sequence `push rbp; mov rbp, rsp`, after the first instruction executes, RBP does not yet reference the current stack frame. When shrink-wrapping optimizations are enabled, the instruction region where RBP still holds the old value becomes larger, increasing the window where the frame pointer is unreliable.

Given these trade-offs, three common configurations have emerged:


GCC 8 is known to omit frame pointers if the function does not need a frame record for x86 ([i386: Don't use frame pointer without stack access](https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;h=8e941ae950ddce1745b4d6819a7131908dd7de24)).
There is a feature request: [Option to force frame pointer](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98018).

With GCC's shrink wrapping optimization (which can be disabled with `-fno-shrink-wrap`), `-fno-omit-frame-pointer` does not ensure there is `push rbp; mov rbp, rsp` at the function beginning.

GCC's `-fschedule-insns2` optimization may insert unrelated instructions between `push rbp` and `mov rbp, rsp`. (<https://gcc.gnu.org/PR55667>)
```cpp
// GCC reschedule instructions for both aarch64 and x86-64
int f(int &, float);
int g(int i, float t) {
  return f(i, t) + f(i, t);
}
```

GCC x86 has [missing optimization with `-fno-omit-frame-pointer`](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=108386).

GCC's s390x port may save link register and FP to floating-point registers in leaf functions.

## libunwind

C++ exception and stack unwinding of profiler/crash reporter usually use libunwind API and DWARF Call Frame Information. In the 1990s, Hewlett-Packard defined a set of libunwind API, which is divided into two categories:

* `unw_*`: The entry points are `unw_init_local` (local unwinding, current process) and `unw_init_remote` (remote unwinding, other processes). Applications that usually use libunwind use this API. For example, Linux perf will call `unw_init_remote`
* `_Unwind_*`: This part is standardized as Level 1: Base ABI of [Itanium C++ ABI: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html). The Level 2 C++ ABI calls these `_Unwind_*` APIs. Among them, `_Unwind_Resume` is the only API that is directly called by C++ compiled code. `_Unwind_Backtrace` is used by a few applications to obtain stack traces. Other functions are called by libsupc++/libc++abi `__cxa_*` functions and `__gxx_personality_v0`.

Hewlett-Packard has open sourced <https://www.nongnu.org/libunwind/> (in addition to many projects called "libunwind"). The common implementations of this API on Linux are:

* libgcc/unwind-* (`libgcc_s.so.1` or `libgcc_eh.a`): Implemented `_Unwind_*` and introduced some extensions: `_Unwind_Resume_or_Rethrow, _Unwind_FindEnclosingFunction, __register_frame` etc.
* llvm-project/libunwind (`libunwind.so` or `libunwind.a`) is a simplified implementation of HP API, which provides part of `unw_*`, but does not implement `unw_init_remote`. Part of the code is taken from ld64. If you use Clang, you can use `--rtlib=compiler-rt --unwindlib=libunwind` to choose
* glibc's internal implementation of `_Unwind_Find_FDE`, usually not exported, and related to `__register_frame_info`

## DWARF Call Frame Information

The unwind instructions required by different areas of the program are described by DWARF Call Frame Information (CFI).
On the ELF platform, a modified form called `.eh_frame` is used.
See [Linux Standard Base Core Specification, Generic Part](https://refspecs.linuxfoundation.org/lsb.shtml) for detail.
Compiler/assembler/linker/libunwind provides corresponding support.

`.eh_frame` is composed of Common Information Entry (CIE) and Frame Description Entry (FDE). A CIE has these fields:

* length: The size of the length field plus the value of length must be an integral multiple of the address size.
* CIE\_id: Constant 0. This field is used to distinguish a CIE and a FDE. In a FDE, this field is non-zero, representing CIE\_pointer
* version: Constant 1 (or 3 for GAS's riscv port).
* augmentation: A NUL-terminated string describing the CIE/FDE parameter list.
  + `z`: augmentation\_data\_length and augmentation\_data fields are present and provide arguments to interpret the remaining bytes
  + `P`: retrieve one byte (encoding) and a value (length decided by the encoding) from augmentation\_data to indicate the personality routine pointer
  + `L`: retrieve one byte from augmentation\_data to indicate the encoding of language-specific data area (LSDA) in FDEs. The augmentation data of a FDE stores LSDA
  + `R`: retrieve one byte from augmentation\_data to indicate the encoding of initial\_location and address\_range in FDEs
  + `S`: an associated FDE describes a signal frame (used by `unw_is_signal_frame`)
* code\_alignment\_factor: Assuming that the instruction length is a multiple of 2 or 4 (for RISC), it affects the multiplier of parameters such as `DW_CFA_advance_loc`
* data\_alignment\_factor: The multiplier that affects parameters such as `DW_CFA_offset DW_CFA_val_offset`
* return\_address\_register: This is changed from a byte to a ULEB128 in CIE version 3+.
* augmentation\_data\_length: only present if augmentation contains `z`.
* augmentation\_data: only present if augmentation contains `z`. This field provides arguments describing augmentation. For `P`, the argument specifies the personality (1-byte encoding and the encoded pointer). For `R`, the argument specifies the encoding of FDE initial\_location.
* initial\_instructions: bytecode for unwinding, a common prefix used by all FDEs using this CIE
* padding

In `.debug_frame` version 4 or above, address\_size (4 or 8) and segment\_selector\_size are present. `.eh_frame` of CIE version 1 does not have the two fields.
(GAS since 2019 supports `--gdwarf-cie-version={1,3,4}` to set the CIE version.
The riscv port defaults to version 3 to return\_address\_register to use register number >= 256.
Version 4 additionally adds segment\_selector\_size, which is not useful.)

Each FDE has an associated CIE. FDE has these fields:

* length: The length of FDE itself. If it is 0xffffffff, the next 8 bytes (extended\_length) record the actual length. Unless specially constructed, extended\_length is not used
* CIE\_pointer: Subtract CIE\_pointer from the current position to get the associated CIE
* initial\_location: The address of the first location described by the FDE. The value is encoded with a relocation referencing a section symbol
* address\_range: initial\_location and address\_range describe an address range
* augmentation\_data\_length: Present if the associated CIE contains augmentation contains `z`
* augmentation\_data: Present if the associated CIE contains augmentation contains `z`
  + If the associated CIE augmentation contains `L`, language-specific data area will be recorded here
* instructions: bytecode for unwinding, essentially (address,opcode) pairs
* padding: Enough `DW_CFA_nop` instructions (zeros) to make the size match the `length` field. For 64-bit objects, the last FDE is aligned by 8 bytes while other FDEs are aligned by 4.

A CIE may optionally refer to a personality routine in the text section (`.cfi_personality` directive).
A FDE may optionally refer to its associated LSDA in `.gcc_except_table` (`.cfi_lsda` directive).
The personality routine and LSDA are used in Level 2: C++ ABI of Itanium C++ ABI.

`llvm-dwarfdump --eh-frame` and `objdump -Wf` can dump the section.
`objdump -WF` (short for `--dwarf=frames-interp`) gives a tabular output.

```poke
load "elf.pk";
load "dwarf-frame.pk";

type EhFrameCIE = struct {
  Dwarf_Initial_Length length;
  uint32 cie_id == 0;
  uint8 version;
  string augmentation;
  ULEB128 code_alignment_factor;
  LEB128 data_alignment_factor;
  ULEB128 return_address_register;
  if (strchr(augmentation, 'z') < augmentation'length)
    ULEB128 augmentation_length;
  if (strchr(augmentation, 'z') < augmentation'length)
    uint8[augmentation_length.value] augmentation_data;
  uint8[length.value + length'size - OFFSET] initial_instructions;
};

type EhFrameFDE = struct {
  Dwarf_Initial_Length length;
  uint32 cie_pointer;
  int32 initial_location; // augmentation 'R' decides the type
  int32 address_range; // augmentation 'R' decides the type
  uint8[length.value + length'size - OFFSET] instructions;
};

type EhFrameEntry = union {
  EhFrameCIE cie;
  EhFrameFDE fde;
};
```


### `.eh_frame` vs `.debug_frame`

Some target default to `-fasynchronous-unwind-tables` while some default to `-fno-asynchronous-unwind-tables`.

Here is the GCC and Clang behavior:
```text
Compiler options                                  Produced section
-fasynchronous-unwind-tables -fexceptions        .eh_frame
-fno-asynchronous-unwind-tables -fexceptions     .eh_frame
-fasynchronous-unwind-tables -fno-exceptions     .eh_frame
-fno-asynchronous-unwind-tables -fno-exceptions  none (-g0) or .debug_frame (-g1 and above)
```

`.eh_frame` is based on `.debug_frame` introduced in DWARF v2. They have some differences, though:

* `.eh_frame` has the flag of `SHF_ALLOC` (indicating that a section should be part of the process image) but `.debug_frame` does not, so the latter has very few usage scenarios.
* `.debug_frame` supports DWARF64 format (supports 64-bit offsets but the volume will be slightly larger) but `.eh_frame` does not support (in fact, it can be expanded, but lacks demand)
* In the CIE of `.debug_frame`, augmentation instead of augmentation\_data\_length and augmentation\_data is used.
* DWARFv5 doesn't mention augmentation in the FDE of `.debug_frame`.
* The version field in CIEs is different.
* The meaning of CIE\_pointer in FDEs is different. `.debug_frame` indicates a section offset (absolute) and `.eh_frame` indicates a relative offset. This change made by `.eh_frame` is great. If the length of `.eh_frame` exceeds 32-bit, `.debug_frame` has to be converted to DWARF64 to represent CIE\_pointer, and relative offset does not need to worry about this issue (if the distance between FDE and CIE exceeds 32-bit, add a CIE OK)
* In `.eh_frame`, augmentation always includes `zR` and the FDE encoding is typically `DW_EH_PE_pcrel|DW_EH_PE_sdata4` for small code models of AArch64/PowerPC64/x86-64. initial\_location has 4 bytes in GCC (even if `-mcmodel=large`). In `.debug_frame`, 64-bit architectures need 8-byte initial\_location. Therefore, `.eh_frame` is usually smaller than an equivalent `.debug_frame`

For two otherwise equivalent relocatable object files, one using `.debug_frame` while the other using `.eh_frame`, `size(.debug_frame)+size(.rela.debug_frame)` > `size(.eh_frame)+size(.rela.eh_frame)`, perhaps larger by ~20%.
If we compress `.debug_frame` (`.eh_frame` cannot be compressed), `size(compressed .debug_frame)+size(.rela.debug_frame) < size(.eh_frame)+size(.rela.eh_frame)`.

---

For the following function:
```c
void f() {
  __builtin_unwind_init();
}
```

The compiler produces `.cfi_*` (CFI directives) to annotate the assembly, `.cfi_startproc` and `.cfi_endproc` annotate the FDE area, and other CFI directives describe CFI instructions.
A call frame is indicated by an address on the stack. This address is called Canonical Frame Address (CFA), and is usually the stack pointer value of the call site. The following example demonstrates the usage of CFI instructions:
```asm
f:
# At the function entry, CFA = rsp+8
	.cfi_startproc
# %bb.0:
	pushq	%rbp
# Redefine CFA = rsp+16
	.cfi_def_cfa_offset 16
# rbp is saved at the address CFA-16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
# CFA = rbp+16. CFA does not needed to be redefined when rsp changes
	.cfi_def_cfa_register %rbp
	pushq	%r15
	pushq	%r14
	pushq	%r13
	pushq	%r12
	pushq	%rbx
# rbx is saved at the address CFA-56
	.cfi_offset %rbx, -56
	.cfi_offset %r12, -48
	.cfi_offset %r13, -40
	.cfi_offset %r14, -32
	.cfi_offset %r15, -24
	popq	%rbx
	popq	%r12
	popq	%r13
	popq	%r14
	popq	%r15
	popq	%rbp
# CFA = rsp+8
	.cfi_def_cfa %rsp, 8
	retq
.Lfunc_end0:
	.size	f, .Lfunc_end0-f
	.cfi_endproc
```

The assembler parses CFI directives and generates `.eh_frame` (this mechanism was introduced by Alan Modra in 2003). Linker collects `.eh_frame` input sections in .o/.a files to generate output `.eh_frame`.
[In 2006](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=9b8ae42e78e6c3e3bc67c31673233568c27d9e71), GNU as introduced `.cfi_personality` and `.cfi_lsda`.

### `.eh_frame_hdr` and `PT_GNU_EH_FRAME`

To locate the FDE where a pc is located, you need to scan `.eh_frame` from the beginning to find the appropriate FDE (whether the pc falls in the interval indicated by initial\_location and address\_range). The time spent is proportional to the number of scanned CIE and FDE records.
<https://sourceware.org/pipermail/binutils/2001-December/015674.html> introduced `.eh_frame_hdr`, a binary search index table describing (initial\_location, FDE address) pairs.

```poke
type EhFrameHdr = struct {
  uint8 version;
  uint8 eh_frame_ptr_enc;
  uint8 fde_count_enc;
  uint8 table_enc;
  int32 eh_frame_ptr; // eh_frame_ptr_enc decides the type
  int32 fde_count; // fde_count_enc decides the type

  type Entry = struct {
    int32 initial_location; // table_enc decides the type
    int32 address; // table_enc decides the type
  };
  Entry[fde_count] table;
};
```

The linker collects all `.eh_frame` input sections. With `--eh-frame-hdr`, `ld` generates `.eh_frame_hdr` and creates a program header `PT_GNU_EH_FRAME` encompassing `.eh_frame_hdr`.
An unwinder can parse the program headers and look for `PT_GNU_EH_FRAME` to locate `.eh_frame_hdr`. Please check out the example below.

Clang and GCC usually pass `--eh-frame-hdr` to ld, with the exception that `gcc -static` does not pass `--eh-frame-hdr`.
The difference is a historical choice related to `__register_frame_info`.

GNU ld and ld.lld only support `eh_frame_ptr_enc = DW_EH_PE_pcrel | DW_EH_PE_sdata4;` (PC-relative `int32_t`), `fde_count_enc = DW_EH_PE_udata4;` (`uint32_t`), and `table_enc = DW_EH_PE_datarel | DW_EH_PE_sdata4;` (`.eh_frame_hdr`-relative `int32_t`).
(GNU ld also supports `DW_EH_PE_omit` when there is no FDE.)

(
```poke
load "eh_frame_hdr.pk"
load elf
var efile = Elf64_File @ 0#B
var hdr = efile.get_sections_by_name(".eh_frame_hdr")[0]
printf "%Tv\n", EhFrameHdr @ hdr.sh_offset
```
)

### `__register_frame_info`

Before `.eh_frame_hdr` and `PT_GNU_EH_FRAME` were invented, there was a static constructor `frame_dummy` in crtbegin (`crtstuff.c`): calling `__register_frame_info` to register the executable file `.eh_frame`.

Nowadays `__register_frame_info` is only used by programs linked with `-static`. Correspondingly, if you specify `-Wl,--no-eh-frame-hdr` when linking, you cannot unwind (if you use a C++ exception, the program will call `std::terminate`).

### libunwind example

```c
#include <libunwind.h>
#include <stdio.h>

void backtrace() {
  unw_context_t context;
  unw_cursor_t cursor;
  // Store register values into context.
  unw_getcontext(&context);
  // Locate the PT_GNU_EH_FRAME which contains PC.
  unw_init_local(&cursor, &context);
  size_t rip, rsp;
  do {
    unw_get_reg(&cursor, UNW_X86_64_RIP, &rip);
    unw_get_reg(&cursor, UNW_X86_64_RSP, &rsp);
    printf("rip: %zx rsp: %zx\n", rip, rsp);
  } while (unw_step(&cursor) > 0);
}

void bar() {backtrace();}
void foo() {bar();}
int main() {foo();}
```

If you use llvm-project/libunwind：
```sh
$CC a.c -Ipath/to/include -Lpath/to/lib -lunwind
```

If you use nongnu.org/libunwind, there are two options: (a) Add `#define UNW_LOCAL_ONLY` before `#include <libunwind.h>` (b) Link one more library, on x86-64 it is `-l:libunwind-x86_64.so`.
If you use Clang, you can also use `clang --rtlib=compiler-rt --unwindlib=libunwind -I path/to/include a.c`, in addition to providing `unw_*`, it can ensure that `libgcc_s.so` is not linked

* `unw_getcontext`: Get register value (including PC)
* `unw_init_local`
  + Use `dl_iterate_phdr` to traverse executable files and shared objects, and find the `PT_LOAD` program header that contains the PC
  + Find the `PT_GNU_EH_FRAME`(`.eh_frame_hdr`) of the module where you are, and save it in `cursor`
* `unw_step`
  + Binary search for the `.eh_frame_hdr` item corresponding to the PC, record the FDE found and the CIE it points to
  + Execute initial\_instructions in CIE
  + Execute the instructions (bytecode) in FDE. An automaton maintains the current location and CFA. Among the instructions, `DW_CFA_advance_loc` advances the location; `DW_CFA_def_cfa_*` updates CFA; `DW_CFA_offset` indicates that the value of a register is stored at CFA+offset
  + The automaton stops when the current location is greater than or equal to PC. In other words, the executed instruction is a prefix of FDE instructions

An unwinder locates the applicable FDE according to the program counter, and executes all the CFI instructions before the program counter.

The most common directives are:

* `DW_CFA_def_cfa_*`
* `DW_CFA_offset`
* `DW_CFA_advance_loc`

In a `-DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86` build of clang, `.text` is 51.7MiB, `.eh_frame` is 4.2MiB, `.eh_frame_hdr` is 646B. There are 2 CIE and 82745 FDE.

### Remarks on DWARF CFI

Some conditional execution states are not expressible with CFI instructions.
For instance, in the code between `popne {some registers}` and `bne label` below, the registers may or may not be on the stack, and DWARF CFI cannot represent this scenario.

```asm
// AArch32
cmp   this, that
popne {some registers}
...
bne   label
```

CFI instructions are suitable for the compiler to generate code, but cumbersome to write in hand-written assembly.
In 2015, Alex Dowad contributed an awk script to musl libc to parse the assembly and automatically generate CFI directives.
In fact, generating precise CFI instructions is challenging for compilers as well. For a function that does not use a frame pointer, adjusting SP requires outputting a CFI directive to redefine CFA.
GCC does not parse inline assembly, so adjusting SP in inline assembly often results in imprecise CFI.

```c
void foo() {
  asm("subq $128, %rsp\n"
  // Cannot unwind if -momit-leaf-frame-pointer
      "nop\n"
      "addq $128, %rsp\n");
}

int main() {
  foo();
}
```

In glibc, [x86: Remove arch-specific low level lock implementation](https://sourceware.org/git/?p=glibc.git;a=commit;h=c50e1c263ec15e98da3235e663049156fd1afcfa) removed `sysdeps/unix/sysv/linux/x86_64/lowlevellock.h`.
The file used to do something like
```c
#define lll_lock(futex, private) \
       (void)                                                                      \
         ({ int ignore1, ignore2, ignore3;                                         \
            if (__builtin_constant_p (private) && (private) == LLL_PRIVATE)        \
              __asm __volatile (__lll_lock_asm_start                               \
                                "1:\tlea %2, %%" RDI_LP "\n"                       \
                                "2:\tsub $128, %%" RSP_LP "\n"                     \
                                ".cfi_adjust_cfa_offset 128\n"                     \
                                "3:\tcallq __lll_lock_wait_private\n"              \
                                "4:\tadd $128, %%" RSP_LP "\n"                     \
                                ".cfi_adjust_cfa_offset -128\n"                    \
                                "24:"                                              \
                                : "=S" (ignore1), "=&D" (ignore2), "=m" (futex),   \
                                  "=a" (ignore3)                                   \
                                : "0" (1), "m" (futex), "3" (0)                    \
                                : "cx", "r11", "cc", "memory");                    \
...
```

`.cfi_adjust_cfa_offset 128` works with frames using RSP as the CFA but not with RBP.
Unfortunately it is difficult to ensure a frame does not use RBP. Even with `-fomit-frame-pointer`, some conditions will switch to RBP.

The CFIInstrInserter pass in LLVM can insert `.cfi_def_cfa_* .cfi_offset .cfi_restore` to adjust the CFA and callee-saved registers.
[The CFIFixup pass](https://reviews.llvm.org/D114545) in LLVM can insert `.cfi_restore_state .cfi_remember_state`. CFIFixup generated information is more space-efficient and is therefore preferred.

The DWARF scheme also has very low information density. The various compact unwind schemes have made improvement on this aspect. To list a few issues:

* CIE address\_size: nobody uses different values for an architecture. Even if they do (ILP32 ABIs in AArch64 and x86-64), the information is already available elsewhere.
* CIE segment\_selector\_size: It is nice that they cared x86, but x86 itself does not need it anymore:/
* CIE code\_alignment\_factor and data\_alignment\_factor: A RISC architecture with such preference can hard code the values.
* CIE return\_address\_register: I do not know when an architecture wants to use a different register for the return address.
* length: The DWARF's 8-byte form is definitely overengineered... For standard form prologue/epilogue, the field should not be needed.
* initial\_location and address\_range: if a binary search index table is always needed, why do we need the length field?
* instructions: bytecode is flexible but commonly a function prologue/epilogue is of a standard form and the few callee-saved registers can be encoded in a more compact way.
* augmentation\_data: While this provide flexibility, in practice very rarely a function needs anything more than a personality and a LSDA pointer.

Callee-saved registers other than FP are oftentimes unneeded but there is no compiler option to drop them.

binutils implemented [`as --scfi=experimental`](https://sourceware.org/binutils/wiki/gas/SCFI) in 2024 to generate CFI from assembly files for AArch64 and x86.

#### `SHT_X86_64_UNWIND`

`.eh_frame` has special processing in linker/dynamic loader, so conventionally it should use a separate section type, but `SHT_PROGBITS` was used in the design.
In the x86-64 psABI, the type of `.eh_frame` is `SHT_X86_64_UNWIND` (influenced by Solaris).

* In GNU as, `.section .eh_frame,"a",@unwind` will generate `SHT_X86_64_UNWIND`, and `.cfi_*` will generate `SHT_PROGBITS`.
* Since Clang 3.8, `.cfi_*` generates `SHT_X86_64_UNWIND`

`.section .eh_frame,"a",@unwind` is rare (glibc's x86 port, libffi, LuaJIT and other packages), so checking the type of `.eh_frame` is a good way to distinguish Clang/GCC object file :)
For ld.lld 11.0.0, I contributed <https://reviews.llvm.org/D85785> to allow mixed types for `.eh_frame` in a relocatable link;-)

Suggestion to future architectures: When defining processor-specific section types, please do not use 0x70000001 (`SHT_ARM_EXIDX=SHT_IA_64_UNWIND=SHT_PARISC_UNWIND=SHT_X86_64_UNWIND=SHT_LOPROC+1`) for purposes other than unwinding :)
`SHT_CSKY_ATTRIBUTES=0x70000001`:)

#### Linker perspective

Usually in the case of COMDAT group and `-ffunction-sections`, `.data`/`.rodata` needs to be split like `.text`, but `.eh_frame` is monolithic.
Like many other metadata sections, the main problem with the monolithic section is that garbage collection is challenging in the linker.
Unlike some other metadata sections, simply abandoning garbage collecting is not a choice:

* `.eh_frame_hdr` is a binary search index table and duplicate/unused entries can confuse the customers.
* `.eh_frame` is not small. Users want to discard unused FDE.

ld.lld has some special handling for `.eh_frame`:

- In non-relocatable links, `.eh_frame` input sections are represented as `EhInputSection`, different from regular `InputSection`. This is to enable sub-section processing, specifically, CIEs are de-duplicated and FDEs are garbage collected.
- `splitSections`, before `markLive`, splits `.eh_frame` into pieces in preparation for garbage collection.
- `markLive` needs to retain `.eh_frame` pieces.
- `combineEhSections` combines live pieces.
- During relocation scanning, a relocation from `.eh_frame` to a `STT_SECTION` symbol in a discarded section (due to COMDAT group rule) is permitted. Normally such a `STB_LOCAL` relocation from outside of the group is disallowed.
- Don't produce dynamic relocations in `.eh_frame` even if `-z notext` is specified. (<https://reviews.llvm.org/D143136>)
- `-M` requires special code.

When a linker processes `.eh_frame`, it needs to conceptually split `.eh_frame` into CIE/FDE.
ld.lld splits `.eh_frame` before marking sections as live for `--gc-sections`. ld.lld handles CIE and FDE differently:

* relocations in CIEs reference personality routine symbols. The personality routines should be marked. The idea is that personality routines are generally not referenced by other means, and would be discarded if we don't retain them when scanning CIE relocations.
* relocations in FDEs may reference code functions (via initial\_location) and LSDA (via augmentation\_data). Code functions (identified by the `SHF_EXECINSTR` flag) are not marked. LSDA neither in a section group nor having the `SHF_LINK_ORDER` flag is marked.

ld.lld merges identical CIEs.

GNU ld and gold support `--ld-generated-unwind-info` which can synthesize CFI for PLT entries.
This increases CFI coverage, but I think it is largely obsoleted and irrelevant nowadays.
See `--no-ld-generated-unwind-info` in [Explain GNU style linker options#no-ld-generated-unwind-info](/blog/2020-11-15-explain-gnu-linker-options).
Note, such a mechanism is not available for range extension thunks (also linker synthesized code).

For `--icf`, two text sections may may have identical content and relocations but different LSDA, e.g. the two functions may have catch blocks of different types.
We cannot merge the two sections. For simplicity, we can [mark all text section referenced by LSDA](https://reviews.llvm.org/D84610) as not eligible for ICF.

## Compact unwind information

Apple designed the compact unwind format for synchronous unwinding, which has been in production use since 2009 with Mac OS X 10.6.
While there is no formal documentation, you may check out [The Apple Compact Unwinding Format: Documented and Explained](https://faultlore.com/blah/compact-unwinding/).

llvm-project source: `llvm/lib/MC`, `lld/MachO/UnwindInfoSection.cpp`, `libunwind/src/CompactUnwinder.hpp`, `lldb/source/Symbol/CompactUnwindInfo.cpp`

**Compiler output.** LLVM generates "compact unwind group description" records in the `__LD,__compact_unwind` section.
On arm64 platforms, `__eh_frame` can be omitted from compiler output. ([D12258 and OmitDwarfIfHaveCompactUnwind](https://reviews.llvm.org/D122258))

```asm
// Compiler output: a single unwind group descriptor in __LD,__compact_unwind
.quad _foo
.set L1, Lfoo_end-_foo
.long L1
.long compact_unwind_description
.quad personality
.quad lsda_address
```

**Linker processing.** The "compact unwind group description" records are not compact—the 8-byte fields can be compressed further.
The linker merges group descriptors from input files and builds `__TEXT,__unwind_info` (compact unwind information).
In a typical executable, the majority of unwind information resides in `__unwind_info`, while `__eh_frame` is extremely small, if present at all.

The compact unwind format is based on several key principles:

- **Standard prologue/epilogue encoding:** Most functions have regular prologues and epilogues. With some code generation restrictions (e.g., register saving order, no unrelated instructions in the prologue/epilogue), they can be encoded with a 32-bit descriptor. Functions not maintaining the frame pointer chain require more rigid prologue and epilogue instructions. Rare cases that cannot be encoded this way fall back to DWARF CFI.
- **Two-level page table structure:** This compresses 32-bit descriptors further and enables efficient lookup.
  + The section header references an arbitrary number of pages. Each page is 4096 bytes and describes several hundred entries.
  + The page header consists of a common first level header and a regular or compressed second level header.
  + First level headers are stored consecutively after personality routines. The first level describes: the first address mapped by this page (enabling smaller relative addresses in entries), the offset to the second level header, and the base offset into the LSDA array. Note: a first level header points to a single second level header.
  + Second level headers are stored within the respective 4096-byte pages. Each header is immediately followed by its entries. (The header field `entryPageOffset` is a constant.)
  + The compressed second level enables local palettes for common descriptors.
- **Compressed page entries:** In a page using the compressed second level header, an entry encodes a 24-bit relative instruction address and an 8-bit descriptor index.
- **Separate LSDA storage:** LSDA entries are stored separately as 8-byte (function_offset, lsda_offset) pairs. This optimizes for the no-LSDA case but is expensive when more than 50% entries need LSDA. [A quick estimate](https://discourse.llvm.org/t/rfc-improving-compact-x86-64-compact-unwind-descriptors/47471/20) shows that the current approach is beneficial.

**Compact unwind descriptor.** This is referred to as "encoding" in `libunwind/include/mach-o/compact_unwind_encoding.h`.
"The Apple Compact Unwinding Format: Documented and Explained" calls it "opcode".

- `unwind_info_section_header::commonEncodingsArraySectionOffset` points to a global palette containing commonly-used compact unwind descriptors.
- Regular pages store compact unwind descriptors directly alongside entry addresses.
- Compressed pages use local palettes to store compact unwind descriptors more efficiently.

**Descriptor encoding.** The compact unwind descriptor in `__TEXT,__unwind_info` is encoded as:
```c
uint32_t mode_specific_encoding : 24; // vary with different modes

uint32_t mode : 4; // UNWIND_X86_64_MODE_MASK == UNWIND_ARM64_MODE_MASK

uint32_t has_lsda : 1;
uint32_t personality_index : 2;
uint32_t is_not_function_start : 1;
```

Personality routines are encoded as a 2-bit index, where index 0 indicates no personality. Only 3 entries are reserved for personality routines, with C++ and ObjC taking 2 of them.
Since 2023, LLVM [uses DWARF fallback by default for non-canonical personality routines](https://github.com/llvm/llvm-project/commit/e60b30d5e3878e7d91f8872ec4c4dca00d4a2dfc).

In `.gcc_except_table`, call site offsets and landing pad offsets are relative to the `.cfi_lsda` directive location, usually the function start address.
When a function has multiple compact unwind descriptors and use a single `.cfi_lsda` directive, and the Itanium C++ ABI Level 1 unwinder finds a descriptor that is not the first, it needs to scan backwards to find the function start address.

### x86-64 modes

Five modes are defined:

* 0: reserved
* 1: FP-based frame: RBP is frame pointer, frame size is variable
* 2: SP-based frame: frame pointer is not used, frame size is fixed during compilation
* 3: large SP-based frame: frame pointer is not used, the frame size is fixed at compile time but the value is large and cannot be represented by mode 2
* 4: DWARF CFI escape

**FP-based frame (`UNWIND_X86_64_MODE_RBP_FRAME`)**

The mode-specific encoding is:
```c
uint32_t regs : 15; // Up to 5 saved registers
uint32_t : 1; // 0
uint32_t frame_offset : 8; // First saved register is at [RBP-8*frame_offset]
```

The callee-saved registers on x86-64 are: RBX, R12, R13, R14, R15, RBP, where RBP does not need to be saved again.
3 bits can encode a register and 15 bits can encode a sequence of 5 registers.

**SP-based frame, immediate (`UNWIND_X86_64_MODE_STACK_IMMD`)**

The mode-specific encoding is:
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t : 3;
uint32_t size : 8;
```

cnt represents the number of saved registers (maximum 6).
reg\_permutation indicates the sequence number of the saved register.
size\*8 represents the stack frame size.

**SP-based frame, indirect (`UNWIND_X86_64_MODE_STACK_IND`)**

The mode-specific encoding is:
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t adj : 3;
uint32_t size_offset : 8;
```

Similar to SP-based frame. In particular: the stack frame size is read from the text section. The RSP adjustment is usually represented by `subq imm, %rsp`, and size\_offset is used to represent the distance from the instruction to the beginning of the function.
The actual stack size also includes adj\*8.

**DWARF CFI escape (`UNWIND_X86_64_MODE_DWARF`)**

If for various reasons, the compact unwind descriptor cannot be expressed, it must fall back to DWARF CFI.

### ARM64 modes

**SP-based frame (`UNWIND_ARM64_FRAMELESS`)**

**DWARF CFI escape (`UNWIND_ARM64_MODE_DWARF`)**

**FP-based frame (`UNWIND_ARM64_FRAME`)**

### Lack of asynchronous stack walking support

In the current LLVM implementation, each function is represented by only a compact unwind descriptor.

For a stack walking request in the prologue of an RSP-based frame, the tracer can use the PC offset and the saved registers to infer the canonical frame address. However, this fails if unrelated instructions appear in the prologue. LLVM ensures that no such unrelated instructions are present.

When shrink wrapping is enabled, the prologue may not be at the beginning of the function. Since the current descriptor does not describe the prologue offset, unwinding cannot be performed correctly in this case.

Similarly, if an asynchronous stack walking request occurs in an epilogue that is not at the end of the function (due to tail duplication optimization), the SP adjustment is not described, leading to incorrect canonical frame address computation. Currently, tail duplication skips CFI instructions (see <https://reviews.llvm.org/D40979>).

FP-based frames have fewer such problems. Apple's arm64 platforms require frame pointers, which inherently avoid many of these issues.

Overall, this limitation appears to receive little attention, likely because profilers only lose a small percentage of profile accuracy.

### Asynchronous unwinding extension

In fact, if you use multiple descriptors to describe each area of a function, you can still unwind accurately.
OpenVMS's x86-64 port, which is ELF-based, also adopted this format as documented in their "VSI OpenVMS Calling Standard" and their 2018 post: [[RFC] Improving compact x86-64 compact unwind descriptors](https://lists.llvm.org/pipermail/llvm-dev/2018-January/120741.htm)
Unfortunately, they don't open source the implementation.

This approach makes reasonable assumptions:

- Any use of preserved registers must be delayed until all preserved registers have been saved.
- In a function with an RBP-based frame, the prologue must have adjacent `push rbp; mov rbp, rsp` instructions, and the epilogue must have adjacent `mov rsp, rbp; pop rbp` instructions.
- For shrink wrap optimization with a personality routine, instructions that might cause exceptions (floating point exceptions and access violations) cannot be moved into the prologue.

It then repurposes the `length` field:

- A single unwind group describes a (prologue_start_offset, prologue_size, epilogue_is_present) tuple.
  + 2025 discussions suggest that `epilogue_is_present` should be replaced with `epilogue_end`, the gap size between the end of the register-store sequence and the next descriptor start.
- The prologue is conceptually split into two parts: the first part extends up to and including the instruction that decreases RSP; the second part extends to a point after the last preserved register is saved but before any preserved register is modified (this location is not unique, providing flexibility).
  + When unwinding in the prologue, the RSP register value can be inferred from the PC and the set of saved registers.
- Since register restoration is idempotent (restoring preserved registers multiple times during unwinding causes no harm), there is no need to describe `pop $reg` sequences. The unwind group just need one bit to describe whether the 1-byte `ret` instruction is present.
- The `length` field in the compact unwind group descriptor is repurposed to describe the prologue's two parts.
- By composing multiple unwind groups, potentially with zero-sized prologues or omitting `ret` instructions in epilogues, it can describe functions with shrink wrapping or tail duplication optimization.
- Null frame group (with no prologue or epilogue) is the default and can be used to describe trampolines and PLT stubs.

The proposal isn't clear how to encode a function in `-ffunction-sections` builds. Using two unwind groups for a single function would be too expensive.

```
f0:
  ...
  ret

  .p2align 4
f1:
  ...
  ret
```

**Next-generation compact unwind information**

aengelke suggests in [a discourse discussion](https://discourse.llvm.org/t/rfc-improving-compact-x86-64-compact-unwind-descriptors/47471/11?u=maskray) enlarging the mode-specific encoding from 24 bits to 32 bits.
Combined with prologue start, epilogue end, personality, and mode fields, this creates a 64-bit compact unwind descriptor.

We can update `.eh_frame_hdr` to use a 12-byte structure to encode both the 32-bit address and the 64-bit compact unwind descriptor.
The compact unwind descriptor supports DWARF CFI escape via a special `mode`, allowing `.eh_frame` FDEs that are replaced with compact unwind descriptors to be eliminated.

An optional page table can remove the 32-bit address limitation and enable deduplication of compact unwind descriptors.

```cpp
struct CompactUnwindDescriptor {
  uint64_t reserved : 11;
  // Offset of prologue start into function; -1 implies no prologue.
  // The linker can fold NULL descriptors into non-(-1) prologue_start.
  uint64_t prologue_start : 7;
  // Number of bytes after the end of the register-restore sequence
  // before the beginning of the next descriptor.
  uint64_t epilogue_end : 6;
  // Index into personality function table. 0 implies no personality.
  uint64_t personality_fn : 4;
  // Descriptor mode. 0 means a null frame. 1 means DWARF CFI escape.
  // Other values are arch-specific.
  uint64_t mode : 4;

  // Architecture and mode-specific encoding.
  uint32_t mode_specific_encoding;
};

// RBP-based frame
  /// Size of the prologue, marks the point where all CSRs are saved.
  uint64_t prologue_size : 8;
  /// Saved registers. Details to be discussed. A simple format would be
  /// one bit in the sequence rbp,r15,r14,r13,r12,rbx, indicating whether
  /// it is saved (that'd require just 6 bits).
  uint64_t saved_regs : 24;

// RSP-based frame
  /// Size of the stack frame * 16 (ABI requires 16B alignment).
  /// There is no need for a large frame mode, this currently covers
  /// 16 MiB, which should be enough. (Compilers can use the RBP mode
  /// or DWARF if they require larger stack frames.)
  uint64_t frame_size : 20;
  /// TBD: Specification of alternative push/pop sequence for APX.
  uint64_t is_apx : 1;
  /// TBD: Specification of instruction sequence for -fstack-clash-protection
  uint64_t with_scp : 1;
  /// TBD: I'm sure I forgot things we might need to handle.
  uint64_t reserved : 2;
  /// One bit in the sequence rbp,r15,r14,r13,r12,rbx, indicating whether
  /// it is saved (that'd require just 6 bits). I.e., the first saved reg
  /// of this list is at [CFA-16], the second at [CFA-24], etc.
  /// Maybe we need more options for IPRA, I'm unsure, so I added two
  /// unused bits for now.
  uint64_t saved_regs : 8;
```

**Baseline**

In Nov 2025, I created a branch that ports compact unwind informtion to ELF as a baseline.
<https://github.com/MaskRay/llvm-project/tree/demo-unwind>

**X86 backend**

- Generalize the Mach-O compact unwind code to work for ELF targets.
- Add option "epilog-cfi" to allow disabling epilog CFI (disabled for
  Darwin; see https://reviews.llvm.org/D42848). The current compact unwind
  code does not handle `popq %rbp; .cfi_def_cfa %rsp, 8; ret`

**Integrated assembler**

- Emit .eh_frame CIEs with augmentation character 'C' for compact unwind
- Replace FDE instructions with 64-bit unwind descriptors (extended from
  Mach-O's 32-bit format to support future asynchronous unwinding)
- The augmentation character 'C' mechanism should be considered
  experimental. As a prior art, MIPS compact exception tables uses a
  different section `.eh_frame_entry` instead.

**Linker (lld)**

- Split FDEs into two groups: descriptor-based (augmentation 'C') and
  instruction-based
- Generate .eh_frame_hdr version 2 with 12-byte table entries when compact
  FDEs are present: (pc_ptr, unwind_descriptor_or_fde_ptr)
- TODO: Place LSDA in a separate storage

**Other tools**

- llvm-readelf --unwind: Dump unwind descriptors in .eh_frame_hdr and
  .eh_frame
- llvm-dwarfdump --eh-frame: Recognize augmentation character 'C' in .eh_frame CIEs

Current `.eh_frame_hdr` format (TODO needing overhaul)

```
uint32_t version; // 2
uint32_t eh_frame_ptr_enc; // DW_EH_PE_pcrel | DW_EH_PE_sdata4
uint32_t fde_count_enc; // DW_EH_PE_udata4
uint32_t table_enc; // DW_EH_PE_datarel | DW_EH_PE_sdata8
uint32_t eh_frame_ptr; // offset to .eh_frame
uint32_t fde_count;

struct { uint32_t pc_ptr; uint64_t unwind_desc_or_fde_ptr; } entries[];
// TODO struct { uint32_t pc_ptr; uint32_t lsda_ptr; } lsda[];
```

**AArch64**

<https://www.codalogic.com/blog/2022/10/20/Aarch64-Stack-Frames-Again> demonstrates different stack frames generated by GCC and Clang using the following example:

```cpp
#include <string>
#include <iostream>

std::string merge( std::string a, std::string b, std::string c ) {
    std::string d = a + b;
    std::string e = a + d + b;
    return d + e;
}
```

**RISC-V**

`-msave-restore` may generate library calls to save and restore non-volatile registers in prologue and epilogue code.
A compact unwind implementation needs to account for it.

The Zcmp extension (primarily targeting embedded class CPUs) provides instruction to save and restore ra and s{0-11}.
However, the register save order is [incompatible with `-fno-omit-frame-pointer`](https://github.com/riscvarchive/riscv-code-size-reduction/issues/194).
Xqccmp is a variant of Zcmp that is compatible with `-fno-omit-frame-pointer`.


GCC and Clang generate multiple `addi sp, sp, imm` instructions to adjust the stack pointer.
This makes it challenging to support asynchronous compact unwind information.
```c
void bar(int *);
void foo() { int x[1101]; bar(x); }
```

## ARM exception handling

Divided into `.ARM.exidx` and `.ARM.extab`

`.ARM.exidx` is a binary search index table, composed of 2-word pairs.
The first word is 31-bit PC-relative offset to the start of the region.
The second word uses the program description more clearly:
```c
if (indexData == EXIDX_CANTUNWIND)
  return false;  // like an absent .eh_frame entry. In the case of C++ exceptions, std::terminate
if (indexData & 0x80000000) {
  extabAddr = &indexData;
  extabData = indexData; // inline
} else {
  extabAddr = &indexData + signExtendPrel31(indexData);
  extabData = read32(&indexData + signExtendPrel31(indexData)); // stored in .ARM.extab
}
```

`tableData & 0x80000000` means a compact model entry, otherwise means a generic model entry.

`.ARM.exidx` is equivalent to enhanced `.eh_frame_hdr`, compact model is equivalent to inlining the personality and lsda in `.eh_frame`. Consider the following three situations:

* If the C++ exception will not be triggered and the function that may trigger the exception will not be called: no personality is needed, only one `EXIDX_CANTUNWIND` entry is needed, no `.ARM.extab`
* If a C++ exception is triggered but no landing pad is required: personality is `__aeabi_unwind_cpp_pr0`, only a compact model entry is needed, no `.ARM.extab`
* If there is a catch: `__gxx_personality_v0` is required, `.ARM.extab` is required

`.ARM.extab` is equivalent to the combined `.eh_frame` and `.gcc_except_table`.

There is no `.ARM.extab` counterpart for AArch64. See <https://github.com/ARM-software/abi-aa/issues/344> for a feature request.

### Generic model

```c
uint32_t personality; // bit 31 is 0
uint32_t : 24;
uint32_t num : 8;
uint32_t opcodes[];   // opcodes, variable length
uint8_t lsda[];       // variable length
```

In construction.

## Windows x86-64 exception handling

<https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170>

`MSVC /d2epilogunwind` enables Windows x64 Unwind V2 information.

See also `llvm/lib/Target/X86/X86WinEHUnwindV2.cpp`
If there is a `.seh_endepilogue`, add a `.seh_unwindv2start` before the first in-epilogue POP, or `seh_endepilogue` if there is no such POP.
This directive emits a `UOP_Epilog` code.

## Windows ARM64 exception handling

Update: [Windows ARM64 Frame Unwind Code Details](https://www.corsix.org/content/windows-arm64-unwind-codes)

See <https://docs.microsoft.com/en-us/cpp/build/arm64-exception-handling>, this is my favorite coding scheme.
Support the unwinding of mid-prolog and mid-epilog. Support function fragments (used to represent unconventional stack frames such as shrink wrapping).

Saved in two sections `.pdata` and `.xdata`.

```c
uint32_t function_start_rva;
uint32_t Flag : 2;
uint32_t Data : 30;
```

For canonical form functions, Packed Unwind Data is used, and no `.xdata` record is required; for descriptors that cannot be represented by Packed Unwind Data, it is stored in `.xdata`.

### Packed Unwind Data

```c
uint32_t FunctionStartRVA;
uint32_t Flag : 2;
uint32_t FunctionLength : 11;
uint32_t RegF : 3;
uint32_t RegI : 4;
uint32_t H : 1;
uint32_t CR : 2;
uint32_t FrameSize : 9;
```

## MIPS compact exception tables

The specification is available at <https://github.com/itanium-cxx-abi/cxx-abi/blob/main/MIPSCompactEH.pdf>.

Binutils [added support](https://sourceware.org/cgit/binutils-gdb/commit/?id=2f0c68f23bb3132cd5ac466ca8775c0d9e4960cd) in 2015, though the [GCC patch](https://inbox.sourceware.org/gcc-patches/FD3DCEAC5B03E9408544A1E416F112420192C8DEFB@NA-MBX-04.mgc.mentorg.com/) remains unmerged.

The following is based on my understanding of this format.

**Compiler output.** The directive `.cfi_sections .eh_frame_entry` instructs the assembler to emit index table entries to the `.eh_frame_entry` section.
Then, `.cfi_fde_data` and `.cfi_inline_lsda` directives can be used.
[`.cfi_fde_data opcode1, ...`](https://sourceware.org/binutils/docs/as/CFI-directives.html) betweens a pair of `.cfi_startproc` and `.cfi_endproc` describes the frame unwind opcodes where each opcode takes one byte.
The frame unwind opcodes describes the semantics of prologue instructions, similar to Windows ARM64 Frame Unwind Codes.

**Assembler processing.** The assembler generates a `.eh_frame_entry.*` section for each section with compact unwind information.
Each `.eh_frame_entry` is a pair of 4 bytes, where the first word is like the first word in a `.eh_frame_hdr` entry.
An `.eh_frame_entry` entry takes one of three forms:

- Inline compact: `(even pc, unwind_data)`. This form can be used when there are at most 3 opcodes (3 bytes) and no personality routine.
- Out-of-line compact: `(odd pc, even unwind_ptr)` where `unwind_ptr` points to unwind data in the `.gnu_extab` section.
- Legacy: `(odd pc, odd legacy_unwind_ptr)` where `legacy_unwind_ptr` points to the legacy `.eh_frame` section.

TODO: Describe `.cfi_inline_lsda`, which appears related to `__gnu_compact_pr[1-3]`.

**Linker processing.** GNU ld concatenates `.eh_frame_entry` and `.eh_frame_entry.*` sections, sorting them by address.
The following internal linker script fragment adds a header before the entries:

```
.eh_frame_hdr   : { *(.eh_frame_hdr) *(.eh_frame_entry .eh_frame_entry.*) }
```

Although the section name remains the traditional `.eh_frame_hdr`, the version is set to 2.
`.eh_frame_hdr` is covered by the traditional `PT_GNU_EH_FRAME` program header.
The linker also defines the symbol `__GNU_EH_FRAME_HDR` to hold the `.eh_frame_hdr` address.

TODO: Describe LSDA representation, which is more compact than the traditional `.gcc_except_table` section.

While the current implementation seems synchronous only, extending it to asynchronous unwinding is natural.
<https://inbox.sourceware.org/gcc-patches/55F0C4D5.6080507@redhat.com/> suggests that opcodes can be introduced to describe `DW_CFA_remember_state`/`DW_CFA_restore_state`.
TODO: This might be similar to `END_C` in Windows ARM64 for zero-length prologs that still have unwind state.

## Linux kernel ORC unwind tables

For x86-64, the Linux kernel uses its own unwind tables: ORC. You can find its documentation on <https://www.kernel.org/doc/html/latest/x86/orc-unwinder.html> and there is an lwn.net introduction [The ORCs are coming](https://lwn.net/Articles/728339/).

objtool decodes instructions and analyzes call frame information.
When `--orc` is specified (e.g. `tools/objcopy/objtool --orc --link vmlinux.o`), objtool generates `.orc_header`, `.orc_unwind`, and `.orc_unwind_ip`.
For an object file assembled from:
```asm
.globl foo
.type foo, @function
foo:
  ret
```

At two addresses the unwind information changes: the start of foo and the end of foo, so 2 ORC entries will be produced.
If the DWARF CFA changes (e.g. due to push/pop) in the middle of the function, there may be more entries.

`.orc_unwind_ip` contains two entries, representing the PC-relative addresses.
```
Relocation section '.rela.orc_unwind_ip' at offset 0x2028 contains 2 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000000  0000000500000002 R_X86_64_PC32          0000000000000000 .text + 0
0000000000000004  0000000500000002 R_X86_64_PC32          0000000000000000 .text + 1
```

`.orc_unwind` contains two entries of type `orc_entry`. The entries encode how IP/SP/BP of the previous frame are stored.
```c
struct orc_entry {
  s16 sp_offset; // sp_offset and sp_reg encode where SP of the previous frame is stored
  s16 bp_offset; // bp_offset and bp_reg encode where BP of the previous frame is stored
  unsigned sp_reg:4;
  unsigned bp_reg:4;
  unsigned type:2; // how IP of the previous frame is stored
  unsigned end:1;
} __attribute__((__packed__));
```

You may find similarities in this scheme and `UNWIND_MODE_BP_FRAME` and `UNWIND_MODE_STACK_IMMD` in Apples's compact unwind descriptors.
The ORC scheme uses 16-bit integers so assumably `UNWIND_MODE_STACK_IND` will not be needed.
During unwinding, most callee-saved registers other than BP are unneeded, so ORC does not bother recording them.

The linker will resolve relocations in `.orc_unwind_ip` and create `__start_orc_unwind_ip/__stop_orc_unwind_ip/__start_orc_unwind/__stop_orc_unwind` delimiter the section contents.
Then, a host utility `scripts/sorttable` sorts the contents of `.orc_unwind_ip` and `.orc_unwind`.
To unwind a stack frame, `unwind_next_frame`

* performs a binary search into the `.orc_unwind_ip` table to figure out the relevant ORC entry
* retrieves the previous SP with the current SP, `orc->sp_reg` and `orc->sp_offset`.
* retrieves the previous IP with `orc->type` and other values.
* retrieves the previous BP with the currrent BP, the previous SP, `orc->bp_reg` and `orc->bp_offset`. `bp->reg` can be `ORC_REG_UNDEFINED/ORC_REG_PREV_SP/ORC_REG_BP`.

## SFrame

SFrame is a stack walking format extended from ORC unwind tables. The proposed scenario is for profilers.
However, it sacrifices functionality (e.g. personality, LSDA, callee-saved registers), unsuitable for C++ exceptions.
In addition, its stack offsets are less compact than `.eh_frame`'s bytecode-style CFI instructions.

Please check out [Remarks on SFrame](/blog/2025-09-28-remarks-on-sframe).

### LLVM

In LLVM, `Function::needsUnwindTableEntry` decides whether CFI instructions should be emitted: `hasUWTable() || !doesNotThrow() || hasPersonalityFn()`

On ELF targets, if a function has `uwtable` or `personality`, or does not have `nounwind` (`needsUnwindTableEntry`), it marks that `.eh_frame` is needed in the module.
Then, a function gets `.eh_frame` if `needsUnwindTableEntry` or `-g[123]` is specified.

To ensure no `.eh_frame`, every function needs `nounwind`.

`uwtable(sync)` and `uwtable(async)` specify the amount of unwind information.
(See [[RFC] Asynchronous unwind tables attribute](https://lists.llvm.org/pipermail/llvm-dev/2021-November/153768.html).

If `.eh_frame` is not produced, but at least one function makes `Function::needsUnwindTableEntry` return true, `.debug_frame` is produced if `llvm::MachineModuleInfo::DbgInfoAvailable` is true or `-fforce-dwarf-frame` is specified.

`lib/CodeGen/AsmPrinter/AsmPrinter.cpp:352`

### Epilogue

It remains an open question how the future stack unwinding strategy should evolve for profiling purposes.
We have at least 3 routes:

* compact unwind scheme.
* hardware assisted. Piggybacking on security hardening features like shadow call stack. However, this unlikely provides more information about callee-saved registers.
* mainly FP-based. People don't use FP due to performance loss. If `-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer` doesn't hurt performance that much, it may be better than some information in `.eh_frame`. We can use unwind information to fill the gap, e.g. for shrink wrapping.

Unwind information is difficult to have 100% coverage.
Linker generated code (PLT and range extension thunks) generally does not have unwind information coverage.
