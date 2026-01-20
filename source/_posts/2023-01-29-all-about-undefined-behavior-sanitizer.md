---
layout: post
title: All about UndefinedBehaviorSanitizer
author: MaskRay
tags: [llvm,sanitizer]
---

Updated in 2025-01.

[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) (UBSan) is an undefined behavior detector for C/C++.
It consists of code instrumentation and a runtime. Both components have multiple independent implementations.

Clang [implemented](https://github.com/llvm/llvm-project/commit/a3e2ff19e59c587ea14611eb64400c6d0ecabcee) the first few checks in 2009-12, initially named `-fcatch-undefined-behavior`.
In 2012 `-fsanitize=undefined` was added and `-fcatch-undefined-behavior` was [removed](https://github.com/llvm/llvm-project/commit/b1b0ab41e79f4f11ab21e6e56ded7147241f8615).
GCC 4.9 [implemented](de5a5fa1395db2cb5da4d0593fef40ec22378576) `-fsanitize=undefined` in 2013-08.

The runtime used by Clang lives in `llvm-project/compiler-rt/lib/ubsan`.
GCC from time to time syncs its downstream fork of the sanitizers part of compiler-rt (`libsanitizer`).
The end of the article lists some alternative runtime implementations.

<!-- more -->

## Available checks

There are many undefined behavior [checks](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) which can be detected.
One can specify `-fsanitize=xxx,yyy` where xxx and yyy are the names of the individual checks, or more commonly, specify `-fsanitize=undefined` to enable all of them.

Some checks are not undefined behavior per language standards, but are often code smell and lead to surprising results.
They are `implicit-unsigned-integer-truncation`, `implicit-integer-sign-change`, `unsigned-integer-overflow`, etc.

We can use `-###` to get the list of default UBSan checks.
```
% clang -fsanitize=undefined -xc /dev/null '-###'
... -fsanitize=alignment,array-bounds,bool,builtin,enum,float-cast-overflow,function,integer-divide-by-zero,nonnull-attribute,null,pointer-overflow,return,returns-nonnull-attribute,shift-base,shift-exponent,signed-integer-overflow,unreachable,vla-bound,vptr" "-fsanitize-recover=alignment,array-bounds,bool,builtin,enum,float-cast-overflow,function,integer-divide-by-zero,nonnull-attribute,null,pointer-overflow,returns-nonnull-attribute,shift-base,shift-exponent,signed-integer-overflow,vla-bound,vptr" ...
```

GCC implements [slightly fewer](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html##:~:text=fsanitize=undefined) checks than Clang.

If any check in the "undefined" group is enabled, `#if __has_feature(undefined_behavior_sanitizer)` will apply.

## Modes

UBSan provides 3 modes to allow tradeoffs between code size and diagnostic verboseness.

* Default mode
* Minimal runtime (`-fsanitize-minimal-runtime`)
* Trap mode (`-fsanitize-trap=undefined`): no runtime

Let's use the following program to compare the instrumentation and runtime behaviors.
```c
#include <stdio.h>
int foo(int a) { return a * 2; }
int main() {
  int x;
  scanf("%d", &x);
  printf("a %d\n", foo(x));
  printf("b %d\n", foo(x));
}
```

### Default mode

With the umbrella option `-fsanitize=undefined` or the specific `-fsanitize=signed-integer-overflow`, Clang emits the following LLVM IR for `foo`:

```llvm
define dso_local i32 @foo(i32 noundef %a) local_unnamed_addr #0 {
entry:
  %0 = add i32 %a, 1073741824
  %1 = icmp sgt i32 %0, -1
  br i1 %1, label %cont, label %handler.mul_overflow, !prof !4, !nosanitize !5

handler.mul_overflow:                             ; preds = %entry
  %2 = zext i32 %a to i64, !nosanitize !5
  tail call void @__ubsan_handle_mul_overflow(ptr nonnull @1, i64 %2, i64 2) #3, !nosanitize !5
  br label %cont, !nosanitize !5

cont:                                             ; preds = %handler.mul_overflow, %entry
  %3 = shl i32 %a, 1
  ret i32 %3
}
```

The `add` and `icmp` instructions check whether the argument is in the range [-0x40000000, 0x40000000) (no overflow).
If yes, the `cont` branch is taken and the non-overflow result is returned.
Otherwise, the emitted code calls the callback `__ubsan_handle_mul_overflow` (implemented by the runtime) with arguments describing the source location.

By default, all UBSan checks except `return,unreachable` are recoverable (i.e. non-fatal).
After `__ubsan_handle_mul_overflow` (or another callback) prints an error, the program keeps executing.
The source location is marked and further errors about this location are [suppressed](https://github.com/llvm/llvm-project/commit/765c266892d712db91b2a1677584f09feb862109).
This deduplication feature makes logs less noisy.
In one program invocation, we may observe errors from potentially multiple source locations.

We can let Clang exit the program upon a UBSan error (with error code 1, customized by `UBSAN_OPTIONS=exitcode=2`).
Just specify `-fno-sanitize-recover` (alias for `-fno-sanitize-recover=all`), `-fno-sanitize-recover=undefined`, or `-fno-sanitize-recover=signed-integer-overflow`.
A non-return callback will be emitted in place of `__ubsan_handle_mul_overflow`.
Note that the emitted LLVM IR terminates the basic block with an `unreachable` instruction.
```llvm
handler.mul_overflow:                             ; preds = %entry
  %2 = zext i32 %a to i64, !nosanitize !5
  tail call void @__ubsan_handle_mul_overflow_abort(ptr nonnull @1, i64 %2, i64 2) #4, !nosanitize !5
  unreachable, !nosanitize !5
```

#### Linking

The UBSan callbacks are provided by the runtime. For a link action, we need to inform the compiler driver that the runtime should be linked.
This can be done by specifying `-fsanitize=undefined`.
(Actually, any specific UBSan check that needs runtime can be used, e.g. `-fsanitize=signed-integer-overflow`.)

```sh
clang -c -O2 -fsanitize=undefined a.c
clang -fsanitize=undefined a.o -o a
```

Some checks (`function,vptr`) call C++ specific callbacks implemented in `libclang_rt.ubsan_standalone_cxx.a`.
We need to use `clang++ -fsanitize=undefined` for the link action (or use `clang -fsanitize=undefined -fsanitize-link-c++-runtime`).

```
% clang -fsanitize=undefined a.o '-###' |& grep --color ubsan
... "--whole-archive" ".../libclang_rt.ubsan_standalone.a" "--no-whole-archive" "--dynamic-list=.../libclang_rt.ubsan_standalone.a.syms" ...
% clang++ -fsanitize=undefined a.o '-###' |& grep --color ubsan
... "--whole-archive" ".../libclang_rt.ubsan_standalone.a" "--no-whole-archive" "--dynamic-list=.../libclang_rt.ubsan_standalone.a.syms" "--whole-archive" ".../libclang_rt.ubsan_standalone_cxx.a" "--no-whole-archive" "--dynamic-list=.../libclang_rt.ubsan_standalone_cxx.a.syms" ...
% clang++ -fsanitize=undefined -shared-libsan a.o '-###' |& grep --color ubsan
... ".../libclang_rt.ubsan_standalone.so" ...
```

When linking an executable, with the default `-static-libsan` mode on many targets, Clang Driver passes `--whole-archive $resource_dir/lib/$triple/libclang_rt.ubsan.a --no-whole-archive` to the linker.
GCC and some platforms prefer shared runtime/dynamic runtime. See [All about sanitizer interceptors](/blog/2023-01-08-all-about-sanitizer-interceptors#elf-platforms).

Some sanitizers (`address`, `memory`, `thread`, etc) ship a copy of UBSan runtime files.

### Minimal runtime

The default mode provides verbose diagnostics which help programmers identify the undefined behavior.
On the other hand, the detailed log helps attackers and the code size may be a concern in some configurations.

UBSan provides a minimal runtime mode to log very little information.
Specify `-fsanitize-minimal-runtime` for both compile actions and link actions to enable the mode.

```
% clang -fsanitize=undefined -fsanitize-minimal-runtime a.c -o a
% ./a <<< 1073741824
ubsan: mul-overflow by 0x000055565bab6839
a -2147483648
b -2147483648
% clang -fsanitize=undefined -fno-sanitize-recover -fsanitize-minimal-runtime a.c -o a
% ./a <<< 1073741824
ubsan: mul-overflow by 0x000056443f9a7839
[1]    3857513 IOT instruction  ./a <<< 1073741824
```

In the emitted LLVM IR, a different set of callbacks `__ubsan_handle_*_minimal` are used.
They take no argument and therefore make instrumented code much smaller.

```llvm
handler.mul_overflow:                             ; preds = %entry
  tail call void @__ubsan_handle_mul_overflow_minimal() #4, !nosanitize !6
  br label %cont, !nosanitize !6
```

The UBSan minimal runtime uses a separate set of runtime files (`libclang_rt.ubsan_minimal.*`) to decrease the runtime size.
```
% clang -fsanitize=undefined -fsanitize-minimal-runtime a.c '-###' |& grep --color=auto ubsan_minimal
... "--whole-archive" "/tmp/Rel/lib/clang/17/lib/x86_64-unknown-linux-gnu/libclang_rt.ubsan_minimal.a" "--no-whole-archive" "--dynamic-list=/tmp/Rel/lib/clang/17/lib/x86_64-unknown-linux-gnu/libclang_rt.ubsan_minimal.a.syms" ...
```

### Trap mode

When `-fsanitize=signed-integer-overflow` is in effect, we can specify `-fsanitize-trap=signed-integer-overflow` so that Clang will emit a trap instruction instead of a callback.
This will greatly decrease the size bloat from instrumentation compared to the default mode.

Usually, we specify the umbrella options [`-fsanitize-trap=undefined`](https://reviews.llvm.org/D10464) or `-fsanitize-trap` (alias for `-fsanitize-trap=all`).
```sh
clang -S -emit-llvm -O2 -fsanitize=undefined -fsanitize-trap=undefined a.c
```

(The initial `-fcatch-undefined-behavior` patch in 2013 added `-fsanitize-undefined-trap-on-error`, which is now an alias for `-fsanitize-trap=undefined`.
`-fsanitize-undefined-trap-on-error` is deprecated in Clang and should not be used for new projects.
[GCC feature request](https://gcc.gnu.org/PR109489).)

Instead of calling a callback, the emitted LLVM IR will call the LLVM intrinsic `llvm.ubsantrap` which lowers to a trap instruction aborting the program.
As a side benefit of avoiding UBSan callbacks, we don't need a runtime. Therefore, `-fsanitize=undefined` can be omitted for link actions.
Note: `-fsanitize-trap=xxx` overrides `-fsanitize-recover=xxx`.

```llvm
define dso_local i32 @foo(i32 noundef %a) local_unnamed_addr #0 {
entry:
  %0 = add i32 %a, 1073741824
  %1 = icmp sgt i32 %0, -1
  br i1 %1, label %cont, label %trap, !nosanitize !5

trap:                                             ; preds = %entry
  tail call void @llvm.ubsantrap(i8 12) #4, !nosanitize !5
  unreachable, !nosanitize !5

cont:                                             ; preds = %entry
  %2 = shl nsw i32 %a, 1
  ret i32 %2
}
```

`llvm.ubsantrap` has an integer argument.
On some architectures this can change the encoding of the trap instruction with different error types.

On x86-64, `llvm.ubsantrap` lowers to the 4-byte `ud1l ubsan_type(%eax),%eax` (UD1 with an address-size override prefix; `ud1l` is the AT&T syntax).
AArch64 provides `BRK` with 16-bit immediate.

However, many other architectures do not provide sufficient encoding space for their trap instructions.
For example, PowerPC provides `trap` (alias for `tw 31,0,0`). In `tw TO,RA,RB`, when we specify `RA=RB=0`, only 4 bits of the 5-bit `TO` can be used, which is insufficient.

For AArch64 and x86-64, we can register a signal handler to disassemble the instruction at `si->si_addr`.
It can distinguish UBSan check errors from other faults. We can even decode the UBSan type numbers from [`#define LIST_SANITIZER_CHECKS`](https://github.com/llvm/llvm-project/blob/main/clang/lib/CodeGen/CodeGenFunction.h) and give a better diagnostic.

```cpp
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void handler(int signo, siginfo_t *si, void *uc) {
#ifdef __aarch64__
  if (signo == SIGTRAP && si && si->si_addr) {
    uint32_t insn;
    memcpy(&insn, si->si_addr, sizeof(insn));
    if ((insn & 0xffe0001f) == 0xd4200000 && ((insn >> 13) & 255) == 'U') {
      char buf[64];
      int len = snprintf(buf, sizeof buf, "TRAP(%u) at %p, possibly UBSan error\n", (insn>>5 & 255), si->si_addr);
      write(STDOUT_FILENO, buf, len);
    }
  }
#endif
#ifdef __x86_64__
  if (signo == SIGILL && si && si->si_code == ILL_ILLOPN && si->si_addr) {
    auto *pc = reinterpret_cast<const unsigned char *>(si->si_addr);
    if (pc[0] == 0x67 && pc[1] == 0x0f && pc[2] == 0xb9 && pc[3] == 0x40) {
      char buf[64];
      int len = snprintf(buf, sizeof buf, "UD1 at %p, possibly UBSan error\n", pc);
      write(STDOUT_FILENO, buf, len);
    }
  }
#endif
  abort();
}

int foo(int a) { return a * 2; }

int main() {
  struct sigaction sa = {};
  sa.sa_flags = SA_SIGINFO;
  sa.sa_sigaction = &handler;
#if defined(__aarch64__)
  sigaction(SIGTRAP, &sa, nullptr);
#elif defined(__x86_64__)
  sigaction(SIGILL, &sa, nullptr);
#endif

  int x;
  scanf("%d", &x);
  printf("a %d\n", foo(x));
  printf("b %d\n", foo(x));
}
```

```
% clang++ -fsanitize=undefined -fsanitize-trap=undefined a.cc -o a
% ./a <<< 1073741824
TRAP(12) at 0xaaaaea87eb48, possibly UBSan error
[1]    529341 abort (core dumped)  ./a <<< 1073741824
```

TODO: [arm64: Support Clang UBSAN trap codes for better reporting](https://lore.kernel.org/r/20230203173946.gonna.972-kees@kernel.org)

## Together with other sanitizers

UBSan can be used together with many other sanitizers.

```sh
clang -fsanitize=address,undefined a.c
clang -fsanitize=fuzzer,undefined -fno-sanitize-recover a.c
clang -fsanitize=hwaddress,undefined a.c
clang -fsanitize=memory,undefined a.c
clang -fsanitize=thread,undefined a.c
clang -fsanitize=cfi,undefined -flto -fvisibility=hidden a.c
```

Typically one may want to test multiple sanitizers for a project.
The ability to use UBSan with another sanitizer decreases the number of configurations.
In practice people prefer `-fsanitize=address,undefined` and `-fsanitize=hwaddress,undefined` over other combinations.
Of course, adding UBSan on top of another instrumentation means more overhead and size bloat.

Using UBSan with libFuzzer is interesting.
We can run `-fsanitize=fuzzer,address,undefined -fno-sanitize-recover` to make libFuzzer abort when an AddressSanitizer or UndefinedBehaviorSanitizer error is detected.
Note: original libFuzzer developers have [stopped](https://llvm.org/docs/LibFuzzer.html#status) active work on libFuzzer and switched to [Centipede](https://github.com/google/centipede).

## Runtime options

As mentioned, most UBSan checks are recoverable by default. Specify `halt_on_error=1` to get a behavior similar to compile-time `-fno-sanitize-recover=undefined`.

```
% UBSAN_OPTIONS=halt_on_error=1 ./a <<< 1073741824
a.c:2:27: runtime error: signed integer overflow: 1073741824 * 2 cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior a.c:2:27 in
```

We can define `__ubsan_default_options` to set the default options.
```cpp
extern "C" const char *__ubsan_default_options() {
  return "print_stacktrace=1";
}
```

## Issue suppression

The GNU function attribute `__attribute__((no_sanitize("undefined")))` can disable instrumentation for a function.
(GCC additionally supports a deprecated attribute `__attribute__((no_sanitize_undefined)))`.)

If `$resource_dir/share/ubsan_ignorelist.txt` is present, it will be used as the default system ignorelist (which can be overridden with `-fsanitize-system-ignorelist=`).
See [Sanitizer special case list](https://clang.llvm.org/docs/SanitizerSpecialCaseList.html) for its format.
This file instructs Clang CodeGen to disable instrumentations for specified functions or files.
One can specify `-fsanitize-ignorelist=` multiple times to use more ignorelists.

In a large code base, enabling a check entails fixing all existing issues or working around them with the `no_sanitize` function attribute.
An ignorelist offers a mechanism to incrementally enable a check.
This is useful for toolchain maintainers who need to fight with new code.

For example, once all source files except `[a-m]*` are free of UBSan errors (or suppressed with function attributes), we can use the following patterns in `ubsan_ignorelist.txt`.
```
[alignment]
src:[a-m]*
src:./[a-m]*
```

`./` is for included files.

Suppression can be done at runtime as well. UBSan supports a runtime option `suppressions`: `UBSAN_OPTIONS=suppressions=a.supp`.

## Stack traces

By default the diagnostic does not include a stack trace. Specify `print_stacktrace=1` to get one.
If the program is compiled with `-g1` or `-g`, and `llvm-symbolizer` is in a `PATH` directory, we can get a symbolized stack trace.

```
% UBSAN_OPTIONS=print_stacktrace=1 ./a <<< 1073741824
a.c:2:27: runtime error: signed integer overflow: 1073741824 * 2 cannot be represented in type 'int'
    #0 0x55f582e9d2f1 in foo /tmp/c/a.c:2:27
    #1 0x55f582e9d2f1 in main /tmp/c/a.c:6:20
    #2 0x7fe67838f189 in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
    #3 0x7fe67838f244 in __libc_start_main csu/../csu/libc-start.c:381:3
    #4 0x55f582e75d80 in _start (/tmp/c/a+0x17d80)

SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior a.c:2:27 in
a -2147483648
b -2147483648
```

Like other sanitizers, UBSan runtime supports stack unwinding with DWARF Call Frame Information and frame pointers.
Many targets enable `.eh_frame` (a variant of DWARF Call Frame Information) by default (`-fasynchronous-unwind-tables`).
If it is disabled, ensure `-fno-omit-frame-pointer` is in effect (many targets default to `-fomit-frame-pointer` with `-O1` and above).

## `-fsanitize=alignment`

Unaligned memory accesses are expensive on some architectures or will cause traps.

C11 6.3.2.3 specifies:

> An integer may be converted to any pointer type. Except as previously specified, the result is implementation-defined, might not be correctly aligned, might not point to an entity of the referenced type, and might be a trap representation.
>
> A pointer to an object type may be converted to a pointer to a different object type. If the resulting pointer is not correctly aligned for the referenced type, the behavior is undefined. ...

C++ is not as explicit. However, I believe [basic.align] paragraph 1 is sufficient:

> ... An object type imposes an alignment requirement on every object of that type; ...

In addition, C++ [expr.static.cast]p14 says of conversions from a misaligned pointer:

> A prvalue of type “pointer to cv1 void” can be converted to a prvalue of type “pointer to cv2 T”, where T is an object type and cv2 is the same cv-qualification as, or greater cv-qualification than, cv1. If the original pointer value represents the address A of a byte in memory and A does not satisfy the alignment requirement of T, then the resulting pointer value is unspecified. ...

Which is allowed to be an invalid pointer value, which the compiler is then permitted to give whatever semantics we like, such as disallowing it being passed to memcpy.

Both GCC and Clang implement `-fsanitize=alignment` generally by instrumenting memory access operations instead of places where pointers are constructed.
Here is a list.

```cpp
*p  // load
*p = 1;  // store
int &r = *p;  // bind a reference
s->member  // member access within misaligned address
new (s) S  // placement new at a misaligned address
char *__attribute__((assume_aligned(64))) x   // actually misaligned
__builtin_assume_aligned(p, 64)  // when p is misaligned
```

[RFC: Enforcing pointer type alignment in Clang](https://lists.llvm.org/pipermail/llvm-dev/2016-January/094012.html) describes Clang's decision.

If `p` is unaligned, it should be a pointer to `char`, `unsigned char`, or `std::byte` and the following code can be used.
(Newer Clang's `-fsanitize=alignment` will report a runtime failure when `p` is misaligned.)
```c
uint32_t x;
memcpy(&x, p, sizeof(uint32_t)); // Load from an unaligned pointer
memcpy(p, &x, sizeof(uint32_t)); // Store to an unaligned pointer
```

If using dedicated unaligned load/store functions is inconvenient, we can utilize a GCC extension:

```c
typedef int32_t __attribute__((aligned(1))) unaligned_int32_t;
// (C++ specific) Or: using unaligned_int32_t __attribute__((aligned(1))) = int32_t;

int32_t foo_a(int32_t *x) { return *x; } // instrumented by -fsanitize=alignment
unaligned_int32_t foo_u(unaligned_int32_t *x) { return *x; } // not instrumented
```

* <https://github.com/Blosc/c-blosc2/pull/550>
* ffmpeg has a `--disable-fast-unaligned` configure option

Notes about hardware alignment checking:

In x86, if the AM bit is set in the CR0 register and the AC bit is set in the EFLAGS register, alignment checking of user-mode data accessing is enabled.

```cpp
#include <stdio.h>
alignas(8) char a[100];
int main() {
  long b;
  asm("pushf\norq $(1<<18), (%%rsp)\npopf\n"
      "movq 0(%1), %0\n" // if 0 is changed to 1, SIGBUS due to alignment check
      "pushf\nandq $~(1<<18), (%%rsp)\npopf\n"
      : "=r"(b) : "r"(a));
  printf("%ld\n", b);
}
```

If alignment checking is enabled and lazy PLT binding is enabled, the `printf` call will trigger alignment checking exception in `movlpd (%rdi),%xmm1` (in `../sysdeps/x86_64/multiarch/../multiarch/strcmp-sse2.S`).
If `-Wl,-z,now` is specified, the program will succeed on an Intel CPU but may still fail on AMD CPU due to misaligned `movups %xmm1,0xc8(%rsp)` in `__printf_buffer`.

## `-fsanitize=enum`

C++ [dcl.enum] says:

> For an enumeration whose underlying type is fixed, the values of the enumeration are the values of the underlying type. Otherwise, the values of the enumeration are the values representable by a hypothetical integer type with minimal width M such that all enumerators can be represented. The width of the smallest bit-field large enough to hold all the values of the enumeration type is M. It is possible to define an enumeration that has values not defined by any of its enumerators. If the enumerator-list is empty, the values of the enumeration are as if the enumeration had a single enumerator with value 0.

## `-fsanitize=function`

See [Control-flow integrity -fsanitize=function](/blog/2022-12-18-control-flow-integrity#fsanitizefunction).

## Miscellaneous

In Clang, unlike many other sanitizers, UndefinedBehaviorSanitizer is performed as part of Clang CodeGen instead of a LLVM pass in the optimization pipeline.
(`-fsanitize=object-size` is special as it requires passes after clangCodeGen.)

With the default mode and the minimal runtime, UBSan instrumentation inserts callbacks. This makes the instrumented function non-leaf.

`-ftrapv` checks can be replaced with `-fsanitize=signed-integer-overflow -fsanitize-trap=signed-integer-overflow`. `-ftrapv` is more or less obsoleted.

## Selected applications

`compiler-rt/lib/ubsan` is written in C++. It uses features that are unavailable in some environments (e.g. some operating system kernels).
Adopting UBSan in kernels typically requires reimplementing the runtime (usually in C).

In the Linux kernel, [UBSAN: run-time undefined behavior sanity checker](https://git.kernel.org/linus/c6d308534aef6c99904bf5862066360ae067abc4) introduced `CONFIG_UBSAN` and a runtime implementation in 2016.

NetBSD [implemented](https://blog.netbsd.org/tnf/entry/introduction_to_%C2%B5ubsan_a_clean) µUBSan in 2018.

Android's doc: <https://source.android.com/docs/security/test/sanitizers#undefinedbehaviorsanitizer>

## SelectiveUBSan

https://discourse.llvm.org/t/rfc-add-llvm-allow-runtime-check-intrinsic/77641

Excessive instrumentation can bloat code size. A solution leverages PGO profiles to selectively remove instrumentation from frequently executed basic blocks.

When `-mllvm -ubsan-guard-checks` is specified, the inserted UBSan checks will be guarded by a [`llvm.allow.ubsan.check`](https://llvm.org/docs/LangRef.html#llvm-allow-ubsan-check-intrinsic) test.
`llvm::LowerAllowCheckPass` utilizes profile to replace `llvm.allow.ubsan.check` calls with 0 or 1, depending on the basic block hotness (cl::opt `lower-allow-check-percentile-cutoff-hot`).
`llvm::LowerAllowCheckPass` needs to be after inliner and profile matching.

Clang's `__builtin_allow_runtime_check("check")` lowers to an intrinsic call `llvm.allow.runtime.check`, which is similar to `llvm.allow.ubsan.check`.

The user-facing Clang option is `-fsanitize-skip-hot-cutoff=`.
