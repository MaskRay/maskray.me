layout: post
title: When can glibc be built with Clang?
author: MaskRay
tags: [clang,glibc]
---

In September, I wrote "So, dear glibc, will you be happy with my sending Clang patches?" in [Build glibc with LLD 13](/blog/2021-09-05-build-glibc-with-lld).
We have come to a turning point.

In Linux Plumbers Conference 2021, at the glibc Birds of a Feather session, I asked the Clang buildability question to the glibc stewards.
(Interlude: I did not realize that I should attend the conference (it was a great opportunity from an outlier to meet some glibc folks).
In Tuesday, Wei Wu (lazyparser) kindly gave me his account: "想去参加LPC么？我会议太多了今天参加不过来".
I happily accepted it and typed the question during the glibc session.)

So I got positive responses.
"Carlos: Yes, we could be happy with clang buildability." "Joseph: Patches should be split into logical changes."
This is really great news!
My unnesting patch had sat there for a while and I was unsure about the Clang buildability interest.

<!-- more -->

Of course, every change needs to be assessed where they belong to, LLVM/Clang or glibc.

My interest in Clang/LLD buildability did not arise suddenly. It has been multiple years.
I made my first patch improving LLD's glibc buildability in June 2018.
Probably since June 2019, I pushed first LLVM fix mproving glibc buildability
[`[MC][ELF] Don't create relocations with section symbols for STB_LOCAL ifunc`](https://reviews.llvm.org/rGc841b9abf039ec0457752cd96f7e4716c1c7a323).
I have sporadic contribution since then.
Today I think there is not too much to improve on LLVM/Clang's side, but many need glibc adapatation.

As a rough estimate, we may need 30 patches to fix glibc build on x86-64 for the `--disable-werror` configuration.
Fixing other popular architectures or all tests will take more.
Adrian Ratiu mentioned that [shebs/clangify](https://sourceware.org/git/?p=glibc.git;a=shortlog;h=refs/heads/shebs/clangify) exists in [A tale of two toolchains and glibc](https://www.collabora.com/news-and-blog/blog/2021/09/30/a-tale-of-two-toolchains-and-glibc/).
The branch in the 2018 era has about 30 commits.

---

Below I will introduce the main challenges (GCC nested functions, asm label after first use, Clang's less permissive warnings) and a selective set of miscellaneous changes.

## GCC nested functions

glibc's dynamic loader (often called ld.so) implementation (`elf/` plus arch-specific code in `sysdeps/*/`) made extensive use of a feature called "nested functions".
<https://gcc.gnu.org/onlinedocs/gcc/Nested-Functions.html> says

> A nested function is a function defined inside another function. Nested functions are supported as an extension in GNU C, but are not supported by GNU C++.

This is a no-go for Clang which does not support this feature (`error: function definition is not allowed here`).
The dynamic loader usage actually impacted readability and debuggability, so people agreed that the usage should simply be removed.

As a worst example, the dynamic loader had a function like:
```c
#ifndef RESOLVE_MAP
static
#else
auto
#endif
inline void __attribute__ ((unused, always_inline))
elf_get_dynamic_info (struct link_map *l)
{
```

Its file could be included twice, one with `RESOLVE_MAP` defined and one without.
It was difficult to understand the code intention.

As of 2021-10-07, the usage in the dynamic loader has been removed by my [`elf: Avoid nested functions in the loader [BZ #27220]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=490e6c62aa31a8aa5c4a059f6e646ede121edf0a).
Adhemerval Zanella fixed some fallout.

The glibc dynamic loader code is extremely tricky. I think not taking inspiration from NetBSD code in 1992 was unfortunate.

Remaining usage include:

* `nss/makedb.c:add_key` (fixed by [`nss: Unnest nested function add_key`](https://sourceware.org/git/?p=glibc.git;a=commit;h=53d19edf7b7ab506b510c9c879a575c8484d075f))
* `posix/regcomp.c:{seek_collating_symbol_entry,...}` (fixed by [`regex: Unnest nested functions in regcomp.c`](https://sourceware.org/git/?p=glibc.git;a=commit;h=fdcd177fd36c60ddc9cbc6013831413dbd83c3f9))

## asm label after first use

In GCC and Clang, an asm label can change the symbol name for a definition or a non-definition declaration.
glibc makes heavy use of this feature even in public headers.
In some places it is definitely a code smell problem. Function attributes are added to one function in multiple places while adding them when the first declaration would be better.

`#pragma redefine_extname` is a similar feature which can change the symbol name.
In Clang, `#pragma redefine_extname` is converted to an asm label and shares code with the asm label implementation.

Clang before 12.0.0 had an issue that a built-in function ignored the asm label.
I fixed it and made some notes on <https://maskray.me/blog/2020-10-15-intra-call-and-libc-symbol-renaming> (TODO: translate it into English).

```c
typedef unsigned long size_t;

#ifdef ASM_LABEL
extern void *memcpy(void *, const void *, size_t) asm("__GI_memcpy");
#else
#pragma redefine_extname memcpy __GI_memcpy // before the first use, works
extern void *memcpy(void *, const void *, size_t);
#endif

void *test_memcpy(void *dst, const void *src, size_t n) {
  // Clang < 12:  callq   memcpy
  // Clang >= 12: callq   __GI_memcpy
  return memcpy(dst, src, n);
}
```

Unfortunately, there is a limitation. If a `#pragma redefine_extname` is declared after the first use of the symbol, the rename operation will be silently ignored.

```c
// a.c
typedef unsigned long size_t;

extern void *memcpy(void *, const void *, size_t);

void *test_memcpy(void *dst, const void *src, size_t n) {
  return memcpy(dst, src, n);
}

#ifdef ASM_LABEL
extern void *memcpy(void *, const void *, size_t) asm("__GI_memcpy");
#else
#pragma redefine_extname memcpy __GI_memcpy // before the first use, works
#endif
```

If an asm label applies to a redeclaration after the first use, Clang will kindly give an error to notify that it cannot handle the situation:

```text
% clang a.c -S -o - | grep callq
        callq   memcpy@PLT
% clang a.c -S -o - -DASM_LABEL
a.c:18:14: error: cannot apply asm label to function after its first use
extern void *memcpy(void *, const void *, size_t) asm("__GI_memcpy");
             ^                                        ~~~~~~~~~~~~~
1 error generated.
```

This is related to Clang's one-function-at-a-time parsing and code generation strategy. It is very difficult to match GCC's behavior.

glibc makes extensive usage of `hidden_proto` and `libc_hidden_proto`, where such problems may occur in many places.

There are cases where a .h file has an `extern inline` definition, followed by references from other inline functions.
A .c file adds asm label to the declaration.
Disabling `__USE_EXTERN_INLINES` can work around these issues, but many issues are due to other code organization difficulties.

## Aliasing an asm label

`nptl/pthread_create.c` has
```c
/* Globally enabled events.  */
td_thr_events_t __nptl_threads_events;
libc_hidden_proto (__nptl_threads_events)
libc_hidden_data_def (__nptl_threads_events)
```
which desugars to
```c
td_thr_events_t __nptl_threads_events;
extern __typeof (__nptl_threads_events) __nptl_threads_events __asm__ ("" "__GI___nptl_threads_events") __attribute__ ((visibility ("hidden")));
extern __typeof (__nptl_threads_events) __EI___nptl_threads_events __asm__("" "__nptl_threads_events");
extern __typeof (__nptl_threads_events) __EI___nptl_threads_events __attribute__((alias ("" "__GI___nptl_threads_events"))) __attribute__ ((__copy__ (__nptl_threads_events)));
```

GCC produces a hidden visibility `__GI___nptl_threads_events` and its default visibility alias `__nptl_threads_events`.

Clang reports an error:
```text
pthread_create.c:49:1: error: alias must point to a defined variable or function
libc_hidden_data_def (__nptl_threads_events)
^
```

GCC knows that `__GI___nptl_threads_events` (not a C declaration) is a definition, so `__attribute__((alias ("" "__GI___nptl_threads_events")))` can be used, but Clang does not know this fact.

## Alias to a weak alias

`wcsmbs/mbrtowc.c` has
```c
#include <wchar.h>

size_t
__mbrtowc (wchar_t *pwc, const char *s, size_t n, mbstate_t *ps)
...

libc_hidden_def (__mbrtowc)
weak_alias (__mbrtowc, mbrtowc)
libc_hidden_weak (mbrtowc)
```
which desugars to
```c
// Expanded from libc_hidden_proto (mbrtowc)
extern __typeof (mbrtowc) mbrtowc __asm__ ("" "__GI_mbrtowc") __attribute__ ((visibility ("hidden")));

// Expanded from libc_hidden_proto (__mbrtowc)
extern __typeof (mbrtowc) __mbrtowc __asm__ ("" "__GI___mbrtowc") __attribute__ ((visibility ("hidden")));

size_t
__mbrtowc (wchar_t *pwc, const char *s, size_t n, mbstate_t *ps)
...

extern __typeof (__mbrtowc) __EI___mbrtowc __asm__("" "__mbrtowc");
extern __typeof (__mbrtowc) __EI___mbrtowc __attribute__((alias ("" "__GI___mbrtowc"))) ;
extern __typeof (__mbrtowc) mbrtowc __attribute__ ((weak, alias ("__mbrtowc"))) ;  // mbrtowc => __EI___mbrtowc
extern __typeof (mbrtowc) __EI_mbrtowc __asm__("" "mbrtowc");
extern __typeof (mbrtowc) __EI_mbrtowc __attribute__((alias ("" "__GI_mbrtowc"))) __attribute__((weak));  // __EI_mbrtowc => mbrtowc; -Wignored-attributes here
```

Clang reports an error:
```text
mbrtowc.c:121:1: error: alias will always resolve to __GI___mbrtowc even if weak definition of __GI_mbrtowc is overridden [-Werror,-Wignored-attributes]
libc_hidden_weak (mbrtowc)
^
```

The declaration `__EI_mbrtowc` is a weak alias to `__GI_mbrtowc`.
`__GI_mbrtowc` is an asm label name. The original name belongs to the declaration `mbrtowc` which is itself an alias to `__mbrtowc`.
The Clang diagnostic just mentions the asm label names. Despite the warning, the emitted symbol properties match GCC:

```text
    26: 0000000000000000   596 FUNC    GLOBAL HIDDEN     1 __GI___mbrtowc
    33: 0000000000000000   596 FUNC    GLOBAL DEFAULT    1 __mbrtowc
    34: 0000000000000000   596 FUNC    WEAK   HIDDEN     1 __GI_mbrtowc
    35: 0000000000000000   596 FUNC    WEAK   DEFAULT    1 mbrtowc
```

To make `-Werror` builds work, we can compile with `-Wno-ignored-attributes`.

## `_Float128`

`long double` and float128 support in GCC is a huge mess.

In its PowerPC port, `long double` has 3 mangling schemes:

* `-mlong-double-64`: `e`
* `-mlong-double-128 -mabi=ibmlongdouble`: `g`
* `-mlong-double-128 -mabi=ieeelongdouble`: `u9__ieee128` (gcc <= 8.1: `U10__float128`)

In its x86 port, two different `long double` configurations share the same mangling code.

* `-mlong-double-64`: `e`
* `-mlong-double-80`: `e`
* `-mlong-double-128`: `g`

(I fixed some Clang issues in <https://reviews.llvm.org/D64276>.)

`__float128` has the same mangling code with 128-bit long double but is considered a distinct type in both C and C++.
So if you have two overloads `foo(__float128)` and `foo(long double)`, you will get a compiler/linker error due to conflicting symbols.

`_Float128` is identical to `__float128`.
`_Float128` is available in C mode but not in C++ mode.
When a future standard C++ introduces a float128 type, it has to be yet another new type to be different from the 128-bit long double.

```c
// C
#include <stdio.h>

int main() {
  // _Float128 = __float128
  puts(_Generic((_Float128)0, __float128: "same", default: "different"));
  // _Float128 != long double, even in -mlong-double-128 mode.
  puts(_Generic((_Float128)0, long double: "same", default: "different"));
}
```

Anyhow, glibc builds some float128 code on some PowerPC, x86, and perhaps a few other architectures.
Clang doesn't support `_Float128`. I have a pending patch <https://reviews.llvm.org/D111382> but given GCC's C++ issue I am now unsure whether Clang should support `_Float128` before the C++ support case is clearer.

For now, falling back to `typedef __float128 _Float128` is probably a good choice.

```patch
--- a/sysdeps/x86/bits/floatn.h
+++ b/sysdeps/x86/bits/floatn.h
@@ -29,5 +29,6 @@
 #if (defined __x86_64__                                                        \
      ? __GNUC_PREREQ (4, 3)                                            \
-     : (defined __GNU__ ? __GNUC_PREREQ (4, 5) : __GNUC_PREREQ (4, 4)))
+     : (defined __GNU__ ? __GNUC_PREREQ (4, 5) : __GNUC_PREREQ (4, 4))) \
+    || defined __clang__
 # define __HAVE_FLOAT128 1
 #else
@@ -88,5 +89,5 @@ typedef __float128 _Float128;

 /* __builtin_huge_valf128 doesn't exist before GCC 7.0.  */
-#  if !__GNUC_PREREQ (7, 0)
+#  if !__GNUC_PREREQ (7, 0) && !defined __clang__
 #   define __builtin_huge_valf128() ((_Float128) __builtin_huge_val ())
 #  endif
@@ -97,5 +98,5 @@ typedef __float128 _Float128;
    attempts to use _Float128 sNaNs will not work properly with older
    compilers.  */
-#  if !__GNUC_PREREQ (7, 0)
+#  if !__GNUC_PREREQ (7, 0) && !defined __clang__
 #   define __builtin_copysignf128 __builtin_copysignq
 #   define __builtin_fabsf128 __builtin_fabsq
@@ -109,5 +110,5 @@ typedef __float128 _Float128;
    been a __builtin_signbitf128 in GCC and the type-generic builtin is
    only available since GCC 6.  */
-#  if !__GNUC_PREREQ (6, 0)
+#  if !__GNUC_PREREQ (6, 0) && !defined __clang__
 #   define __builtin_signbitf128 __signbitf128
 #  endif
```

## `-fheinous-gnu-extensions`

Chris Lattner added the funny option in commit cda4d7e19619e136f2b597110b6a52ee9d641862.
It has an associated comment for `CheckAsmLValue`:
```text
GNU C has an extremely ugly extension whereby they silently ignore "noop" casts in places where an lvalue is required by an inline asm.
We emulate this behavior when -fheinous-gnu-extensions is specified, but provide a strong guidance to not use it.
This method checks to see if the argument is an acceptable l-value and returns false if it is a case we can handle.
```

This does look like wrong usage but also has a niche [type checking](https://gcc.gnu.org/pipermail/gcc-patches/2021-October/581470.html) feature:

```c
// GCC
void foo() {
  int i;
  long l;
  __asm ("" : "=r" ((unsigned)i));
  __asm ("" : "=r" ((long)l));
  __asm ("" : "=r" ((long long)l));
  __asm ("" : "=r" ((int)l)); // error: lvalue required in ‘asm’ statement
  __asm ("" : "=r" ((long)i)); // error: lvalue required in ‘asm’ statement
}
```

```c
// Clang
void foo() {
  int i;
  long l;
  __asm ("" : "=r" ((unsigned)i)); // error: invalid use of a cast in a inline asm context requiring an lvalue: remove the cast or build with -fheinous-gnu-extensions
  __asm ("" : "=r" ((long)l)); // ...
  __asm ("" : "=r" ((long long)l)); // ...
  __asm ("" : "=r" ((int)l)); // ...
  __asm ("" : "=r" ((long)i)); // ...
}
```

I attempted a fix [`[PATCH] include/longlong.h: Remove incorrect lvalue to rvalue conversion from asm output constraints`](https://gcc.gnu.org/pipermail/gcc-patches/2021-October/581257.html).
If GCC/glibc folks don't accept the loss of type checking, we probably have to carry around `-fheinous-gnu-extensions`.
```patch
--- a/stdlib/longlong.h
+++ b/stdlib/longlong.h
@@ -493,6 +493,6 @@ extern UDItype __umulsidi3 (USItype, USItype);
 #define add_ssaaaa(sh, sl, ah, al, bh, bl) \
   __asm__ ("add{q} {%5,%1|%1,%5}\n\tadc{q} {%3,%0|%0,%3}"              \
-          : "=r" ((UDItype) (sh)),                                     \
-            "=&r" ((UDItype) (sl))                                     \
+          : "=r" (sh),                                                 \
+            "=&r" (sl)                                                 \
           : "%0" ((UDItype) (ah)),                                     \
             "rme" ((UDItype) (bh)),                                    \
```

## LLVM's integrated assembler

On most targets, Clang uses LLVM's integrated assembler.
It lacks support for some unpopular features and may be stricter than GNU as in some cases.

```text
% cat a.s
bndmov %bnd0,(%rsp)
bnd
jmp *%r11
% as a.s
% clang -c a.s
a.s:2:1: error: invalid instruction mnemonic 'bnd'
bnd
^~~
```

I attempted to [improve `sysdeps/x86_64/configure.ac`](https://sourceware.org/pipermail/libc-alpha/2021-October/131827.html) but was asked to just remove the deprecated Intel MPX support.
So I sent [`[PATCH] elf: Remove Intel MPX support (lazy PLT, ld.so profile, and LD_AUDIT)`](https://sourceware.org/pipermail/libc-alpha/2021-October/131831.html).

As another example, `sysdeps/powerpc/powerpc64/le/power10/memmove.S` uses the directive `.machine power9`.
For some other x86 projects I noticed that GNU as' `.arch` supports a very wide range of `.arch` operands.

In LLVM, `*AsmParser` files do not do a good job tracking what features are needed to assemble instructions.
It is also tricky to recognize all GNU as supported operands.
The ISA feature compatibility check, while looks nice, actually provides a lower value.

So I patched LLVM's x86 and PowerPC targets to ignore `.arch` and `.machine` operands.

* A few `sysdeps/i386/fpu/*.S` files use [`.tfloat`](https://sourceware.org/binutils/docs/as/i386_002dFloat.html) (80-bit extended precision floating point) which is unsupported by LLVM's integrated assembler.
* `sysdeps/x86_64/fpu/multiarch/Makefile` uses `-msse2avx` to encode SSE instructions with VEX prefix. LLVM's integrated assembler doesn't support the option.

### Inline asm parsing

While GCC does minimal parsing of an inline asm statement (<https://gcc.gnu.org/onlinedocs/gcc/Size-of-an-asm.html>), Clang parses everything.

glibc uses an inline asm trick to get the constant value of a macro (e.g. a Linux syscall number).
`@@@` is invalid asm, so Clang rejects it.

```text
% cat a.c
#include <linux/version.h>
#include <sys/syscall.h>
#include <bits/syscall.h>

int main() {
  asm ("@@@%0@@@" :: "i"((long int)(X)));
}
% gcc -S -DX=__NR_mmap a.c -o - | grep @@@
        @@@$9@@@
% clang -S -DX=__NR_mmap a.c -o /dev/null
a.c:6:8: error: unexpected token at start of statement
  asm ("@@@%0@@@" :: "i"((long int)(X)));
       ^
<inline asm>:1:2: note: instantiated into assembly here
        @@@$9@@@
        ^
1 error generated.
```

The fix is simple. We can just surround the invalid part with a pair of `/* */`.
I avoid `#` because it may have weird interpretation on some exotic architecture (<https://sourceware.org/binutils/docs/as/Comments.html#Comments>).

## More rigid semantic analysis and finicky warnings

### `_Static_assert`

The first operand of a `_Static_assert` declaration must be an integer constant expression.
C11 defines operands which are required to make up an integer constant expression.

> 6 An integer constant expression shall have integer type and shall only have operands that are integer constants, enumeration constants, character constants, sizeof expressions whose results are integer constants, _Alignof expressions, and floating constants that are the immediate operands of casts. Cast operators in an integer constant expression shall only convert arithmetic types to integer types, except as part of an operand to the sizeof or _Alignof operator.
>
> 10 An implementation may accept other forms of constant expressions.

An implementation may or may not accept a const object (e.g. `const size_t allocation_size = 32768;`) as an operand.
Clang is simple: a const object cannot be used as an operand in a constant expression.
GCC is inconsistent ([PR102502](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=102502)):

* `-O2`: accepted
* `-O2 -Wpedantic`: `warning: expression in static assertion is not an integer constant expression [-Wpedantic]`
* `-O0`: `error: expression in static assertion is not constant`

```text
% cat reduce.i
const int __alloc_dir_allocation_size = 8;
void __alloc_dir() { _Static_assert(__alloc_dir_allocation_size, ""); }

% gcc reduce.i -c -std=c11
reduce.i: In function ‘__alloc_dir’:
reduce.i:2:37: error: expression in static assertion is not constant
    2 | void __alloc_dir() { _Static_assert(__alloc_dir_allocation_size, ""); }
      |                                     ^~~~~~~~~~~~~~~~~~~~~~~~~~~
% gcc reduce.i -c -std=c11 -O1
% gcc reduce.i -c -std=c11 -O2
% gcc reduce.i -c -std=c11 -O2 -Wpedantic
reduce.i: In function ‘__alloc_dir’:
reduce.i:2:37: warning: expression in static assertion is not an integer constant expression [-Wpedantic]
    2 | void __alloc_dir() { _Static_assert(__alloc_dir_allocation_size, ""); }
      |                                     ^~~~~~~~~~~~~~~~~~~~~~~~~~~

% clang reduce.i -c -std=c11
reduce.i:2:37: error: static_assert expression is not an integral constant expression
void __alloc_dir() { _Static_assert(__alloc_dir_allocation_size, ""); }
                                    ^~~~~~~~~~~~~~~~~~~~~~~~~~~
1 error generated.
```

`sysdeps/unix/sysv/linux/opendir.c` has such an issue, which has been fixed by [`linux: Fix a possibly non-constant expression in _Static_assert`](https://sourceware.org/git/?p=glibc.git;a=commit;h=aa783f9a7b774d67487daa9376095738aef5cf88).

## Miscellaneous

`misc/sys/cdefs.h` has
```c
# define __va_arg_pack() __builtin_va_arg_pack ()
# define __va_arg_pack_len() __builtin_va_arg_pack_len ()
```

Clang does not support [`__builtin_va_arg_pack`](https://gcc.gnu.org/onlinedocs/gcc/Constructing-Calls.html).

`sysdeps/unix/sysv/linux/errlist-compat.c` is compiled with GCC-specific `-fno-toplevel-reorder`.
With `-fno-toplevel-reorder`, the GCC output looks like:
```asm
        .hidden _sys_errlist_internal
        .globl  _sys_errlist_internal
        .type   _sys_errlist_internal, @object
        .size   _sys_errlist_internal, 1072
_sys_errlist_internal:
         ...

        .globl __GLIBC_2_1_sys_errlist
        .set __GLIBC_2_1_sys_errlist, _sys_errlist_internal
        .type __GLIBC_2_1_sys_errlist, %object
        .size __GLIBC_2_1_sys_errlist, 125 * (64 / 8)
```

The second `.size` directive wins. `__GLIBC_2_1_sys_errlist` has size 125 * (64 / 8) = 1000.

Without `-fno-toplevel-reorder`, the GCC output looks like:
```asm
        .globl __GLIBC_2_1_sys_errlist
        .set __GLIBC_2_1_sys_errlist, _sys_errlist_internal
        .type __GLIBC_2_1_sys_errlist, %object
        .size __GLIBC_2_1_sys_errlist, 125 * (64 / 8)

        .hidden _sys_errlist_internal
        .globl  _sys_errlist_internal
        .type   _sys_errlist_internal, @object
        .size   _sys_errlist_internal, 1072
_sys_errlist_internal:
         ...
```

In GNU assembler, `.set __GLIBC_2_1_sys_errlist, _sys_errlist_internal` changes the value to an `O_symbol` (instead of `O_constant`), because `_sys_errlist_internal` hasn't been defined.
In `resolve_symbol_value`, the size field will be overridden by the symbol's size.
Therefore `.size __GLIBC_2_1_sys_errlist, 125 * (64 / 8)` is ignored, regardless of where it is placed.
`__GLIBC_2_1_sys_errlist` has the same size as `_sys_errlist_internal`, 1072.

`check-abi-libc`, glibc expects the size 1000.

## Appendix

Clang buildability of glibc will enable instrumentation and make some research convenient.

* [Debloating Software through Piece-Wise Compilation and Loading](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-quach.pdf), USENIX Security Symposium, Baltimore, Maryland, August 2018: "Due to known fundamental limitations in compiling glibc using LLVM[1], we piece-wise compiled musl-libc—another popular and comprehensive flavor of the C library."
* [Hardware-Assisted Fine-Grained Control-Flow Integrity: Adding Lasers to Intel’s CET/IBT](https://static.sched.com/hosted_files/lssna2021/8f/LSS_FINEIBT_JOAOMOREIRA.pdf), Linux Security Summit 2021: "GLIBC support on top of GRTE branch"
