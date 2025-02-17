layout: post
title: Control-flow integrity
author: MaskRay
tags: [llvm]
---

Updated in 2023-05.

A control-flow graph (CFG) is a graph representation of all paths that might be traversed through a program during its execution.
Control-flow integrity (CFI) refers to security policy dictating that program execution must follow a control-flow graph.
This article describes some features that compilers and hardware can use to enforce CFI, with a focus on llvm-project implementations.

CFI schemes are typically divided into forward-edge (e.g. indirect calls) and backward-edge (mainly function returns).
It should be noted that exception handling and symbol interposition are not included in these categories, as far as my understanding goes.

<!-- more -->

Let's start with backward-edge CFI.
Fine-grained schemes check whether a return address refers to a possible caller in the control-flow graph.
However, this is a very difficult problem, and the additional guarantee may not be that useful.

Coarse-grained schemes, on the other hand, simply just check that return addresses have not been tampered with.
Return addresses are typically stored in a memory region called the "stack" along with function arguments, local variables, and register save areas.
Stack smashing is an attack that overwrites the return address to hijack the control flow.
The name was made popular by Aleph One (Elias Levy)'s paper _Smashing The Stack For Fun And Profit_.

## StackGuard/Stack Smashing Protector

_StackGuard: Automatic Adaptive Detection and Prevention of Buffer-Overflow Attacks_ (1998) introduced an approach to detect tampered return addresses on the stack.

GCC 4.1 [implemented](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=7d69de618e732d343228a07d797a30e39a6363f4) `-fstack-protector` and `-fstack-protector-all`.
Additional variants have been added over the years, such as [`-fstack-protector-strong`](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=f6bc1c4a12af78d96c951547d9693e6e805162da) for GCC 4.9 and [`-fstack-protector-explicit`](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=5434dc0730795cf6a2ef8a9fe20e4dcc9cd077be) in 2015-01.

* In the prologue, the canary is loaded from a secret location and placed before the return address on the stack.
* In the epilogue, the canary is loaded again and compared with the entry before the return address.

The idea is that an attack overwriting the return address will likely overwrite the canary value as well.
If the attacker doesn't know the canary value, returning from the function will crash.

The canary is stored either in a libc global variable (`__stack_chk_guard`) or a field in the thread control block (e.g. `%fs:40` on x86-64).
Every architecture may have a preference on the location.
In a glibc port, only one of `-mstack-protector-guard=global` and `-mstack-protector-guard=tls` is supported ([BZ #26817](https://sourceware.org/bugzilla/show_bug.cgi?id=26817)).
(Changing this requires symbol versioning wrestling and may not be [necessary](https://sourceware.org/pipermail/libc-alpha/2021-January/121618.html) as we can move to a more fine-grained scheme.)
Under the current thread stack allocation scheme, the thread control block is allocated next to the thread stack.
This means that a large out-of-bounds stack write can potentially overwrite the canary.

```c
void foo(const char *a) {
  char s[10];
  strcpy(s, a);
  puts(s);
}
```
```asm
# x86-64 assembly
  movq    %fs:40, %rax                      # load the canary from the thread control block
  movq    %rax, 24(%rsp)                    # place the value before the return address
  ...
  movq    %fs:40, %rax                      # load the canary again
  cmpq    24(%rsp), %rax                    # compare it with the entry before the return address
  jne     .LBB0_2                           # fail if mismatching
  <epilogue>
.LBB0_2:
  callq   __stack_chk_fail@PLT

# AArch64 assembly
  adrp    x8, __stack_chk_guard
  ldr     x8, [x8, :lo12:__stack_chk_guard] # load the canary from a global variable
  stur    x8, [x29, #-8]                    # place the value before the saved frame pointer/return address
  ...
  adrp    x8, __stack_chk_guard
  ldur    x9, [x29, #-8]                    # load the entry before the saved frame pointer/return address
  ldr     x8, [x8, :lo12:__stack_chk_guard] # load the canary again
  cmp     x8, x9
  b.ne    .LBB0_2                           # fail if mismatching
  <epilogue>
.LBB0_2:
  bl      __stack_chk_fail
```

GCC source code lets either libssp or libc provide `__stack_chk_guard` (see `TARGET_LIBC_PROVIDES_SSP`). It seems that libssp is obsoleted for modern systems.

musl [sets](74a28a8af21977ebbc2945beb879f1b9b6ff13ba) `((char *)&__stack_chk_guard)[1] = 0;`:

* The NUL byte serves as a terminator to sabotage string reads as information leak.
* A string overflow which attempts to preserve the canary while overwriting bytes after the canary (implementation detail of `struct pthread`) will fail.
* One byte overwrite is still detectable.
* The set of possible values is decreased, but this seems acceptable for 64-bit systems.

There are two related function attributes: `stack_protect` and `no_stack_protector`. Strangely `stack_protect` is not named `stack_protector`.

`-mstack-protector-guard-symbol=` can change the global variable name from `__stack_chk_guard` to something else.
The option was introduced for the Linux kernel.
See `arch/*/include/asm/stackprotector.h` for how the canary is initialized in the Linux kernel. It is at least per-cpu.
Some ports support [per-task stack canary](https://git.kernel.org/linus/0a1213fa7432778b71a1c0166bf56660a3aab030) (search `config STACKPROTECTOR_PER_TASK`).

## Retguard

OpenBSD introduced Retguard in 2017 to improve security. Details can be found in [RETGUARD and stack canaries](https://isopenbsdsecu.re/mitigations/retguard/).
Retguard is more fine-grained than StackGuard, but it has more expensive function prologues and epilogues.

For each instrumented function, a cookie is allocated from a pool of 4000 entries.
In the prologue, `return_address ^ cookie` is pushed next to the return address (similar to the [XOR Random Canary](http://phrack.org/issues/56/5.html)).
The epilogue pops the XOR value and the return address and verifies that they match (`(value ^ cookie) == return_address`).

Not encrypting the return address directly is important to preserve return address prediction for the CPU.
However, if a static branch predictor exists (likely not-taken for a forward branch) the initial prediction is likely wrong.
The two `int3` instructions are used to disrupt ROP gadgets that may form from `je ...; retq` (`02 cc cc c3` is `addb %ah, %cl; int3; retq`), but they don't remove the `pop+ret` gadget.
([ROP gadgets removal](https://isopenbsdsecu.re/mitigations/rop_removal/) notes that gadget removal may not be useful.)

```asm
// From https://www.openbsd.org/papers/eurobsdcon2018-rop.pdf
// prologue
ffffffff819ff700: 4c 8b 1d 61 21 24 00 mov 2367841(%rip),%r11 # <__retguard_2759>
ffffffff819ff707: 4c 33 1c 24          xor (%rsp),%r11
ffffffff819ff70b: 55                   push %rbp
ffffffff819ff70c: 48 89 e5             mov %rsp,%rbp
ffffffff819ff70f: 41 53                push %r11

// epilogue
ffffffff8115a457: 41 5b                pop %r11
ffffffff8115a459: 5d                   pop %rbp
ffffffff8115a45a: 4c 33 1c 24          xor (%rsp),%r11
ffffffff8115a45e: 4c 3b 1d 03 74 ae 00 cmp 11432963(%rip),%r11 # <__retguard_2759>
ffffffff8115a465: 74 02                je ffffffff8115a469
ffffffff8115a467: cc                   int3
ffffffff8115a468: cc                   int3
ffffffff8115a469: c3                   retq
```

## SafeStack

[Code-Pointer Integrity](https://dslab.epfl.ch/pubs/cpi.pdf) (2014) proposed stack object instrumentation, which was merged into LLVM in 2015.
To use it, we can run `clang -fsanitize=safe-stack`. In a link action, the driver option links in the runtime `compiler-rt/lib/safestack`.

The pass moves some stack objects into a separate stack, which is normally referenced by a thread-local variable `__safestack_unsafe_stack_ptr` or via a function call `__safestack_pointer_address`.
These objects include those that are not guaranteed to be free of stack smashing, mainly via `ScalarEvolution`.

As an example, the local variable `a` below is moved to the unsafe stack as there is a risk that `bar` may have out-of-bounds accesses.

```c
void bar(int *);
void foo() {
  int a;
  bar(&a);
}
```

```asm
foo:                                    # @foo
# %bb.0:                                # %entry
        pushq   %r14
        pushq   %rbx
        pushq   %rax
        movq    __safestack_unsafe_stack_ptr@GOTTPOFF(%rip), %rbx
        movq    %fs:(%rbx), %r14
        leaq    -16(%r14), %rax
        movq    %rax, %fs:(%rbx)
        leaq    -4(%r14), %rdi
        callq   bar@PLT
        movq    %r14, %fs:(%rbx)
        addq    $8, %rsp
        popq    %rbx
        popq    %r14
        retq
```

## Shadow stack

This technique is very old. [Stack Shield](https://www.angelfire.com/sk/stackshield/) (1999) is an early scheme based on assembly instrumentation.

During a function call, the return address is stored in a shadow stack. The normal stack may contain a copy, or (as a variant) not at all.
Upon return, an entry is popped from the shadow stack. It is either used as the return address or (as a variant) compared with the normal return address.

Below we describe some userspace and hardware-assisted schemes.

### Userspace

#### `-fsanitize=shadow-call-stack`

See <https://clang.llvm.org/docs/ShadowCallStack.html>.
During a function call, the instrumentation stores the return address in a shadow stack.
When returning from the function, the return address is popped from the shadow stack.
Additionally, the return address is stored on the regular stack for return address prediction and compatibility with unwinders, but it is otherwise unused.

To use this for AArch64, run `clang --target=aarch64-unknown-linux-gnu -fsanitize=shadow-call-stack -ffixed-x18`.
```asm
str	x30, [x18], #8      // push the return address to the shadow stack
sub	sp, sp, #32
stp	x29, x30, [sp, #16] // the normal stack contains a copy as well
...
ldp	x29, x30, [sp, #16]
add	sp, sp, #32
ldr	x30, [x18, #-8]!    // pop the return address from the shadow stack
ret
```

GCC [ported](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=102768) the feature in 2022-02 (milestone: 12.0).

This is [implemented](https://reviews.llvm.org/D84414) for RISC-V as well.
The implementation has switched from x18 to the platform register x3 (gp).

```asm
// clang --target=riscv64-unknown-linux-gnu -fsanitize=shadow-call-stack
addi    gp, gp, 8
sd      ra, -8(gp)      // push the return address to the shadow stack
addi    sp, sp, -32
sd      ra, 24(sp)      // the normal stack contains a copy as well
...
addi    sp, sp, 32
ld      ra, -8(s2)      // pop the return address from the shadow stack
addi    gp, gp, -8
ret
```

When using RVC, the push operation on the shadow stack uses 6 bytes.
The `addi gp, gp, 8; sd ra, -8(gp)` code sequence is deliberate.
If we do `sd; addi` instead, and a `-fsanitize=shadow-call-stack` instrumented signal handler happens right after `sd`, after returning from the signal handler the shadow stack entry will be the incorrect value from the signal handler.

In the Linux kernel, select `CONFIG_SHADOW_CALL_STACK` to use this scheme.

### Hardware-assisted

#### Intel Control-flow Enforcement Technology and AMD Shadow Stack

Supported by Intel's 11th Gen and AMD Zen 3.

A RET instruction pops the return address from both the regular stack and the shadow stack, and compares them.
A control protection exception (#CP) is raised in case of a mismatch.

On Windows the scheme is branded as Hardware-enforced Stack Protection.
In the MSVC linker, `/cetcompat` marks an executable image as compatible with Control-flow Enforcement Technology (CET) Shadow Stack.

`setjmp/longjmp` need to save and restore the shadow stack pointer.

Linux v6.6 introduced user-space API for shadow stack (`arch_prctl` values `ARCH_SHSTK_ENABLE/ARCH_SHSTK_DISABLE/ARCH_SHSTK_LOCK`).

### Stack unwinding

By storing only return addresses, the shadow stack allows for an efficient stack unwinding scheme.
As a result, the frame pointer is no longer necessary, as we no longer need a frame chain to trace through return addresses.

However, in many implementations, accessing the shadow stack from a different process can be challenging.
This can make out-of-process unwinding difficult.

## Armv8.3 Pointer Authentication

Also available as Armv8.1-M Pointer Authentication.

Instructions are provided to sign a pointer with a 64-bit user-chosen context value (usually zero, X16, or SP) and a 128-bit secret key.
The computed Pointer Authentication Code is stored in the unused high bits of the pointer.
The instructions are allocated from the HINT space for compatibility with older CPUs.

A major use case is to sign/authenticate the return address.
`paciasp` is inserted at the start of the function prologue to sign the to-be-saved LR (X30). (It serves as an implicit `bti c` as well.)
`autiasp` is inserted at the end of the function prologue to authenticate the loaded LR.

```asm
paciasp                        # instrumented
sub     sp, sp, #0x20
stp     x29, x30, [sp, #0x10]
...
ldp     x29, x30, [sp, #0x10]
add     sp, sp, #0x20
autiasp                        # instrumented
ret
```

ld.lld added support in <https://reviews.llvm.org/D62609> (with substantial changes afterwards).
If `-z pac-plt` is specified, `autia1716` is used for a PLT entry; a relocatable file with `.note.gnu.property` and the `GNU_PROPERTY_AARCH64_FEATURE_1_PAC` bit cleared gets a warning.

FreeBSD has supported Pointer Authentication in a number of patches, e.g. [D31261](https://reviews.freebsd.org/D31261), [D42226](https://reviews.freebsd.org/D42226).
As of 2023-10, the AArch64 kernel [uses PAC and BTI by default](https://reviews.freebsd.org/D42079).

---

Let's discuss forward-edge CFI schemes.

Ideally, for an indirect call, we would want to enforce that the target is among the targets in the control-flow graph.
However, this is difficult to implement efficiently, so many schemes only ensure that the function signature matches.
For example, in the code `void f(void (*indirect)(void)) { indirect(); }`, if `indirect` actually refers to a function with arguments, the CFI scheme will flag this indirect call.

In C, checking that the argument and return types match suffices. Many schemes use type hashes.
C11 6.5.2.2p9 says

> If the function is defined with a type that is not compatible with the type (of the expression) pointed to by the expression that denotes the called function, the behavior is undefined.

In C++, language constructs such as virtual functions and pointers to member functions add more restriction to what can be indirectly called.
`-fsanitize=cfi` has implemented many C++ specific checks that are missing from many other CFI schemes.
C++ [expr.call] says

> Calling a function through an expression whose function type E is different from the function type F of the called function's definition results in undefined behavior unless the type “pointer to F” can be converted to the type “pointer to E” via a function pointer conversion ([conv.fctptr]).

C++ has a stricter requirement while C allows some minor mismatches due to the function type compatibility rules.

```c
void callee0(int (*a)[]) {}
void callee1(int (*a)[1]) {}
void f(void (*fp)(int (*)[])) { fp(0); }
int main() {
  int a[1];
  f(callee0);
  f(callee1); // compatible but flagged by -fsanitize=function (after D148827), -fsanitize=kcfi, and -Wincompatible-function-pointer-types-strict
}
```

## `pax-future.txt`

<https://pax.grsecurity.net/docs/pax-future.txt> (c.2) describes a scheme where

* a call site provides a hash indicating the intended function prototype
* the callee's epilogue checks whether the hash matches its own prototype

In case of a mismatch, it indicates that the callee does not have the intended prototype.
The program should terminate.

```asm
callee
epilogue:     mov register,[esp]
              cmp [register+1],MAGIC
              jnz .1
              retn
          .1: jmp esp

caller:
              call callee
              test eax,MAGIC
```

The epilogue assumes that a call site hash is present, so an instrumented function cannot be called by a non-instrumented call site.

## `-fsanitize=cfi`

See <https://clang.llvm.org/docs/ControlFlowIntegrity.html>.
Clang has implemented the traditional indirect function call check and many C++ specific checks under this umbrella option.
These checks all rely on [Type Metadata](https://llvm.org/docs/TypeMetadata.html) and link-time optimizations: LTO collects functions with the same signature, enabling efficient type checks.

The underlying LLVM IR technique is shared with virtual function call optimization (devirtualization).

```cpp
struct A { void f(); };
struct B { void f(); };
void A::f() {}
void B::f() {}
static void (A::*fptr)();

int main(int argc, char **argv) {
  A *a = new A;
  if (argv[1]) fptr = (void (A::*)())&B::f; // caught by -fsanitize=cfi-mfcall
  else fptr = &A::f; // good
  (a->*fptr)();
  delete a;
}
```

```cpp
struct A { virtual void f() {} };
struct B : A { void g() {} virtual ~B() {} };
struct C : A { void g() {} };
struct D { virtual void f() {} };
void af(A *a) { a->f(); }
void bg(B *b) { b->g(); }
int main(int argc, char **argv) {
  B b;
  C c;
  D d;
  af(reinterpret_cast<A *>(&d)); // caught by -fsanitize=cfi-vcall
  bg(&b); // good
  bg(reinterpret_cast<B *>(&c)); // caught by -fsanitize=cfi-nvcall
}
```

For virtual calls, the related virtual tables are placed together in the object file. This requires LTO.
After loading the dynamic type of an object, `-fsanitize=cfi` can efficiently check whether the type matches an address point of possible virtual tables.

```asm
	.section	.data.rel.ro._ZTV1D,"aGw",@progbits,_ZTV1D,comdat
	.p2align	3, 0x0
_ZTV1D:
	.quad	0; .quad	_ZTI1D; .quad	_ZN1D1fEv

	.section	.data.rel.ro..L__unnamed_1,"aw",@progbits
	.p2align	3, 0x0
.L__unnamed_1:
	.quad	0; .quad	_ZTI1B; .quad	_ZN1A1fEv; .quad	_ZN1BD2Ev; .quad	_ZN1BD0Ev
	.zero	24
	.quad	0; .quad	_ZTI1A; .quad	_ZN1A1fEv
	.zero	8
	.quad	0; .quad	_ZTI1C; .quad	_ZN1A1fEv
...

	movq	(%rdi), %rax
	leaq	__typeid__ZTS1A_global_addr(%rip), %rdx
	movq	%rax, %rcx
	subq	%rdx, %rcx
	rorq	$__typeid__ZTS1A_align, %rcx
	movq	%rcx, %rdx
	subq	$__typeid__ZTS1A_size_m1@ABS8, %rdx
	ja	.LBB0_2
	jmp	.LBB0_1
.LBB0_1:
	movl	$1, %edx
	shll	%cl, %edx
	movl	$__typeid__ZTS1A_inline_bits, %ecx
	andl	%edx, %ecx
	cmpl	$0, %ecx
	jne	.LBB0_3
.LBB0_2:	# %trap
	ud1l	2(%eax), %eax
.LBB0_3:	# %cont
	callq	*(%rax)
```

Cross-DSO CFI requires a runtime (see `compiler-rt/lib/cfi`).

## Control Flow Guard

Control Flow Guard was introduced in Windows 8.1 Preview. It was [implemented](https://reviews.llvm.org/D65761) in llvm-project in 2019.
To use it, we can run `clang-cl /guard:cf` or `clang --target=x86_64-pc-windows-gnu -mguard=cf`.

The compiler instruments indirect calls to call a global function pointer (`___guard_check_icall_fptr` or `__guard_dispatch_icall_fptr`) and records valid indirect call targets in special sections: `.gfids$y` (address-taken functions), `.giats$y` (address-taken IAT entries), `.gljmp$y` (longjmp targets), and `.gehcont$y` (ehcont targets).
An instrumented file with applicable functions defines the `@feat.00` symbol with at least one bit of 0x4800.

The linker combines the sections, marks additional symbols (e.g. `/entry`), creates address tables (`__guard_fids_table`, `__guard_iat_table`, `__guard_longjmp_table` (unless `/guard:nolongjmp`), `__guard_eh_cont_table`).

At run-time, the global function pointer refers to a function that verifies that an indirect call target is valid.

```asm
leaq   target(%rip), %rax
callq  *%rax

=>

leaq   target(%rip), %rax
callq  *__guard_dispatch_icall_fptr(%rip)
```

## eXtended Flow Guard

This is an improved Control Flow Guard that is similar to `pax-future.txt` (c.2). To use it, run `cl.exe /guard:xfg`.
On x86-64, the instrumentation calls the function pointer `__guard_xfg_dispatch_icall_fptr` instead of `__guard_dispatch_icall_fptr`.
It also provides the prototype hash as an argument.
The runtime checks whether the prototype hash matches the callee.

```c
void bar(int a) {}
void foo(void (*f)(int)) { f(42); }
```

```asm
gxfg$y  SEGMENT
__guard_xfg?foo@@YAXP6AXH@Z@Z DDSymXIndex:  FLAT:?foo@@YAXP6AXH@Z@Z
        DD      00H
        DQ      ba49be2b36da9170H
gxfg$y  ENDS
;       COMDAT gxfg$y
gxfg$y  SEGMENT
__guard_xfg?bar@@YAXH@Z DDSymXIndex:  FLAT:?bar@@YAXH@Z
        DD      00H
        DQ      c6c1864950d77370H
gxfg$y  ENDS

?bar@@YAXH@Z PROC ; bar, COMDAT
        ret     0
?bar@@YAXH@Z ENDP

?foo@@YAXP6AXH@Z@Z PROC ; foo, COMDAT
  mov rax, rcx
  mov r10, -4124868134247632016 ; c6c1864950d77370H
  mov ecx, 42 ; 0000002aH
  rex_jmp QWORD PTR __guard_xfg_dispatch_icall_fptr
?foo@@YAXP6AXH@Z@Z ENDP ; foo
```

## `-fsanitize=function`

This option instructs Clang to check indirect calls through a function pointer of the wrong type, using an approach similar to [pax-future.txt](#pax-future.txt).

```sh
% echo 'void f() {} int main() { ((void (*)(int))f)(42); }' > a.cc
% clang++ -g1 -fsanitize=function a.cc
% ./a.out
a.cc:1:26: runtime error: call to function f() through pointer to incorrect function type 'void (*)(int)'
/tmp/c/a.cc:1: note: f() defined here
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior a.cc:1:26 in
% clang++ -fsanitize=function -fsanitize-trap=function a.cc
% ./a.out
[1]    2566898 illegal hardware instruction  ./a.out
```

The feature is part of [UndefinedBehaviorSanitizer](/blog/2023-01-29-all-about-undefined-behavior-sanitizer), and many general UBSan features such as `-fsanitize-trap=` and `-fno-sanitize-recover=` can be used.
`-fsanitize=undefined` implies `-fsanitize=function`.
Since [Clang 17](https://reviews.llvm.org/D148785), we can specify `-fno-sanitize-recover=function` to let Clang exit the program upon a UBSan error (with error code 1).

`-fsanitize=function` instruments applicable functions (non-member functions and static member functions) and function pointer call sites.

For instrumented functions, clangCodeGen sets the [`func_sanitize` metadata](https://llvm.org/docs/LangRef.html#func-sanitize-metadata).
LLVMCodeGen adds two words after the function entry: a magic integer (signature) and a type hash.

For instrumented function pointer call sites (pointers to member functions are excluded), clangCodeGen inserts load and type hash checking instructions.

Let's say the `main` function has an indirect call to `f`, we may get the following assembly.
```asm
_Z1fv:
  .long   3238382334                      # 0xc105cafe
  .long   2772461324                      # type hash
  ...

main:
  ...
  # Load the first word from the function pointer and check whether it is a signature.
  cmpl    $-1056584962, _Z1fv-8(%rip)     # imm = 0xC105CAFE
  # If not, the function may be uninstrumented. Skip.
  jne     .LBB2_3
  # Load the second word and dereference it.
  cmpl    $27004076, _Z1fv-4(%rip)        # imm = 0x19C0CAC
  # Does it match the type hash that the call site expects?
  jne     .LBB2_2
  # We are good. Make an indirect call
.LBB2_3:
  xorl    %edi, %edi
  jmpq    *%rax

  # Fail
  ...
  callq   __ubsan_handle_function_type_mismatch@PLT
```

Every instrumented function is expected to be preceded by the magic integer (signature).
If the callee pointer is not preceded by the signature, the call site is skipped for type checking.
Here is an example that an uninstrumented callee is skipped for type checking.
```sh
echo 'void f(); int main() { ((void (*)(int))f)(42); }' > a.cc
echo 'void f() {}' > b.cc
clang++ -c -g1 -fsanitize=function a.cc && clang++ -c -g1 b.cc
clang++ -fsanitize=function a.o b.o
./a.out  # no error
```

The memory load makes `-fsanitize=function` inapplicable to `-mexecute-only=` targets.

Before Clang 17, the code sequence was quite different.
First, `-fsanitize=function` needed typeinfo objects (`_ZTI*`) from C++ RTTI and were therefore C++ specific (changed by my <https://reviews.llvm.org/D148785> and <https://reviews.llvm.org/D148827>).
Second, the instrumentation placed two words after the function entry, one of which was an unconditional jump instruction skipping the two words/instructions (changed by my <https://reviews.llvm.org/D148665>).

The old code sequence looks like this:
```asm
_Z1fv:
  .long   846595819                    # jmp
  .long   .L__llvm_rtti_proxy-_Z3funv
  ...

main:
  ...
  # Load the first word from the function pointer and check whether it is a signature.
  cmpl    $846595819, (%rdi)
  # If not, the function may be uninstrumented. Skip.
  jne     .LBB1_3
  # Load the second word and dereference it.
  movslq  4(%rsi), %rax
  movq    (%rax,%rsi), %rdx
  # Is it the desired typeinfo object?
  leaq    _ZTIFvvE(%rip), %rax
  # If not, fail
  cmpq    %rax, %rdx
  jne     .LBB1_2
  ...

.section        .data.rel.ro,"aw",@progbits
  .p2align        3, 0x0
.L__llvm_rtti_proxy:
  .quad   _ZTIFvvE
```

The two words at the function entry are incompatible with Intel Indirect Branch Tracking (IBT) and Armv8.5 Branch Target Identification.

Notes: 

<https://github.com/llvm/llvm-project/commit/b453cd64a79cc7cda87fadcde8aba5fd6ca9e43d> added the initial compiler support.
[No -fsanitize=function warning when calling noexcept function through non-noexcept pointer in C++17](https://github.com/llvm/llvm-project/commit/8c85bca5a5db8e84891d1cc849efb0773bddf71c) removed exception specifiers when generating a typeinfo object.

<https://github.com/llvm/llvm-project/commit/5745eccef54ddd3caca278d1d292a88b2281528b> and <https://github.com/llvm/llvm-project/commit/e215996a2932ed7c472f4e94dc4345b30fd0c373> added support for non-unique typeinfo objects.

## `-fsanitize=kcfi`

Introduced to llvm-project in [2022-11](https://reviews.llvm.org/D119296) (milestone: 16.0.0. Glad as a reviewer).
GCC [feature request](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=107048).

The instrumentation does the following:

* Stores a hash of the function prototype before the function entry.
* Loads the hash at an indirect call site and traps if it does not match the expected prototype.
* Records the trap location in a section named `.kcfi_traps`. The Linux kernel uses the section to check whether a trap is caused by KCFI.
* Defines a weak absolute symbol `__kcfi_typeid_<func>` if the function has a C identifier name and is address-taken.

```asm
__cfi_bar:
	.rept 11; nop; .endr
	movl	$27004076, %eax                 # imm = 0x19C0CAC
bar:
	retq

__cfi_foo:
	.rept 11; nop; .endr
	movl	$2992198919, %eax               # imm = 0xB2595507
foo:
	movq	%rdi, %rax
	movl	$42, %edi
	movl	$4267963220, %r10d              # imm = 0xFE63F354
	addl	-4(%rax), %r10d
	je	.Ltmp0
.Ltmp1:
	ud2
	.section	.kcfi_traps,"ao",@progbits,.text
.Ltmp2:
	.long	.Ltmp1-.Ltmp2
	.text
.Ltmp0:
	jmpq	*%rax                           # TAILCALL
```

The large number of NOPs is to leave room for FineIBT patching at run-time.

An instrumented indirect call site cannot call an uninstrumented target.
This property appears to be satisfied in the Linux kernel.

See [cfi: Switch to -fsanitize=kcfi](https://git.kernel.org/linus/89245600941e4e0f87d77f60ee269b5e61ef4e49) for the Linux kernel implementation.
The kernel expects a specific conditional branch instruction and a specific trap instruction.

### `-fsanitize=kcfi` implementation notes

To emit `.kcfi_traps` and a preferred trap instruction, the instrumentation cannot be done purely in ClangCodeGen. Instead, the following scheme is used:

* `clang::CodeGen::CodeGenModule::SetFunctionAttributes` sets the KCFI type id for an instrumented function.
* `clang::CodeGen::CodeGenFunction::EmitCall` emits "kcfi" [operand bundles](https://llvm.org/docs/LangRef.html#kcfi-operand-bundles).
* `SelectionDAGBuilder::visitCall` visits the indirect `CallInst` and builds a `SDNode` using `XXXTargetLowering::LowerCall`.
* In a target-overridden `TargetPassConfig::addPreSched`, add a KCFI pass to emit a `KCFI_CHECK` target-specific pseudo instruction before an indirect call instruction.
* In a target AsmPrinter, lower `KCFI_CHECK` pseudo instructions to a code sequence that crash if the type hashes don't match.

If a target doesn't implement `KCFI_CHECK` lowering, `llvm/lib/Transforms/Instrumentation/KCFI.cpp` is used to replace "kcfi" operand bundles with if conditions, then no `TargetPassConfig::addPreSched` or `KCFI_CHECK` lowering will be needed.
In this case, `llvm.debugtrap` is used to generate trap instructions.

`KCFI_CHECK` lowering has some complexity to allocate a temporary register. This needs to following the calling convention which can be modified by many compiler options and function attributes.

## FineIBT

This is a hybrid scheme that combines instrumentation and hardware features. It uses both an indirect branch target indicator (ENDBR) and a prototype hash check.
For an analysis, see <https://dustri.org/b/paper-notes-fineibt.html>.

The compromise `testb 0x11, fs:0x48` can be disabled at run-time.
`fs:0x48` utilizes an unused field `tcbhead_t:unused_vgetcpu_cache` in `sysdeps/x86_64/nptl/tls.h`.
This allows an attacker to disable FineIBT by zeroing this byte.

[x86/ibt: Implement FineIBT](https://git.kernel.org/linus/89245600941e4e0f87d77f60ee269b5e61ef4e49) is the Linux kernel implementation which patches `-fsanitize=kcfi` output into FineIBT.

## Hardware-assisted

### Intel Indirect Branch Tracking

This is part of Intel Control-flow Enforcement Technology.
When enabled, the CPU guarantees that every indirect branch lands on a special instruction (ENDBR, `endbr32` for x86-32 and `endbr64` for x86-64).
In case of an violation, a control-protection (#CP) exception will be raised.
A jump/call instruction with the `notrack` prefix skips the check.

With `-fcf-protection={branch,full}`, the compiler inserts ENDBR at the start of a basic block which may be reached indirectly.
This is conservative:

* every address-taken basic block needs ENDBR
* every non-internal-linkage function needs `endbr` as it may be reached via PLT
* every function compiled for the large code model needs ENDBR as it may be reached via a large code model branch
* landing pads for exception handling need ENDBR

`__attribute__((nocf_check))` can disable ENDBR insertion for a function.

GCC defaults to `-mno-cet-switch`. When jump tables are enabled (default), a switch statement with many cases compiles to a `NOTRACK` indirect jump and case handlers without `ENDBR`.
The `NOTRACK` looks scary, but it is difficult to hijack the control flow to the indirect jump bypassing the bound checking.
`-mcet-switch` generates a tracked indirect jump and case handlers with `ENDBR`, increasing the number of gadgets. Whether they can be usefully exploited depends on the program.
However, jumping to the middle of a function is intuitively more useful than jumping to the entry.
`-m[no-]cet-switch` is a trade-off. Users who want to avoid both `NOTRACK` and `ENDBR` can specify `-fno-jump-tables` to disable jump tables.

ld.lld added support in [2020-01](https://reviews.llvm.org/D59780):

* PLT entries need `endbr`
* If all relocatable files contain `.note.gnu.property` (which contains `NT_GNU_PROPERTY_TYPE_0` notes) with the `GNU_PROPERTY_X86_FEATURE_1_IBT` bit set, or `-z force-ibt` is specified, the output file will have the bit and synthesize ENDBR compatible PLT entries.

If a relocatable file does not contain `.note.gnu.property` (e.g. compiled without the relevant option or assembled from an assembly file without the section), the linker will not mark the output as compatible with the feature.
You can specify `-z force-ibt`, but you will get a warning.

The GNU ld implementation uses a second PLT scheme (`.plt.sec`). I was sad about it, but in the end I decided to follow suit for ld.lld.
Some tools (e.g. objdump) use heuristics to detect `foo@plt` and using an alternative form will require them to adapt.
mold nevertheless settled on its own scheme.

`swapcontext` may return with an indirect branch. This function can be annotated with `__attribute__((indirect_branch))` so that the call site will be followed by an `endbr`.

glibc can be built with `--enable-cet`. By default it uses the `NT_GNU_PROPERTY_TYPE_0` notes to decide whether to enable CET.
If any of the executable and all shared objects doesn't have the `GNU_PROPERTY_X86_FEATURE_1_IBT` bit, rtld will call `arch_prctl` with `ARCH_CET_DISABLE` to disable IBT. SHSTK is similar.
A dlopen'ed shared object triggers the check as well.

As of 2023-01, the Linux kernel support is work in progress. It does not support `ARCH_CET_STATUS`/`ARCH_CET_DISABLE` yet.

qemu does not support Indirect Branch Tracking yet.

In 2023-10, mold added an experiment `-z rewrite-endbr` to change some `endbr` to `nop`.
I think a safe approach is to tag compiler-generated ENDBR with a new relocation type.

#### Don't conservatively mark non-local-linkage functions

The current scheme as well as Arm BTI conservatively adds an indirect branch indicator to every non-internal-linkage function, which increases the attack surface.
[Alternative CET ABI](https://groups.google.com/g/x86-64-abi/c/iQWEW-iW8DQ) mentioned that we can use `NOTRACK` in PLT entries, and use new relocation types to indicate address-significant functions.
Supporting `dlsym`/`dlclose` needs special tweaking.

Technically the address significance part can be achieved with the existing `SHT_LLVM_ADDRSIG`, then we can avoid introducing a new relocation type for each architecture.
I am somewhat unhappy that `SHT_LLVM_ADDRSIG` optimizes for size and does not make binary manipulation convenient (`objcopy` and `ld -r` sets `sh_link=0` of `SHT_LLVM_ADDRSIG` and invalidates the section).
See [Explain GNU style linker options](/blog/2020-11-15-explain-gnu-linker-options).
Since we just deal with x86, introducing new x86-32/x86-64 relocation types is not bad.

When LTO is concerned, this imposes more difficulties as LLVMCodeGen does not know (<https://reviews.llvm.org/D140363>):

* whether a function is visibile to a native relocatable file (which may take the address) (`VisibleToRegularObj`)
* or whether the address of a function is taken in at least one IR module.

`F.hasAddressTaken()` does not provide the necessary information:

* If the link mixes bitcode files and ELF relocatable files, for a function in a bitcode file, `F.hasAddressTaken()` doesn't indicate that its address is not taken by an ELF relocatable file.
* For ThinLTO, a function may have false `F.hasAddressTaken()` for the definition in one module and true `F.hasAddressTaken()` for a reference in another module.

In theory we can instruct LTO to compute the address-taken property (which aligns closely to `!GV.use_empty() && !GV.hasAtLeastLocalUnnamedAddr()`), and provide `VisibleToRegularObj` to LLVMCodeGen.
Then we can determine the necessity of ENDBR: `AddressTaken || (!F.hasLocalLinkage() && (VisibleToRegularObj || ExportDynamic))`.

Considering the large code model and potential inclusion of range extension thunks, a function might still be indirectly jumped to even if the preceding condition evaluates to false.
As a precaution, we'll need to identify and label such call sites with the NOTRACK prefix, if we decide 

### Armv8.5 Branch Target Identification

This is similar to Intel Indirect Branch Tracking.
When enabled, the CPU ensures that every indirect branch lands on a special instruction, otherwise a Branch Target exception is raised.
The most common landing instructions are `bti {c,j,jc}` with different branch instruction compatibility.
`paciasp` and `pacibsp` are implicitly `bti c`.

This feature is more fine-grained: every memory page may set a bit `VM_ARM64_BTI` (set with the mmap flag `PROT_BTI`) to indicate that whether indirect calls from it need protection.
This allows uninstrumented code to mapped simply by skipping `PROT_BTI`.
When the Linux kernel loads an executable `PT_LOAD` segment, the memory map gets the `PROT_BTI` bit.
The BTI-enabled maps can be identified as `VmFlags:.*bt` in `/proc/self/smaps`.

The value of `-mbranch-protection=` can be `none` (no hardening), `standard` (`bti` with a non-leaf protection scope), or `+` separated `bti` and `pac-ret[+b-key,+leaf]`.
If `bti` is enabled, the compiler inserts `bti {c,j,jc}` at the start of a basic block which may be reached indirectly.
For a non-internal-linkage function, its entry may be reached by a PLT or range extension thunk, so it is conservatively marked as needing `bti c`.

ld.lld added support in <https://reviews.llvm.org/D62609> (with substantial changes afterwards).
If all relocatable files with `.note.gnu.property` have set the `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` bit, or `-z force-bti` is specified, the output will have the bit and synthesize BTI compatible PLT entries.

For a PLT entry, the landing instruction for a function can be elided in many cases as it cannot be an indirect branch target.
Potential indirect branch target cases include: canonical PLT entries, non-preemptible ifunc, range extension thunks, and `PLT32` data relocations.
Technically, a range extension thunk can be changes from `caller => thunk =indirect=> callee` to `caller => thunk1 =indirect=> thunk2 => callee`, but this may introduce lots of linker complexity and increase the risk some cases wrong.
<https://github.com/ARM-software/abi-aa/issues/196> tracks documentation update about when a BTI can be elided.

qemu has supported Branch Target Identification since 2019.

---

Arm v8-m also supports Branch Target Identification, which is defined in terms of BTI setting and BTI clearing instructions, which set EPSR.B to one or zero.
When EPSR.B is one, the next executed instruction must be a BTI clearing instruction otherwise an INVSTATE UsageFault is generated.
BTI setting instructions [are](https://reviews.llvm.org/D155485#4534547):

```
* BLX.
* BLXNS.
* When the register holding the branch address is not the LR:
– BX.
– BXNS.
* When the address is loaded into the PC:
– LDR (register).
– LDR (literal).
* When the address is loaded into the PC and the base address register is either not the SP or the SP and write-back of the SP does not occur:
– LDR (immediate).
– LDM, LDMIA, LDMFD.
– LDMDB, LDMEA.
```

## Compiler warnings

`clang -Wcast-function-type` warns when a function pointer is cast to an incompatible function pointer.
Calling such a cast function pointer likely leads to `-fsanitize=cfi`/`-fsanitize=kcfi` runtime errors.
For practical reasons, GCC made a choice to allow some common violations.

Clang 16.0.0 makes `-Wcast-function-type` [stricter](https://reviews.llvm.org/D134831) and warns more cases.
`-Wno-cast-function-type-strict` restores the previous state that ignores many cases including some ABI equivalent cases.

## Summary

Forward-edge CFI

* check that the target may be landed indirectly but does not check the function signature: Control Flow Guard, Intel Indirect Branch Tracking, Armv8.5 Branch Target Identification
* check that the target may be landed indirectly and check the function signature: eXtended Flow Guard, `-fsanitize=cfi`, `-fsanitize=kcfi`, FineIBT

Links:

* RISC-V CFI: <https://github.com/riscv/riscv-cfi>
