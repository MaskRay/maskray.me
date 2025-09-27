layout: post
title: Stack unwinding
author: MaskRay
tags: [gcc,llvm]
---

Update in 2025-09.

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

* `-fno-omit-frame-pointer -mno-omit-leaf-frame-pointer`: all functions maintain frame pointers.
* `-fno-omit-frame-pointer -momit-leaf-frame-pointer`: all non-leaf functions maintain frame pointers. Leaf functions don't maintain frame pointers.
* `-fomit-frame-pointer`: all functions omit frame pointers. `arm*-apple-darwin` and `thumb*-apple-darwin` don't support the option.

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
* A relocation from `.eh_frame` to a `STT_SECTION` symbol in a discarded section (due to COMDAT group rule) is permitted. Normally such a `STB_LOCAL` relocation from outside of the group is disallowed.

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

## SFrame

`.sframe` is a lightweight alternative to `.eh_frame` that uses more compact encoding at the cost of reduced flexibility.
It focuses on describing three key elements: the Canonical Frame Address (CFA), the return address, and the frame pointer.
Unlike `.eh_frame`, it does not include personality routines, Language Specific Data Area (LSDA) information, or the ability to encode extra callee-saved registers.

An `.sframe` section contains a header followed by an optional auxiliary header and arrays of Function Descriptor Entries (FDEs) and Frame Row Entries (FREs).

The auxiliary header, which is currently unused, could be a replacement for the `.eh_frame` augmentation data.
This would be useful for things like personality routines, language-specific data areas (LSDAs), and signal frames.

```cpp
struct sframe_header {
  struct {
    uint16_t sfp_magic;
    uint8_t sfp_version;
    uint8_t sfp_flags;
  } sfh_preamble;
  uint8_t sfh_abi_arch;
  int8_t sfh_cfa_fixed_fp_offset;
  // Used by x86-64 to define the return address slot relative to CFA
  int8_t sfh_cfa_fixed_ra_offset;
  // Size in bytes of the auxiliary header, allowing extensibility
  uint8_t sfh_auxhdr_len;
  // Numbers of FDEs and FREs
  uint32_t sfh_num_fdes;
  uint32_t sfh_num_fres;
  // Size in bytes of FREs
  uint32_t sfh_fre_len;
  // Offsets in bytes of FDEs and FREs
  uint32_t sfh_fdeoff;
  uint32_t sfh_freoff;
} ATTRIBUTE_PACKED;
```

Each FDE describes a function's start address and references its associated Frame Row Entries (FREs).

```cpp
struct sframe_func_desc_entry {
  int32_t sfde_func_start_address;
  uint32_t sfde_func_size;
  uint32_t sfde_func_start_fre_off;
  uint32_t sfde_func_num_fres;
  // bits 0-3 fretype: sfre_start_address type
  // bit 4 fdetype: SFRAME_FDE_TYPE_PCINC or SFRAME_FDE_TYPE_PCMASK
  // bit 5 pauth_key: (AArch64 only) the signing key for the return address
  uint8_t sfde_func_info;
  // The size of the repetitive code block for SFRAME_FDE_TYPE_PCMASK; used by .plt
  uint8_t sfde_func_rep_size;
  uint16_t sfde_func_padding2;
} ATTRIBUTE_PACKED;

template <class AddrType>
struct sframe_frame_row_entry {
  // If the fdetype is SFRAME_FDE_TYPE_PCINC, this is an offset relative to sfde_func_start_address
  AddrType sfre_start_address;
  // bit 0 fre_cfa_base_reg_id: define BASE_REG as either FP or SP
  // bits 1-4 fre_offset_count: typically 1 to 3, describing CFA, FP, and RA
  // bits 5-6 fre_offset_size: byte size of offset entries (1, 2, or 4 bytes)
  sframe_fre_info sfre_info;
} ATTRIBUTE_PACKED;
```

Each FRE contains variable-length stack offsets stored as trailing data with sizes of `uint8_t`, `uint16_t`, or `uint32_t`, determined by the `fre_offset_size` field.
The interpretation of these offsets is architecture-specific:

x86-64:

- First offset: Encodes CFA as `BASE_REG + offset`
- Second offset (if present): Encodes FP as `CFA + offset`  
- Implicit return address: Computed as `CFA + sfh_cfa_fixed_ra_offset` (using header field)

AArch64:

- First offset: Encodes CFA as `BASE_REG + offset`
- Second offset: Encodes return address as `CFA + offset`
- Third offset (if present): Encodes FP as `CFA + offset`

SFrame reduces size compared to `.eh_frame` plus `.eh_frame_hdr` by:

- Eliminating `.eh_frame_hdr` through sorted `sfde_func_start_address` fields
- Replacing CIE pointers with direct FDE-to-FRE references
- Using variable-width `sfre_start_address` fields (1 or 2 bytes) for small functions
- Storing start addresses instead of address ranges. `.eh_frame` address ranges
- Start addresses in a small function use 1 or 2 byte fields, more efficient than `.eh_frame` initial\_location, which needs at least 4 bytes (`DW_EH_PE_sdata4`).
- Hard-coding stack offsets rather than using flexible register specifications

However, the bytecode design of `.eh_frame` can sometimes be more efficient than `.sframe`, as demonstrated on x86-64.

The format includes endianness variations that complicate toolchain support.
I think we should use a little-endian format universally, regardless of the target system's native endianness, removing `template <class Endian>` from C++ code.
On the big-endian z/Architecture, this is efficient: the LOAD REVERSED instructions are used by the bswap versions in the following program, not even requiring extra instructions.

```c
#define WIDTH(x) \
typedef __UINT##x##_TYPE__ [[gnu::aligned(1)]] uint##x; \
uint##x load_inc##x(uint##x *p) { return *p+1; } \
uint##x load_bswap_inc##x(uint##x *p) { return __builtin_bswap##x(*p)+1; }; \
uint##x load_eq##x(uint##x *p) { return *p==3; } \
uint##x load_bswap_eq##x(uint##x *p) { return __builtin_bswap##x(*p)==3; }; \

WIDTH(16);
WIDTH(32);
WIDTH(64);
```

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
