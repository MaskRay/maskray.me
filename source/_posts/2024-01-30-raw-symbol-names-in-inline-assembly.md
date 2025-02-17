layout: post
title: Raw symbol names in inline assembly
author: MaskRay
tags: [gcc,llvm]
---

For operands in asm statements, GCC has supported the constraints "i" and "s" for a long time (since at least 1992).
```
// gcc/common.md
(define_constraint "i"
  "Matches a general integer constant."
  (and (match_test "CONSTANT_P (op)")
       (match_test "!flag_pic || LEGITIMATE_PIC_OPERAND_P (op)")))

(define_constraint "s"
  "Matches a symbolic integer constant."
  (and (match_test "CONSTANT_P (op)")
       (match_test "!CONST_SCALAR_INT_P (op)")
       (match_test "!flag_pic || LEGITIMATE_PIC_OPERAND_P (op)")))
```

<!-- more -->

`CONSTANT_P` matches a class of RTL expressions called [`RTX_CONST_OBJ`](https://gcc.gnu.org/onlinedocs/gccint/RTL-Classes.html).

> An RTX code that represents a constant object. HIGH is also included in this class.

The most interesting objects in this class are constant integers, constant floating points, symbol or label references with a constant offset.
"s" is like "i", but does not match constant integers (e.g. `"s"(1)` is an error). So "s" essentially matches a symbol or label reference with a constant offset.

"s" can be used to create an artificial reference for [linker garbage collection](/blog/2021-02-28-linker-garbage-collection), define sections to hold symbol addresses, or even enable more creative applications.

```cpp
namespace ns { extern int a[2][2]; }
void fun();
void foo() {
label:
  asm(".pushsection .xxx,\"aw\"; .dc.a %0; .popsection" :: "s"(&ns::a[1][1]));
  asm(".reloc ., BFD_RELOC_NONE, %0" :: "s"(fun));
  asm("// %0" :: "s"(&&label));
}
```

C++ templates can make this easier to use.
```cpp
template <class T, T &x>
void ref() { asm (".reloc ., BFD_RELOC_NONE, %0" :: "s"(x)); }
void use() { ref<decltype(ns::a), ns::a>(); }
```

```c
// Materialize the symbol address manually
asm("adrp %0, %1\nadd %0, %0, :lo12:%1" : "=r"(ret) : "S"(&var));
```

Using the generic `r` or `m` constraint in such cases would instruct GCC to generate instructions to compute the address, which can be wasteful if the materialized address isn't actually needed.

```cpp
// aarch64
  asm("// %0" :: "r"(fun));
// adrp    x0, _GLOBAL_OFFSET_TABLE_
// ldr     x0, [x0, #:gotpage_lo15:_Z3funv]
```

The condition `!flag_pic || LEGITIMATE_PIC_OPERAND_P (op)` highlights a key distinction in GCC's handling of symbol references:

* Non-PIC code (`-fno-pic`): The "i" and "s" constraints are freely permitted.
* PIC code (`-fpie` and `-fpic`): The architecture-specific `LEGITIMATE_PIC_OPERAND_P(X)` macro dictates whether these constraints are allowed.

While the default implementation (`gcc/defaults.h`) is permissive (used by MIPS, PowerPC, and RISC-V), many ports impose stricter restrictions, often disallowing preemptible symbols under PIC.

This differentiation probably stems from historical and architectural considerations:

* Non-PIC code: Absolute addresses could be directly embedded in instructions like an immediate integer operand.
* PIC code with dynamic linking: The need for [GOT indirection](/blog/2021-08-29-all-about-global-offset-table) often requires an addressing mode different from absolute addressing and more than one instructions.

Nevertheless, I think this symbol preemptibility limitation for "s" is unfortunate.
Ideally, we could retain the current "i" for immediate integer operand (after linking), and design "s" for a raw symbol name with a constant offset, ignoring symbol preemptibility.
This architecture-agnostic "s" would simplify metadata section utilization and boost code portability.

Below are some architecture-specific notes.

## AArch32

In `gcc/config/arm`, `LEGITIMATE_PIC_OPERAND_P(X)` has a complex definition and it seems to disallow any non-TLS symbol reference, which means that "s" cannot be used for PIC.

"US" can be used for a symbol reference without an offset (e.g. `&a[0]` when `a` is an array) in PIC code, but there is no good way to match `&a[1]`.
To get rid of the `#` prefix, use the modifier "c".

```c
extern int a[4];
void foo() { asm("// %c0" :: "US"(&a[0])); }
```

## AArch64

In `gcc/config/aarch64`, `LEGITIMATE_PIC_OPERAND_P(X)` disallows any symbol reference, which means that "i" and "s" cannot be used for PIC.
Instead, the [constraint "S"](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html) has been supported since the initial port (2012) to reference a symbol or label.

Clang 7 also implemented "S".

## RISC-V

`gcc/config/riscv` uses the generic `LEGITIMATE_PIC_OPERAND_P(X)`, so "s" can be used in PIC mode.

The constraint "S" is supported (since the beginning of the port in 2017) for a similar purpose, but requires a non-preemptible symbol.

I [implemented](https://reviews.llvm.org/D105254) the constraint "S" for Clang 14 but realized that "S" is less useful in GCC, so I sent a patch to [implement "s"](https://github.com/llvm/llvm-project/pull/80201).

## x86

We can use the constraint "s" (or "i") with the modifier "p" to print raw symbol name without syntax-specific prefixes, but it does not work when:

* the symbol is preemptible (similar to RISC-V's "S")
* or `-mcmodel=large`
* or `-mcmodel=medium` for large data

```c
void foo() {
label:
  asm("// %p0" :: "i"(foo));  // Does not work if foo is preemptible
  asm("// %p0" :: "i"(&&label));
}
```

I filed the [feature request](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=105576) for a new constraint in May 2022 and eventually went ahead and implement it by myself.
The patch landed today, catching up the GCC 14 release.
I have also implemented "Ws" for Clang 18.1.

BTW, you can also the modifier "c".

> Require a constant operand and print the constant expression with no punctuation.

I think having such a [long list of modifiers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#x86-Operand-Modifiers) is unfortunate.

## Summary

If your program wants to adopt raw symbol names in inline assembly, consider the following list for best portability and semantics:

* AArch32: "US" (for symbol reference without an offset)
* AArch64: "S"
* x86: "Ws", GCC 14+, Clang 18+
* MIPS/PowerPC/RISC-V: "s"

## Applications

Linux kernel's [jump label patching](https://docs.kernel.org/staging/static-keys.html) showcases the practical benefits of accessing raw symbol names in inline assembly.
Actually, this feature likely drew me down this rabbit hole about raw symbol names a few years ago.

Many ports use the constraint "i", which is more or less a hack.

```
// gcc/common.md
(define_constraint "i"
  "Matches a general integer constant."
  (and (match_test "CONSTANT_P (op)")
       (match_test "!flag_pic || LEGITIMATE_PIC_OPERAND_P (op)")))
```

In the non-PIC mode, "i" does works with a constant, symbol reference, or label reference.
However, in the PIC mode, "i" on a symbol reference is rejected by certain GCC ports (e.g. aarch64).
I went ahead and sent an [arm64 patch](https://lore.kernel.org/linux-arm-kernel/20240131065322.1126831-1-maskray@google.com/T/#u):)

BTW, the jump label patching implementation prevents kernel compilation without optimizations (along with other clever tricks).
`include/linux/jump_label.h` offers an interesting example of [function overloading](https://lore.kernel.org/lkml/152261542576.30503.8474929957674990486.stgit@warthog.procyon.org.uk/) using `__builtin_types_compatible_p` and an undefined symbol.
```c
#define static_branch_likely(x)                                                 \
({                                                                              \
        bool branch;                                                            \
        if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))   \
                branch = !arch_static_branch(&(x)->key, true);                  \
        else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
                branch = !arch_static_branch_jump(&(x)->key, true);             \
        else                                                                    \
                branch = ____wrong_branch_error();                              \
        likely_notrace(branch);                                                         \
})
```

Ideally, if the kernel switches to C++, a template would provide a more elegant and portable solution, enabling compilation without optimizations.
```cpp
template <class T, T &key>
bool arch_static_branch(bool branch) {
  asm_volatile_goto(... : : "Ws"(key), "i" (2 | branch) : : l_yes);
  return false;
l_yes:
  return true;
}
```
