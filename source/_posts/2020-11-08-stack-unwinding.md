layout: post
title: Stack unwinding
author: MaskRay
tags: [gcc,llvm]
---

Update in 2025-06.

[中文版](#中文版)

The main usage of stack unwinding is:

* To obtain a stack trace for debugger, crash reporter, profiler, garbage collector, etc.
* With personality routines and language specific data area, to implement C++ exceptions (Itanium C++ ABI). See [C++ exception handling ABI](/blog/2020-12-12-c++-exception-handling-abi)

<!-- more -->

Stack unwinding tasks can be divided into two categories:

* synchronous: triggered by the program itself, C++ throw, get its own stack trace, etc. This type of stack unwinding only occurs at the function call (in the function body, it will not appear in the prologue/epilogue)
* asynchronous: triggered by a garbage collector, signals or an external program, this kind of stack unwinding can happen in function prologue/epilogue

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

The following code works on many architectures:
```c
#include <stdio.h>
[[gnu::noinline]] void qux() {
  void **fp = __builtin_frame_address(0);
  for (;;) {
#if defined(__riscv) || defined(__loongarch__)
    void **next_fp = fp[-2], *pc = fp[-1];
#elif defined(__powerpc__)
    void **next_fp = fp[0];
    void *pc = next_fp <= fp ? 0 : next_fp[2];
#else
    void **next_fp = *fp, *pc = fp[1];
#endif
    printf("%p %p\n", next_fp, pc);
    if (next_fp <= fp) break;
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
Therefire, in practice, it is not guaranteed that all libraries contain frame pointers.
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

* `-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer`: all functions maintain frame pointers.
* `-fno-omit-frame-pointer -momit-leaf-frame-pointer`: all non-leaf functions maintain frame pointers. Leaf functions don't maintain frame pointers.
* `-fomit-frame-pointer`: all functions maintain frame pointers. `arm*-apple-darwin` and `thumb*-apple-darwin` don't support the option.

For `-O0`, most targets default to `-fno-omit-frame-pointer`.

For `-O1` and above, many targets default to `-fomit-frame-pointer` (while Apple and FreeBSD don't).
Some targets default to `-momit-leaf-frame-pointer`.
Specify `-fno-omit-leaf-frame-pointer` to get a similar effect to `-O0`.

GCC 8 is known to omit frame pointers if the function does not need a frame record for x86 ([i386: Don't use frame pointer without stack access](https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;h=8e941ae950ddce1745b4d6819a7131908dd7de24)).
There is a feature request: [Option to force frame pointer](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98018).

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
* version: Constant 1
* augmentation: A NUL-terminated string describing the CIE/FDE parameter list.
  + `z`: augmentation\_data\_length and augmentation\_data fields are present and provide arguments to interpret the remaining bytes
  + `P`: retrieve one byte (encoding) and a value (length decided by the encoding) from augmentation\_data to indicate the personality routine pointer
  + `L`: retrieve one byte from augmentation\_data to indicate the encoding of language-specific data area (LSDA) in FDEs. The augmentation data of a FDE stores LSDA
  + `R`: retrieve one byte from augmentation\_data to indicate the encoding of initial\_location and address\_range in FDEs
  + `S`: an associated FDE describes a signal frame (used by `unw_is_signal_frame`)
* code\_alignment\_factor: Assuming that the instruction length is a multiple of 2 or 4 (for RISC), it affects the multiplier of parameters such as `DW_CFA_advance_loc`
* data\_alignment\_factor: The multiplier that affects parameters such as `DW_CFA_offset DW_CFA_val_offset`
* return\_address\_register
* augmentation\_data\_length: only present if augmentation contains `z`.
* augmentation\_data: only present if augmentation contains `z`. This field provides arguments describing augmentation. For `P`, the argument specifies the personality (1-byte encoding and the encoded pointer). For `R`, the argument specifies the encoding of FDE initial\_location.
* initial\_instructions: bytecode for unwinding, a common prefix used by all FDEs using this CIE
* padding

In `.debug_frame` version 4 or above, address\_size (4 or 8) and segment\_selector\_size are present. `.eh_frame` does not have the two fields.

Each FDE has an associated CIE. FDE has these fields:

* length: The length of FDE itself. If it is 0xffffffff, the next 8 bytes (extended\_length) record the actual length. Unless specially constructed, extended\_length is not used
* CIE\_pointer: Subtract CIE\_pointer from the current position to get the associated CIE
* initial\_location: The address of the first location described by the FDE. The value is encoded with a relocation referencing a section symbol
* address\_range: initial\_location and address\_range describe an address range
* instructions: bytecode for unwinding, essentially (address,opcode) pairs
* augmentation\_data\_length
* augmentation\_data: If the associated CIE augmentation contains `L` characters, language-specific data area will be recorded here
* padding

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
* The version field in CIEs is different.
* The meaning of CIE\_pointer in FDEs is different. `.debug_frame` indicates a section offset (absolute) and `.eh_frame` indicates a relative offset. This change made by `.eh_frame` is great. If the length of `.eh_frame` exceeds 32-bit, `.debug_frame` has to be converted to DWARF64 to represent CIE\_pointer, and relative offset does not need to worry about this issue (if the distance between FDE and CIE exceeds 32-bit, add a CIE OK)
* In `.eh_frame`, augmentation typically includes `R` and the FDE encoding is `DW_EH_PE_pcrel|DW_EH_PE_sdata4` for small code models of AArch64/PowerPC64/x86-64. initial\_location has 4 bytes in GCC (even if `-mcmodel=large`). In `.debug_frame`, 64-bit architectures need 8-byte initial\_location. Therefore, `.eh_frame` is usually smaller than an equivalent `.debug_frame`

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

### Remarks

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

* `-M` requires special code
* `--gc-sections`. `--gc-sections` occurs before `.eh_frame` deduplication/GC.
* For `-r` and `--emit-relocs`, a relocation from `.eh_frame` to a `STT_SECTION` symbol in a discarded section (due to COMDAT group rule) should be allowed (normally such a `STB_LOCAL` relocation from outside of the group is disallowed).

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

## Compact unwind descriptors

On macOS, Apple designed the compact unwind descriptors mechanism to accelerate unwinding. In theory, this technique can be used to save some space in `__eh_frame`, but it has not been implemented.
The main idea is:

* The FDE of most functions has a fixed mode (specify CFA at the prologue, store callee-saved registers), and the FDE instructions can be compressed to 32-bit.
* Personality/lsda described by CIE/FDE augmentation data is very common and can be extracted as a fixed field.

Only 64-bit will be discussed below. A descriptor occupies 32 bytes

```asm
.quad _foo
.set L1, Lfoo_end-_foo
.long L1
.long compact_unwind_description
.quad personality
.quad lsda_address
```

If you study `.eh_frame_hdr` (binary search index table) and `.ARM.exidx`, you can know that the length field is redundant.

The Compact unwind descriptor is encoded as:
```c
uint32_t : 24; // vary with different modes
uint32_t mode : 4;
uint32_t flags : 4;
```

Five modes are defined:

* 0: reserved
* 1: FP-based frame: RBP is frame pointer, frame size is variable
* 2: SP-based frame: frame pointer is not used, frame size is fixed during compilation
* 3: large SP-based frame: frame pointer is not used, the frame size is fixed at compile time but the value is large and cannot be represented by mode 2
* 4: DWARF CFI escape

### FP-based frame (`UNWIND_MODE_BP_FRAME`)

The compact unwind encoding is:
```c
uint32_t regs : 15;
uint32_t : 1; // 0
uint32_t stack_adjust : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

The callee-saved registers on x86-64 are: RBX, R12, R13, R14, R15, RBP. 3 bits can encode a register, 15 bits are enough to represent 5 registers except RBP (whether to save and where).
stack\_adjust records the extra stack space outside the save register.

### SP-based frame (`UNWIND_MODE_STACK_IMMD`)

The compact unwind encoding is:
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t : 3;
uint32_t size : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

cnt represents the number of saved registers (maximum 6).
reg\_permutation indicates the sequence number of the saved register.
size\*8 represents the stack frame size.

### Large SP-based frame (`UNWIND_MODE_STACK_IND`)

Compact unwind descriptor編碼爲：
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t adj : 3;
uint32_t size_offset : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

Similar to SP-based frame. In particular: the stack frame size is read from the text section. The RSP adjustment is usually represented by `subq imm, %rsp`, and size\_offset is used to represent the distance from the instruction to the beginning of the function.
The actual stack size also includes adj\*8.

### DWARF CFI escape

If for various reasons, the compact unwind descriptor cannot be expressed, it must fall back to DWARF CFI.


In the LLVM implementation, each function is represented by only a compact unwind descriptor. If asynchronous stack unwinding occurs in epilogue, existing implementations cannot distinguish it from stack unwinding in function body.
Canonical Frame Address will be calculated incorrectly, and the caller-saved register will be read incorrectly.
If it happens in prologue, and the prologue has other instructions outside the push register and `subq imm, $rsp`, an error will occur.
In addition, if shrink wrapping is enabled for a function, prologue may not be at the beginning of the function. The asynchronous stack unwinding from the beginning to the prologue also fails.
It seems that most people don't care about this issue. It may be because the profiler loses a few percentage points of the profile.

In fact, if you use multiple descriptors to describe each area of a function, you can still unwind accurately.
OpenVMS proposed [[RFC] Improving compact x86-64 compact unwind descriptors](http://lists.llvm.org/pipermail/llvm-dev/2018-January/120741.html) in 2018, but unfortunately there is no relevant implementation.

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

### Generic model

```c
uint32_t personality; // bit 31 is 0
uint32_t : 24;
uint32_t num : 8;
uint32_t opcodes[];   // opcodes, variable length
uint8_t lsda[];       // variable length
```

In construction.

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

In construction.

## Linux kernel ORC unwind tables

For x86-64, the Linux kernel uses its own unwind tables: ORC. You can find its documentation on <https://www.kernel.org/doc/html/latest/x86/orc-unwinder.html> and there is an lwn.net introduction [The ORCs are coming](https://lwn.net/Articles/728339/).

`objtool orc generate a.o` parses `.eh_frame` and generates `.orc_unwind` and `.orc_unwind_ip`. For an object file assembled from:
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

## Compact C Type Format SFrame

In construction.

As an intended format improving on `.eh_frame`, the saving appears too small.

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
Linker generated code (PLT and range extension thunks) generally does not have unwind informatioin coverage.


# 中文版

Stack unwinding主要有以下作用：

* 獲取stack trace，用於debugger、crash reporter、profiler、garbage collector等
* 加上personality routine和language specific data area後實現C++ exceptions(Itanium C++ ABI)。参见[C++ exception handling ABI](/blog/2020-12-12-c++-exception-handling-abi)

Stack unwinding可以分成兩類：

* synchronous: 程序自身觸發的，C++ throw、獲取自身stack trace等。這類stack unwinding只發生在函數調用處(在function body內，不會出現在prologue/epilogue)
* asynchronous: 由signal或外部程序觸發，這類stack unwinding可以發生在函數prologue/epilogue

## Frame pointer

最經典、最簡單的stack unwinding基於frame pointer：固定一個寄存器爲frame pointer(在x86-64上爲RBP)，函數prologue處把frame pointer放入棧幀，並更新frame pointer爲保存的frame pointer的地址。
frame pointer值和棧上保存的值形成了一個單鏈表。獲取初始frame pointer值(`__builtin_frame_address`)後，不停解引用frame pointer即可得到所有棧幀的frame pointer值。
這種方法不適用於prologue/epilogue的部分指令。

```asm
pushq %rbp
movq %rsp, %rbp # after this, RBP references the current frame
...
popq %rbp
retq  # RBP references the previous frame
```

下面是個簡單的stack unwinding例子：
```c
#include <stdio.h>
[[gnu::noinline]] void qux() {
  void **fp = __builtin_frame_address(0);
  for (;;) {
    printf("%p\n", fp);
    void **next_fp = *fp;
    if (next_fp <= fp) break;
    fp = next_fp;
  }
}
[[gnu::noinline]] void bar() { qux(); }
[[gnu::noinline]] void foo() { bar(); }
int main() { foo(); }
```

基於frame pointer的方法簡單，但是有若干缺陷。

上面的代碼用`-O1`或以上編譯時foo和bar會tail call，程序輸出不會包含foo bar的棧幀(`-fomit-leaf-frame-pointer`並不阻礙tail call)。

實踐中，有時候不能保證所有庫都包含frame pointer。unwind一個線程時，爲了增強健壯性需要檢測一個`next_fp`是否像棧地址。檢測的一種方法是解析`/proc/*/maps`判斷地址是否可讀(慢)，另一種是
```c
#include <fcntl.h>
#include <unistd.h>

// Or use the write end of a pipe.
int fd = open("/dev/random", O_WRONLY);
if (write(fd, address, 1) < 0)
  // not readable
```

Linux上`rt_sigprocmask`更好：
```
#include <errno.h>
#include <fcntl.h>
#include <syscall.h>
#include <unistd.h>

errno = 0;
syscall(SYS_rt_sigprocmask, ~0, address, (void *)0, /*sizeof(kernel_sigset_t)=*/8);
if (errno == EFAULT)
  // not readable
```

另外，預留一個寄存器用於frame pointer會增大text size、有性能開銷(prologue、epilogue額外的指令開銷和少一個寄存器帶來的寄存器壓力)，在寄存器貧乏的x86-32可能相當顯著，在寄存器較爲充足的x86-64可能也有1%以上的性能損失。

### 編譯器行爲

* -O0: 預設`-fno-omit-frame-pointer`，所有函數都有frame pointer
* -O1或以上: 預設`-fomit-frame-pointer`，只有必要情況才設置frame pointer。指定`-fno-omit-leaf-frame-pointer`則可得到類似-O0效果。可以額外指定`-momit-leaf-frame-pointer`去除leaf functions的frame pointer

## libunwind

C++ exception、profiler/crash reporter的stack unwinding通常用libunwind API和DWARF Call Frame Information。上個世紀90年代Hewlett-Packard定義了一套libunwind API，分爲兩類：

* `unw_*`: 入口是`unw_init_local`(local unwinding，當前進程)和`unw_init_remote`(remote unwinding，其他進程)。通常使用libunwind的應用使用這套API。比如Linux perf會調用`unw_init_remote`
* `_Unwind_*`: 這部分標準化爲[Itanium C++ ABI: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)的Level 1: Base ABI。Level 2 C++ ABI調用這些`_Unwind_*` API。其中的`_Unwind_Resume`是唯一被C++編譯後的代碼直接調用的API，其中的`_Unwind_Backtrace`被少數應用用於獲取backtrace，其他函數則會被libsupc++/libc++abi調用。

Hewlett-Packard開源了<https://www.nongnu.org/libunwind/>(除此之外還有很多叫做"libunwind"的項目)。這套API在Linux上的常見實現是：

* libgcc/unwind-* (`libgcc_s.so.1`或`libgcc_eh.a`): 實現了`_Unwind_*`並引入了一些擴展：`_Unwind_Resume_or_Rethrow, _Unwind_FindEnclosingFunction, __register_frame`等
* llvm-project/libunwind (`libunwind.so`或`libunwind.a`)是HP API的一個簡化實現，提供了部分`unw_*`，但沒有實現`unw_init_remote`。部分代碼取自ld64。使用Clang的話可以用`--rtlib=compiler-rt --unwindlib=libunwind`選擇
* glibc的`_Unwind_Find_FDE`內部實現，通常不導出，和`__register_frame_info`有關

## DWARF Call Frame Information

程序不同區域需要的unwind指令由DWARF Call Frame Information (CFI)描述，在ELF平臺上由`.eh_frame`存儲。Compiler/assembler/linker/libunwind提供相應支持。

`.eh_frame`由Common Information Entry (CIE)和Frame Description Entry (FDE)組成。CIE有這些字段：

* length
* CIE\_id: 常數0。該字段用於區分CIE和FDE，在FDE中該字段非0，爲CIE\_pointer
* version: 常數1
* augmentation: 描述CIE/FDE參數列表的字串。`P`字符表示personality routine指針；`L`字符表示FDE的augmentation data存儲了language-specific data area (LSDA)
* address\_size: 一般爲4或8
* segment\_selector\_size: for x86
* code\_alignment\_factor: 假設指令長度都是2或4的倍數(用於RISC)，影響`DW_CFA_advance_loc`等的參數的乘數
* data\_alignment\_factor: 影響`DW_CFA_offset DW_CFA_val_offset`等的參數的乘數
* return\_address\_register
* augmentation\_data\_length
* augmentation\_data: personality
* initial\_instructions
* padding

每個FDE有一個關聯的CIE。FDE有這些字段：

* length: FDE自身長度。若爲0xffffffff，接下來8字節(extended\_length)記錄實際長度。除非特別構造，extended\_length是用不到的
* CIE\_pointer: 從當前位置減去CIE\_pointer得到關聯的CIE
* initial\_location: 該FDE描述的第一個位置的地址。在.o中此處有一個引用section symbol的relocation
* address\_range: initial\_location和address\_range描述了一個地址區間
* instructions: unwind時的指令
* augmentation\_data\_length
* augmentation\_data: 如果關聯的CIE augmentation包含`L`字符，這裏會記錄language-specific data area
* padding

CIE引用text section中的personality。FDE引用`.gcc_except_table`中的LSDA。personality和lsda用於Itanium C++ ABI的Level 2: C++ ABI。

`.eh_frame`基於DWARF v2引入的`.debug_frame`。它們有一些區別：

* `.eh_frame`帶有`SHF_ALLOC` flag(標誌一個section是否應爲內存中鏡像的一部分)而`.debug_frame`沒有，因此後者的使用場景非常少。
* `debug_frame`支持DWARF64格式(支持64-bit offsets但體積會稍大)而`.eh_frame`不支持(其實可以拓展，但是缺乏需求)
* `.debug_frame`的CIE中沒有augmentation\_data\_length和augmentation\_data
* CIE中version的值不同
* FDE中CIE\_pointer的含義不同。`.debug_frame`中表示一個section offset(absolute)而`.eh_frame`中表示一個relative offset。`.eh_frame`作出的這一改變很好。如果`.eh_frame`長度超過32-bit，`.debug_frame`得轉換成DWARF64才能表示CIE\_pointer，而relative offset則無需擔心這一問題(如果FDE到CIE的距離超過32-bit了，追加一個CIE即可)

對於如下的函數：
```c
void f() {
  __builtin_unwind_init();
}
```

編譯器用`.cfi_*`(CFI directive)標註彙編，`.cfi_startproc`和`.cfi_endproc`標識FDE區域，其他CFI directives描述CFI instructions。
一個call frame用棧上的一個地址表示。這個地址叫做Canonical Frame Address (CFA)，通常是call site的stack pointer值。下面用一個例子描述CFI instructions的作用：
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

彙編器解析CFI directives生成`.eh_frame`(這套機制由Alan Modra在2003年引入)。Linker收集.o中的`.eh_frame` input sections生成output `.eh_frame`。
2006年GNU as引入了`.cfi_personality`和`.cfi_lsda`。

### `.eh_frame_hdr`和`PT_GNU_EH_FRAME`

定位一個pc所在的FDE需要從頭掃描`.eh_frame`，找到合適的FDE(pc是否落在initial\_location和address\_range表示的區間)，所花時間和掃描的CIE和FDE記錄數相關。
<https://sourceware.org/pipermail/binutils/2001-December/015674.html>引入了`.eh_frame_hdr`，包含binary search index table描述(initial\_location, FDE address) pairs。

`ld --eh-frame-hdr`可以生成`.eh_frame_hdr`。Linker會另外創建program header `PT_GNU_EH_FRAME`來包含`.eh_frame_hdr`。
Unwinder會尋找`PT_GNU_EH_FRAME`來定位`.eh_frame_hdr`，見下文的例子。

### `__register_frame_info`

在`.eh_frame_hdr`和`PT_GNU_EH_FRAME`發明之前，crtbegin (`crtstuff.c`)中有一個static constructor `frame_dummy`：調用`__register_frame_info`註冊可執行文件的`.eh_frame`。

現在`__register_frame_info`只有`-static`鏈接的程序纔會用到。相應地，如果鏈接時指定`-Wl,--no-eh-frame-hdr`，就無法unwind(如果使用C++ exception則會導致`std::terminate`)。

### libunwind例子

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

如果使用llvm-project/libunwind：
```sh
$CC a.c -Ipath/to/include -Lpath/to/lib -lunwind
```

如果使用nongnu.org/libunwind，兩種選擇：(a) `#include <libunwind.h>`前添加`#define UNW_LOCAL_ONLY` (b) 多鏈接一個庫，x86-64上是`-l:libunwind-x86_64.so`。
使用Clang的話也可用`clang --rtlib=compiler-rt --unwindlib=libunwind -I path/to/include a.c`，除了提供`unw_*`外，能確保不鏈接`libgcc_s.so`

* `unw_getcontext`: 獲取寄存器值(包含PC)
* `unw_init_local`
  + 使用`dl_iterate_phdr`遍歷可執行文件和shared objects，找到包含PC的`PT_LOAD` program header
  + 找到所在module的`PT_GNU_EH_FRAME`(`.eh_frame_hdr`)，存入`cursor`
* `unw_step`
  + 二分搜索PC對應的`.eh_frame_hdr`項，記錄找到的FDE和其指向的CIE
  + 執行CIE中的initial\_instructions
  + 執行FDE中的instructions。維護一個location、CFA，初始指向FDE的initial\_location，指令中`DW_CFA_advance_loc`增加location；`DW_CFA_def_cfa_*`更新CFA；`DW_CFA_offset`表示一個寄存器的值保存在CFA+offset處
  + location大於等於PC時停止。也就是說，執行的指令是FDE instructions的一個前綴

Unwinder根據program counter找到適用的FDE，執行所有在program counter之前的CFI instructions。

有幾種重要的

* `DW_CFA_def_cfa_*`
* `DW_CFA_offset`
* `DW_CFA_advance_loc`

一個`-DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86`的clang，`.text` 51.7MiB、`.eh_frame` 4.2MiB、`.eh_frame_hdr` 646、2個CIE、82745個FDE。

### 註記

CFI instructions適合編譯器生成代碼，而手寫彙編要準確標準每一條指令是繁瑣的，也很容易出錯。
2015年Alex Dowad也musl libc貢獻了awk腳本，解析assembly並自動標註CFI directives。
其實對於編譯器生成的代碼也不容易，對於一個不用frame pointer的函數，調整SP就得同時輸出一條CFI directive重定義CFA。GCC是不解析inline assembly的，因此inline assembly裏調整SP往往會造成不準確的CFI。

```c
void foo() {
  asm("subq $128, %rsp\n"
  // Cannot unwind if -fomit-leaf-frame-pointer
      "nop\n"
      "addq $128, %rsp\n");
}

int main() {
  foo();
}
```

而LLVM裏的CFIInstrInserter可以插入`.cfi_def_cfa_* .cfi_offset .cfi_restore`調整CFA和callee-saved寄存器。

The DWARF scheme also has very low information density. The various compact unwind schemes have made improvement on this aspect. To list a few issues:

* CIE address\_size: nobody uses different values for an architecture. Even if they do (ILP32 ABIs in AArch64 and x86-64), the information is already available elsewhere.
* CIE segment\_selector\_size: It is nice that they cared x86, but x86 itself does not need it anymore:/
* CIE code\_alignment\_factor and data\_alignment\_factor: A RISC architecture with such preference can hard code the values.
* CIE return\_address\_register: I do not know when an architecture wants to use a different register for the return address.
* length: The DWARF's 8-byte form is definitely overengineered... For standard form prologue/epilogue, the field should not be needed.
* initial\_location and address\_range: if a binary search index table is always needed, why do we need the length field?
* instructions: bytecode is flexible but commonly a function prologue/epilogue is of a standard form and the few callee-saved registers can be encoded in a more compact way.
* augmentation\_data: While this provide flexibility, in practice very rarely a function needs anything more than a personality and a LSDA pointer.

#### `SHT_X86_64_UNWIND`

`.eh_frame`在linker/dynamic loader裏有特殊處理，照理應該用一個單獨的section type，但當初設計時卻用了`SHT_PROGBITS`。
在x86-64 psABI中`.eh_frame`的型別爲`SHT_X86_64_UNWIND`(可能是受到Solaris影響)。

* GNU as中，`.section .eh_frame,"a",@unwind`會生成`SHT_X86_64_UNWIND`，而`.cfi_*`則生成`SHT_PROGBITS`。
* Clang 3.8起，`.cfi_*`生成`SHT_X86_64_UNWIND`

`.section .eh_frame,"a",@unwind`很少見(glibc's x86 port,libffi,LuaJIT等少數包)，因此檢查`.eh_frame`的型別是個辨別Clang/GCC object file的好方法:)
我有一個LLD 11.0.0 commit就是在relocatable link時接受兩種型別的`.eh_frame`;-)

給未來架構的建議：在定義processor-specific section types時，請不要把0x70000001 (`SHT_ARM_EXIDX=SHT_IA_64_UNWIND=SHT_PARISC_UNWIND=SHT_X86_64_UNWIND=SHT_LOPROC+1`)用於unwinding以外的用途:)
`SHT_CSKY_ATTRIBUTES=0x70000001`:)

#### Linker角度的問題

通常在COMDAT group和啓用`-ffunction-sections`的情況下，.data/.rodata需要像.text那樣分裂開，但是`.eh_frame`是一個monolithic section。
和很多其他metadata sections一樣，monolithic的主要問題是linker garbage collection有點麻煩。
很其他metadata sections不同的是，簡單的丟棄garbage collecting不是一種選擇：`.eh_frame_hdr`是一個binary search index table，重複/無用的entries會使consumers困惑。

Linker在處理`.eh_frame`時，需要在概念上分裂`.eh_frame`成CIE/FDE。
`--gc-sections`時，概念上的引用關係和實際的relocation是相反的：FDE有一個引用text section的relocation；GC時，若被指向的text section被丟棄，引用它的FDE也應被丟棄。

LLD對`.eh_frame`有些特殊處理：

* `-M`需要特殊代碼
* `--gc-sections`發生在`.eh_frame` deduplication/GC前。CIE中的personality是有效的reference，FDE中的initial\_location應該忽略，FDE中的lsda引用只考慮non-section-group情形
* 在relocatable link中，允許從`.eh_frame`指向一個discarded section(due to COMDAT group rule)的`STT_SECTION` symbol(通常對於discarded section，來自section group外的`STB_LOCAL` relocation應該被拒絕)

## Compact unwind descriptors

在macOS上，Apple設計了compact unwind descriptors機制加速unwinding，理論上這種技術可以用於節省一些`__eh_frame`空間，但並沒有實現。
主要思想是：

* 大多數函數的FDE都有固定的模式(prologue處指定CFA、存儲callee-saved registers)，可以把FDE instructions壓縮爲32-bit。
* CIE/FDE augmentation data描述的personality/lsda很常見，可以提取出來成爲固定字段。

下面只討論64-bit。一個descriptor佔32字節

```asm
.quad _foo
.set L1, Lfoo_end-_foo
.long L1
.long compact_unwind_description
.quad personality
.quad lsda_address
```

如果研究`.eh_frame_hdr`(binary search index table)和`.ARM.exidx`的話，可以知道length字段是冗餘的。

Compact unwind descriptor編碼爲：
```c
uint32_t : 24; // vary with different modes
uint32_t mode : 4;
uint32_t flags : 4;
```

定義了5種mode：

* 0: reserved
* 1: FP-based frame: RBP爲frame pointer，frame size可變
* 2: SP-based frame: 不用frame pointer，frame size編譯期固定
* 3: large SP-based frame: 不用frame pointer，frame size編譯期固定但數值較大，無法用mode 2表示
* 4: DWARF CFI escape

### FP-based frame (`UNWIND_MODE_BP_FRAME`)

Compact unwind descriptor編碼爲：
```c
uint32_t regs : 15;
uint32_t : 1; // 0
uint32_t stack_adjust : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

x86-64上callee-saved寄存器有：RBX,R12,R13,R14,R15,RBP。3 bits可以編碼一個寄存器，15 bits足夠表示除RBP外的5個寄存器(是否保存及保存在哪裏)。
stack\_adjust記錄保存寄存器外的額外棧空間。

### SP-based frame (`UNWIND_MODE_STACK_IMMD`)

Compact unwind descriptor編碼爲：
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t : 3;
uint32_t size : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

cnt表示保存的寄存器數(最大6)。
reg\_permutation表示保存的寄存器的排列的序號。
size\*8表示棧幀大小。

### Large SP-based frame (`UNWIND_MODE_STACK_IND`)

Compact unwind descriptor編碼爲：
```c
uint32_t reg_permutation : 10;
uint32_t cnt : 3;
uint32_t adj : 3;
uint32_t size_offset : 8;
uint32_t mode : 4;
uint32_t flags : 4;
```

和SP-based frame類似。特別的是：棧幀大小是從text section讀取的。RSP調整量通常由`subq imm, %rsp`表示，用size\_offset表示該指令到函數開頭的距離。
實際表示的stack size還要算上adj\*8。

### DWARF CFI escape

如果因爲各種原因，compact unwind descriptor無法表示，就要回退到DWARF CFI。


LLVM實現裏，每一個函數只用一個compact unwind descriptor表示。如果asynchronous stack unwinding發生在epilogue，已有實現無法把它和發生在function body的stack unwinding區分開來。
Canonical Frame Address會計算錯誤，caller-saved寄存器也會錯誤地讀取。
如果發生在prologue，且prologue在push寄存器和`subq imm, $rsp`外有其他指令，也會出錯。
另外如果一個函數啓用了shrink wrapping，prologue可能不在函數開頭處。開頭到prologue間的asynchronous stack unwinding也會出錯。
這個問題似乎多數人都不關心，可能是因爲profiler丟失幾個百分點的profile大家不在乎吧。

其實如果用多個descriptors描述一個函數的各個區域，還是可以準確unwind的。
OpenVMS 2018年提出了[[RFC] Improving compact x86-64 compact unwind descriptors](http://lists.llvm.org/pipermail/llvm-dev/2018-January/120741.html)，可惜沒有相關實現。

## ARM exception handling

分爲`.ARM.exidx`和`.ARM.extab`

`.ARM.exidx`是個binary search index table，由2-word pairs組成。
第一個word是31-bit PC-relative offset to the start of the region。
第二個word用程序描述更加清晰：
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

`tableData & 0x80000000`表示一個compact model entry，否則表示一個generic model entry。

`.ARM.exidx`相當於增強的`.eh_frame_hdr`，compact model相當於內聯了`.eh_frame`中的personality和lsda。考慮下面三種情況：

* 如果不會觸發C++ exception且不會調用可能觸發exception的函數：不需要personality，只需要一個`EXIDX_CANTUNWIND` entry，不需要`.ARM.extab`
* 如果會觸發C++ exception但是不需要landing pad：personality是`__aeabi_unwind_cpp_pr0`，只需要一個compact model的entry，不需要`.ARM.extab`
* 如果有catch：需要`__gxx_personality_v0`，需要`.ARM.extab`

`.ARM.extab`相當於合併的`.eh_frame`和`.gcc_except_table`。

### Generic model

```c
uint32_t personality; // bit 31 is 0
uint32_t : 24;
uint32_t num : 8;
uint32_t opcodes[];   // opcodes, variable length
uint8_t lsda[];       // variable length
```

待補充

## Windows ARM64 exception handling

參見<https://docs.microsoft.com/en-us/cpp/build/arm64-exception-handling>，這是我最欣賞的編碼方案。
支持mid-prolog和mid-epilog的unwinding。支持function fragments(用來表示shrink wrapping等非常規棧幀)。

保存在`.pdata`和`.xdata`兩個sections。

```c
uint32_t function_start_rva;
uint32_t Flag : 2;
uint32_t Data : 30;
```

對於canonical form的函數，使用Packed Unwind Data，不需要`.xdata`記錄；對於Packed Unwind Data無法表示的descriptor，保存在`.xdata`。

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

待補充

## Linux kernel ORC unwind tables

對於x86-64，Linux kernel使用自己的unwind tables：ORC。文檔在<https://www.kernel.org/doc/html/latest/x86/orc-unwinder.html>。lwn.net上有一篇介紹[The ORCs are coming](https://lwn.net/Articles/728339/)。

`objtool orc generate a.o`解析`.eh_frame`並生成`.orc_unwind`和`.orc_unwind_ip`。對於這樣一個.o文件：
```asm
.globl foo
.type foo, @function
foo:
  ret
```

Unwind information在兩個地址發生改變：foo的開頭和末尾，因此需要兩個2 ORC entries。
如果DWARF CFA在函數中間變更(比如因爲push/pop)，可能會需要更多entries。

`.orc_unwind_ip`有兩個entries，表示PC-relative地址。
```
Relocation section '.rela.orc_unwind_ip' at offset 0x2028 contains 2 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000000  0000000500000002 R_X86_64_PC32          0000000000000000 .text + 0
0000000000000004  0000000500000002 R_X86_64_PC32          0000000000000000 .text + 1
```

`.orc_unwind`包含類隔類型爲`orc_entry`的entries，記錄上一個棧幀的IP/SP/BP的存儲位置。
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

你可能會發現這個方案和Apples's compact unwind descriptors的`UNWIND_MODE_BP_FRAME` and `UNWIND_MODE_STACK_IMMD`有相似之處。
ORC方案使用16-bit整數，所以`UNWIND_MODE_STACK_IND`應該是用不到的。
Unwinding時，除了BP以外的多數callee-saved寄存器用不到，所以ORC也沒存儲它們。

Linker會resolve `.orc_unwind_ip`的relocations，並創建`__start_orc_unwind_ip/__stop_orc_unwind_ip/__start_orc_unwind/__stop_orc_unwind`用於定界。
然後，一個host utility `scripts/sorttable`對`.orc_unwind_ip`和`.orc_unwind`進行排序。
運行時`unwind_next_frame`執行下面的步驟unwind一個棧幀：

* 在`.orc_unwind_ip`裏二分搜索需要的ORC entry
* 根據當前SP、`orc->sp_reg`、`orc->sp_offset`獲取上一個棧幀的SP
* 根據`orc->type`和其他信息獲取上一個棧幀的IP
* 根據當前BP、上一個棧幀的SP、`orc->bp_reg`、`orc->bp_offset`獲取上一個棧幀的BP

### LLVM

In LLVM, `Function::needsUnwindTableEntry` decides whether CFI instructions should be emitted: `hasUWTable() || !doesNotThrow() || hasPersonalityFn()`

On ELF targets, if a function has `uwtable` or `personality`, or does not have `nounwind` (`needsUnwindTableEntry`), it marks that `.eh_frame` is needed in the module.
Then, a function gets `.eh_frame` if `needsUnwindTableEntry` or `-g[123]` is specified.

To ensure no `.eh_frame`, every function needs `nounwind`.

`uwtable` is coarse-grained: it does not specify the amount of unwind information.
[[RFC] Asynchronous unwind tables attribute](https://lists.llvm.org/pipermail/llvm-dev/2021-November/153768.html) proposes to make it a gradient.

`lib/CodeGen/AsmPrinter/AsmPrinter.cpp:352`

### 尾声

用於profiling的stack unwinding策略會怎樣發展是個開放問題。
我們有至少三種路線：

* compact unwind
* 硬件支持。
* 主要基於FP
