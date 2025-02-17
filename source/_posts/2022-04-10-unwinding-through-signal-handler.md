layout: post
title: Unwinding through a signal handler
author: MaskRay
tags: [glibc,libunwind]
---

This post has some notes about unwinding through a signal handler.
You may want to read [Stack unwinding](/blog/2020-11-08-stack-unwinding) first.

```cpp
// a.c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <libunwind.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

static void handler(int signo) {
  unw_context_t context;
  unw_cursor_t cursor;
  unw_getcontext(&context);
  unw_init_local(&cursor, &context);
  unw_word_t pc, sp;
  do {
    unw_get_reg(&cursor, UNW_REG_IP, &pc);
    unw_get_reg(&cursor, UNW_REG_SP, &sp);
    printf("pc=0x%016zx sp=0x%016zx", (size_t)pc, (size_t)sp);
    Dl_info info = {};
    if (dladdr((void *)pc, &info))
      printf(" %s:%s", info.dli_fname, info.dli_sname ? info.dli_sname : "");
    puts("");
  } while (unw_step(&cursor) > 0);
  exit(0);
}

int main() {
  signal(SIGUSR1, handler);
  raise(SIGUSR1);
  return 1;
}
```
(printf and dladdr are not required to be async-signal-safe functions, but here we apparently know using them can't cause problems.)

Tips: we can additionally add the following code block to get memory mappings.
```c
  char buf[128];
  FILE *f = fopen("/proc/self/maps", "r");
  while (fgets(buf, sizeof buf, f))
    printf("%s", buf);
  fclose(f);
```

Build the program with either llvm-project libunwind or nongnu libunwind:
```sh
# ninja -C /tmp/Debug unwind builtins
clang -g -I llvm-project/libunwind/include a.c -no-pie --unwindlib=libunwind --rtlib=compiler-rt -ldl -Wl,-E,-rpath,/tmp/Debug/lib/x86_64-unknown-linux-gnu -o llvm

# autoreconf -i; mkdir -p out/debug; ../../configure CFLAGS='-O0 -g' CXXFLAGS='-O0 -g'; make -j 20
libunwind=/tmp/p/libunwind
clang -g -I $libunwind/include -I $libunwind/out/debug/include a.c -no-pie $libunwind/out/debug/src/.libs/libunwind.a $libunwind/out/debug/src/.libs/libunwind-x86_64.a -llzma -ldl -Wl,-E -o nongnu
```
(Some targets default to `-fno-asynchronous-unwind-tables`. In the absence of C++ exceptions, we need at least `-funwind-tables`.)

## glibc x86-64

With either implementation, the output looks like the following on Linux glibc x86-64. I annotated the lines with location information.
```text
pc=0x0000000000206d2a sp=0x00007fffd366bce0 ./nongnu: # in handler, the instruction after call unw_getcontext
pc=0x00007f5962cb0920 sp=0x00007fffd366c500 /lib/x86_64-linux-gnu/libc.so.6: # __restore_rt
pc=0x00007f5962cb08a1 sp=0x00007fffd366d200 /lib/x86_64-linux-gnu/libc.so.6:gsignal # raise
pc=0x0000000000206cfd sp=0x00007fffd366d320 ./nongnu:main
pc=0x00007f5962c9b7fd sp=0x00007fffd366d340 /lib/x86_64-linux-gnu/libc.so.6:__libc_start_main
pc=0x0000000000206bba sp=0x00007fffd366d410 ./nongnu:_start # from crt1.o
```

`__restore_rt` is a sigreturn trampoline defined in glibc `sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c`:
```asm
  nop
.align 16
__restore_rt:
  movq $15, %rax  # __NR_rt_sigreturn
  syscall
```
(Newer ports use VDSO.)

glibc's `sigaction` sets the `sa_restorer` field of `sigaction` to `__restore_rt`, and sets the `SA_RESTORER`.
The kernel sets up the `__restore_rt` frame with saved process context information (`ucontext_t` structure) before jumping to the signal handler. See kernel `arch/x86/kernel/signal.c:setup_rt_frame`.
Upon returning from the signal handler, control passes to `__restore_rt`. See `man 2 sigreturn`.

`__restore_rt` is implemented in assembly. It comes with DWARF call frame information in `.eh_frame`.

```text
% llvm-dwarfdump -eh-frame /lib/x86_64-linux-gnu/libc.so.6
...
00002458 00000010 00000000 CIE
  Format:                DWARF32
  Version:               1
  Augmentation:          "zRS"
  Code alignment factor: 1
  Data alignment factor: -8
  Return address column: 16
  Augmentation data:     1B

  DW_CFA_nop:
  DW_CFA_nop:


0000246c 00000078 00000018 FDE cie=00002458 pc=0003c91f...0003c929
  Format:       DWARF32
  DW_CFA_def_cfa_expression: DW_OP_breg7 RSP+160, DW_OP_deref
  DW_CFA_expression: R8 DW_OP_breg7 RSP+40
  DW_CFA_expression: R9 DW_OP_breg7 RSP+48
  DW_CFA_expression: R10 DW_OP_breg7 RSP+56
  DW_CFA_expression: R11 DW_OP_breg7 RSP+64
  DW_CFA_expression: R12 DW_OP_breg7 RSP+72
  DW_CFA_expression: R13 DW_OP_breg7 RSP+80
  DW_CFA_expression: R14 DW_OP_breg7 RSP+88
  DW_CFA_expression: R15 DW_OP_breg7 RSP+96
  DW_CFA_expression: RDI DW_OP_breg7 RSP+104
  DW_CFA_expression: RSI DW_OP_breg7 RSP+112
  DW_CFA_expression: RBP DW_OP_breg7 RSP+120
  DW_CFA_expression: RBX DW_OP_breg7 RSP+128
  DW_CFA_expression: RDX DW_OP_breg7 RSP+136
  DW_CFA_expression: RAX DW_OP_breg7 RSP+144
  DW_CFA_expression: RCX DW_OP_breg7 RSP+152
  DW_CFA_expression: RSP DW_OP_breg7 RSP+160
  DW_CFA_expression: RIP DW_OP_breg7 RSP+168
  DW_CFA_nop:
  DW_CFA_nop:

  0x3c91f: CFA=DW_OP_breg7 RSP+160, DW_OP_deref: RAX=[DW_OP_breg7 RSP+144], RDX=[DW_OP_breg7 RSP+136], RCX=[DW_OP_breg7 RSP+152], RBX=[DW_OP_breg7 RSP+128], RSI=[DW_OP_breg7 RSP+112], RDI=[DW_OP_breg7 RSP+104], RBP=[DW_OP_breg7 RSP+120], RSP=[DW_OP_breg7 RSP+160], R8=[DW_OP_breg7 RSP+40], R9=[DW_OP_breg7 RSP+48], R10=[DW_OP_breg7 RSP+56], R11=[DW_OP_breg7 RSP+64], R12=[DW_OP_breg7 RSP+72], R13=[DW_OP_breg7 RSP+80], R14=[DW_OP_breg7 RSP+88], R15=[DW_OP_breg7 RSP+96], RIP=[DW_OP_breg7 RSP+168]
...
```

The `DW_OP_breg7 RSP` offsets correspond to the `ucontext_t` offsets of these registers.
```text
% cat sysdeps/unix/sysv/linux/x86_64/libc_sigaction.c
...
   do_cfa_expr                                                          \
   do_expr (8 /* r8 */, oR8)                                            \
   do_expr (9 /* r9 */, oR9)                                            \
   do_expr (10 /* r10 */, oR10)                                         \
% cat sysdeps/unix/sysv/linux/x86_64/ucontext_i.sym
...
#define ucontext(member)        offsetof (ucontext_t, member)
#define mcontext(member)        ucontext (uc_mcontext.member)
#define mreg(reg)               mcontext (gregs[REG_##reg])

oRBP            mreg (RBP)
oRSP            mreg (RSP)
oRBX            mreg (RBX)
```

With the information, libunwind can unwind through the sigreturn trampoline without knowing the `ucontext_t` structure.
Note that all general purpose registers are encoded. `libunwind/docs/unw_get_reg.man` says

> However, for signal frames (see unw_is_signal_frame(3)), it is usually possible to access all registers.

Volatile registers are also saved in the saved process context information.
This is different from other frames where volatile registers' information is typically lost.

## glibc AArch64

The output looks like:
```text
pc=0x0000000000214b10 sp=0x0000ffffe81f6050 ./nongnu: # handler
pc=0x0000ffffa55cd5b0 sp=0x0000ffffe81f6c70 linux-vdso.so.1:__kernel_rt_sigreturn
pc=0x0000ffffa5438070 sp=0x0000ffffe81f7ed0 /lib/aarch64-linux-gnu/libc.so.6:gsignal
pc=0x0000000000214bfc sp=0x0000ffffe81f8000 ./nongnu:main
pc=0x0000ffffa5425090 sp=0x0000ffffe81f8010 /lib/aarch64-linux-gnu/libc.so.6:__libc_start_main
pc=0x00000000002149cc sp=0x0000ffffe81f8160 ./nongnu: # _start
pc=0x00000000002149cc sp=0x0000ffffe81f8160 ./nongnu: # _start
```

As a relatively new port, Linux AArch64 defines the sigreturn trampoline `__kernel_rt_sigreturn` in the VDSO (see `arch/arm64/kernel/vdso/sigreturn.S`). This is unlike x86-64 which defines the function in libc.
We can use gdb to dump the VDSO.

```text
(gdb) i proc m
process 430749
...
      0xfffff7ffc000     0xfffff7ffd000     0x1000        0x0 [vdso]
(gdb) dump binary memory vdso.so 0xfffff7ffc000     0xfffff7ffd000
```

```asm
  nop

.globl __kernel_rt_sigreturn
__kernel_rt_sigreturn:
  mov     x8, #__NR_rt_sigreturn  // 0xad
  svc     #0x0
```

As of Linux 5.8 (<https://git.kernel.org/linus/87676cfca14171fc4c99d96ae2f3e87780488ac4>), `vdso.so` does not have `PT_GNU_EH_FRAME`. Therefore unwinders (llvm-project libunwind, nongnu libunwind, `libgcc_s.so.1`) ignore its unwind tables.
In gdb, `gdb/aarch64-linux-tdep.c` recognizes the two instructions and encodes how the kernel sets up the `ucontext_t` structure.

Previously, `vdso.so` generated a small set of CFI instructions to encode X29 (FP) and X30 (LR).
```text
% llvm-dwarfdump -eh-frame vdso.so
000000c0 0000001c 00000000 CIE
  Format:                DWARF32
  Version:               1
  Augmentation:          "zRS"
  Code alignment factor: 4
  Data alignment factor: -8
  Return address column: 30
  Augmentation data:     1B

  DW_CFA_def_cfa: WSP +0
  DW_CFA_def_cfa: W29 +0
  DW_CFA_offset: W29 0
  DW_CFA_offset_extended_sf: W30 8
  DW_CFA_nop:
  DW_CFA_nop:
  DW_CFA_nop:

  CFA=W29: W29=[CFA], W30=[CFA+8]

000000e0 00000010 00000024 FDE cie=000000c0 pc=000005b0...000005b8
  Format:       DWARF32
  DW_CFA_nop:
  DW_CFA_nop:
  DW_CFA_nop:

  0x5b0: CFA=W29: W29=[CFA], W30=[CFA+8]

```

However, there was a serious problem: CFI cannot describe a sigreturn trampoline frame.
AArch64 does not define a register number for PC and provides no direct way to encode the PC of the previous frame.
Instead, it sets return\_address\_register to X30 and the unwinder updates the PC to whatever value the saved X30 is.
Actually, with nongnu libunwind and `unw_get_reg(&cursor, UNW_REG_IP, &pc); unw_get_reg(&cursor, UNW_AARCH64_X30, &x30);`, we know `pc == x30`.
This approach works fine when LR forms a chain since we know between two adjacent frames, the sets `{PC, X30}` differ by one element.
However, when unwinding through the sigreturn trampoline, the CFI can describe the previous PC but not the previous X30.

## musl x86-64

`src/signal/x86_64/restore.s` implements a sigreturn trampoline `__restore_rt`. There is no `.eh_frame` information.

nongnu libunwind does not know that `__restore_rt` is a sigreturn trampoline (`unw_is_signal_frame` always returns 0).
On ELF targets, `-O1` and above typically imply `-fomit-frame-pointer` and many functions do not save RBP.
Note: some functions may save RBP even with `-fomit-frame-pointer`.

In the absence of a valid frame chain, combined with the fact that nongnu libunwind does not recognize Linux x86-64's sigreturn trampoline, libunwind cannot unwind through the `__restore_rt` frame.
gdb recognizes the sigreturn trampoline frame and with its FP-based unwinding it can retrieve several frames, but not the ones above raise.

```text
% ld.lld @response.release.txt && ./nongnu
pc=0x0000000000206add sp=0x00007ffc018618a0 0 ./nongnu:
pc=0x00007f9fedcd602f sp=0x00007ffc018620c0 0 /home/ray/musl/out/release/lib/libc.so:
pc=0x0000000000000000 sp=0x00007ffc01862db0 0
% gdb ./nongnu -x =(printf 'b handler\nhandle SIGUSR1 nostop\nr\nbt')
...
#0  handler (signo=10) at a.c:9
#1  <signal handler called>
#2  0x00007ffff7fae78a in __restore_sigs () from /home/ray/musl/out/release/lib/libc.so
#3  0x00007ffff7fae8f1 in raise () from /home/ray/musl/out/release/lib/libc.so
#4  0x0000000000000000 in ?? ()
```

If musl is built with `-fno-omit-frame-pointer`, nongnu libunwind will use its FP-based fallback (see `src/x86_64/Gstep.c`). The output looks like:
```text
pc=0x0000000000206ada sp=0x00007fffd51b1830 0 ./nongnu:
pc=0x00007f0f09352858 sp=0x00007fffd51b2040 0 /home/ray/musl/out/release-fp/lib/libc.so:__setjmp
pc=0x0000000000206aaa sp=0x00007fffd51b2db0 0 ./nongnu:main
pc=0x00007f0f092f88ec sp=0x00007fffd51b2dd0 0 /home/ray/musl/out/release-fp/lib/libc.so:
pc=0x00000000002069d6 sp=0x00007fffd51b2e00 0 ./nongnu:_start
```

`unw_step` uses the saved RBP to infer RSP/RBP/RIP in the previous frame.
If the signal handler saves RBP and calls `unw_step`, the saved RBP is essentially the RBP value in the  frame.
```c
rbp_loc = DWARF_LOC(rbp, 0);
rsp_loc = DWARF_VAL_LOC(c, rbp + 16);
rip_loc = DWARF_LOC (rbp + 8, 0);
```

Actually, not every source file needs to be built with `-fno-omit-frame-pointer`.
We just need to build the source files that transfer control to the user program, and their callers.
For this example, building `src/signal/raise.c` with `-fno-omit-frame-pointer` allows us to unwind to `main`.
Additionally rebuilding `src/env/__libc_start_main.c` allows us to unwind to `_start`.

musl's Makefile specifies `-fno-asynchronous-unwind-tables` (see [option to enable eh_frame](https://www.openwall.com/lists/musl/2021/07/16/1) for a 2011 discussion).
If CFLAGS `-g` is specified, `libc.so` will have `.debug_frame`.
gdb can retrieve the caller of `raise`:
```text
#0  handler (signo=10) at a.c:9
#1  <signal handler called>
#2  __restore_sigs (set=set@entry=0x7fffffffe240) at ../../arch/x86_64/syscall_arch.h:40
#3  0x00007ffff7fa36e0 in raise (sig=sig@entry=10) at ../../src/signal/raise.c:11
#4  0x00000000002071ff in main () at a.c:33
```

nongnu libunwind can be built with `--enable-debug-frame` to support `.debug_frame`.
Unfortunately, since it does not recognize the , it cannot retrieve the `main` frame for this example.

## RISC-V

Like AArch64, Linux RISC-V defines the sigreturn trampoline [`__vdso_rt_sigreturn`](https://github.com/torvalds/linux/blob/master/arch/riscv/kernel/vdso/rt_sigreturn.S) in the VDSO.
```asm
ENTRY(__vdso_rt_sigreturn)
	.cfi_startproc
	.cfi_signal_frame
	li a7, __NR_rt_sigreturn
	scall
	.cfi_endproc
ENDPROC(__vdso_rt_sigreturn)
```

llvm-project libunwind added support for unwinding through a sigreturn trampoline in <https://reviews.llvm.org/D148499> (2023-05).

The output looks like the following on Arch Linux RISC-V (riscv64gc).
```
[root@archlinux riscv64]# ./b
pc=0x0000000000010a82 sp=0x00007fffcdd00910 ./b:
pc=0x00007fff8af83800 sp=0x00007fffcdd01520 linux-vdso.so.1:__vdso_rt_sigreturn
pc=0x00007fff8ae7fbee sp=0x00007fffcdd01960 /usr/lib/libc.so.6:
pc=0x00007fff8ae4ad66 sp=0x00007fffcdd019b0 /usr/lib/libc.so.6:gsignal
pc=0x0000000000010a3c sp=0x00007fffcdd019c0 ./b:main
pc=0x00007fff8ae3b1d4 sp=0x00007fffcdd019e0 /usr/lib/libc.so.6:
pc=0x00007fff8ae3b27c sp=0x00007fffcdd01b10 /usr/lib/libc.so.6:__libc_start_main
pc=0x00000000000109a0 sp=0x00007fffcdd01b60 ./b:_start
```

## Unwinders' compatibility with libc implementations

The values represent how the unwinder unwinds through the  frame.

<style type="text/css">
table.segment-tree {
  table-layout: fixed;
}
table.segment-tree td {
  border: 1px solid #ddd;
  text-align: center;
}
</style>

<table>
<thead><td></td><td>Linux glibc</td><td>Linux musl</td></thead>
<tr><td>nongnu libunwind AArch64</td><td>recognizes  in VDSO</td><td>not tested</td></tr>
<tr><td>nongnu libunwind x86-64</td><td>.eh_frame in libc.so.6</td><td>unwindable if FP is enabled</td></tr>
<tr><td>gdb AArch64</td><td>recognizes  in VDSO</td><td>not tested</td></tr>
<tr><td>gdb x86-64</td><td>recognizes </td><td>recognizes sigreturn trampoline</td></tr>
</table>

Links to  frame related code

* gcc `libgcc/config/aarch64/linux-unwind.h:aarch64_fallback_frame_state`
* gdb `gdb/aarch64-linux-tdep.c:aarch64_linux_rt_sigframe`, `gdb/amd64-linux-tdep.c:amd64_linux_sigtramp_start`, `gdb/riscv-linux-tdep.c:riscv_linux_sigframe`
* llvm-project libunwind <https://reviews.llvm.org/D90898>. We now use [`syscall(SYS_rt_sigprocmask, ...)`](https://github.com/llvm/llvm-project/pull/74791) to check whether a PC pointer is addressable.
* Linux kernel `arch/x86/kernel/signal.c:setup_rt_frame` `arch/riscv/kernel/vdso/rt_sigreturn.S:__vdso_rt_sigreturn`

## Core dump

The kernel core dumper `coredump.c` is simple.
The glibc `__restore_rt` page or the VDSO is not prioritized in the presence of a core file limit.
If the page is missing in the core file, `gdb prog core -ex bt -batch` will not be able to unwind past the .
A userspace core dumper may be handy.

## Frame pointer based unwinding

```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

[[gnu::noinline]] void unwind() {
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
    printf("%p %p", next_fp, pc);
    Dl_info info = {};
    if (dladdr((void *)pc, &info))
      printf(" %s:%s", info.dli_fname, info.dli_sname ? info.dli_sname : "");
    puts("");

    if (next_fp <= fp) break;
    fp = next_fp;
  }
}

void handler() {
  unwind();
}

[[gnu::noinline]] void qux() { signal(SIGUSR1, handler); raise(SIGUSR1); }
[[gnu::noinline]] void bar() { qux(); }
[[gnu::noinline]] void foo() { bar(); }
int main() { foo(); }
```
