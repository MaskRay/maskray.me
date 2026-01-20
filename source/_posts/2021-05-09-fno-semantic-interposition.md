---
layout: post
title: -fno-semantic-interposition
author: MaskRay
tags: [gcc,llvm,linker]
---

Updated in 2022-05.

This article is a continuation to [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic).
It focuses on the GCC/Clang option `-fno-semantic-interposition` and why it can (sometimes incredibly) optimize `-fPIC` programs.

<!-- more -->

## Source level implication

Now let's discuss how the compiler models C/C++ in terms of the binary format semantics.
An external linkage function/variable has `STB_GLOBAL` binding and `STV_DEFAULT` visibility by default.
(Note: we exclude vague linkage definitions for our discussion.)

```c
// f and g are STB_GLOBAL STV_DEFAULT.
void f() { ... }
void g() { f(); }
```

A `-fpic` compiled object file can be linked as a shared object.
`f` is preemptible when linked as a shared object.
GCC's `-fpic` interpretation is to anticipate that the definition may change and therefore suppress interprocedural optimizations including inlining.

The emitted assembly looks like:
```asm
.globl f
f:
  ...

.globl g
g:
  call f@PLT # or call f; R_X86_64_PLT32
```

`f` cannot be inlined into `g`. `g` cannot make use of `f`'s characteristics for optimizations.
(This turns out to be the biggest difference between `-fpie` and `-fpic`. A `-fpie` object file cannot make a shared object, so a definition is known to be non-preemptible.)
In CPython's case, they said it was up to 30% due to suppressed interprocedural optimizations.

The pessimization does not stop here.
See my next article [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic) for the cost.

This is a feature used by extremely few libraries that penalizes most other libraries.
I'll say fewer than 0.1%. A portable project needs `ld -interposable`, `DYLD_FORCE_FLAT_NAMESPACE` or `__attribute__((section("__DATA,__interpose")))` to work on Mach-O, but there is no counterpart on Windows.
Read on.

On PE/COFF, `f` cannot be interposed.

## GCC `-fno-semantic-interposition`

GCC 5 introduced `-fno-semantic-interposition` to optimize `-fpic`.
It assumes that interposition will not change the semantics (and side effects) of the interposed function.
First, GCC can apply interprocedural optimizations including inlining like `-fno-pic` and `-fpie`.
Second, in the emitted assembly, a function call will go through a local alias to avoid PLT if linked with `-shared`.

```asm
.globl f
f:                      # STB_GLOBAL
  ...
  .set f.localalias, f  # STB_LOCAL

g:
  call f.localalias
```

If `f` is a non-definition declaration, `-fno-semantic-interposition` has no behavior difference.

Now let's look at a more complex example with an address taken operation.

```c
void f() { asm("#"); }
void *g() {
  f();
  return f;
}
```

`gcc -O2 -fpic -fno-semantic-interposition` produces
```asm
g:
  #
  movq f@GOTPCREL(%rip), %rax
  ret
```

Note that GCC still uses the original symbol for address taking so that interposition can keep pointer equality.

Note that GCC does not use a local alias for global variables.
This is important to keep copy relocations working. Scroll to the bottom of the article for a libc++ variable interspoition example.
Disabling variable interposition would cause breakage.

I filed [PR100937](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100937) and posted [a configure option patch](https://gcc.gnu.org/pipermail/gcc-patches/2021-June/572018.html).
It was immediately closed as wontfix.

In 2021-11, GCC changed `-Ofast` to imply `-fno-semantic-interposition`: [options: Make -Ofast switch off -fsemantic-interposition](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=458d2c689963d8461d84670a3d8988cd6ecbfd81).

## Clang -fno-semantic-interposition

### Long-standing behavior

It turns out that the first merit of the GCC feature "interprocedural optimizations including inlining are applicable" is actually Clang's long-standing behavior for external linkage definitions in `-fpic` code.

When `-fsemantic-interposition` was contributed by Serge Guelton, I noted that we should keep the aggressive behavior, even if it differs from GCC, not to regress the long-standing optimizations.
(ipconstprop, inliner, sccp, sroa treat normal ExternalLinkage `GlobalObject`s non-interposable.)

### `dso_local` inference in `-fpic -fno-semantic-interposition` mode

I contributed an optimization to Clang 11: in `-fpic -fno-semantic-interposition` mode, default visibility external linkage definitions get the `dso_local` specifier, like in `-fno-pic` and `-fpie` modes.
For `-fpic` code, access to a `dso_local` symbol will go through its local alias `.Lfoo$local`.
For `-fno-pic` and `-fpie` code, access to a `dso_local` symbol can use the original `foo` because the object file shall not be linked with `-shared`.

With `dso_local`, there are some noticeable behavior differences:

* variable access: access the local symbol directly instead of going through a GOT indirection
* function call: `call .Lfoo$local`
* taking the address of a function: similar to a variable access

For the previous C example, the emitted assembly will look like the following.
```asm
.globl f
f:         # STB_GLOBAL
.Lf$local: # STB_LOCAL
  ...
g:
  call .Lf$local
```

The local alias is a `.L` symbol. This is deliberate:

* The assembler suppresses the symbol table entry (if not RISC-V `-mrelax`) and converts relocations to reference the section symbol instead.
* Tools cannot be confused by two symbols at the same location. In an object file produced by GCC, currently llvm-objdump will label a function as `f.localalias`.

The branch target symbol will have a different type: `STT_FUNC` -> `STT_NOTYPE`.
In some processor supplementary ABI, there may be certain implications on range extension thunks.
AArch64 is known to be good but ABI makers should be aware of this.

GCC doesn't use local aliases for global variables.
It turns out that I [misinterepreted GCC's semantics](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100483),
I plan to remove local aliases for global variables in Clang.
This rarely matters for performance.

In Clang cc1, there are three modes:

* `-fsemantic-interposition`: this represents `-fpic -fsemantic-interposition`. Don't set `dso_local` on default visibility external linkage definitions. Emit a module flag metadata `SemanticInterposition` to disallow interprocedural optimizations.
* `-fhalf-no-semantic-interposition`: this represents `-fpic` without a semantic interposition option. Don't set `dso_local` on default visibility external linkage definitions. However, interprocedural optimizations on such definitions are allowed.
* (default): this represents either of `-fno-pic`, `-fpie`, and `-fpic -fno-semantic-interposition`. Set `dso_local` on default visibility external linkage definitions. Interprocedural optimizations on such definitions are allowed.

### Supported targets

As of Clang 12, `-fno-semantic-interposition` is only effective on x86.

Clang 13 will support AArch64 (<https://reviews.llvm.org/D101873>) and RISC-V (<https://reviews.llvm.org/D101876>).

## What GlobalValues can be optimized in LLVM IR?

It must be a definition for the linker, so `available_externally` and `extern_weak` linkages are excluded.
It must be a definition not replaceable by a different copy in the same linkage unit, so `common, {linkonce,weak}{,_odr}` linkages are excluded.
It must lower to a symbol with a non-local binding, so `private` and `internal` linkages are excluded.
The `appending` linkage can only be used by special LLVM internal global variables.

So the only optimizable linkage is `external`.
We should additionally exclude an `external` in a deduplicate COMDAT, which is replaceable.

## ThinLTO

I believe ThinLTO `-fpic -fno-semantic-interposition` works for ThinLTO:)
ThinLTO required two changes: if a GlobalVariable is converted to a declaration, we should drop the `dso_local` specifier (D74749 D74751).

## Space overhead

In GCC's `foo.localalias` scheme, there is an extra symbol table entry (`sizeof(Elf64_Sym) = 24`) and a string in the string table.

In Clang's `.Lfoo$local` scheme, this generally costs a `STT_SECTION` symbol table entry (the entry can usually be suppressed).

## Can Clang default to `-fno-semantic-interposition`?

Clang currently has three modes.
There are some optimization opportunities between the half mode and the full `-fno-semantic-interposition`.
It is natural to ask whether we can drop the half mode and make `-fpic` default to the full mode.

This is something I'd like Clang to do, but I'll note that there is still some risk.
Some points favoring the changed default.

### Interprocedural optimizations (including inlining)

For ELF -fpic, Clang never suppresses interprocedural optimizations (including inlining) on default visibility external linkage definitions.
So projects relying on blocked interprocedural optimizations have been broken for years. They only probably work recently by specifying `-fsemantic-interposition`.

### Assembler behavior for `VK_None`

```asm
.globl f
f:
  ret

.globl g
g:
  bl f   # VK_None
  ret
```

Before 2020-01 (<https://reviews.llvm.org/D72197>), the integrated assembler resolved the fixup when the target symbol and the location are in the same section.
There was no relocation, so the linker could not produce a PLT.

Non-x86 targets typically use `VK_None` for branch instruction. x86 uses `VK_PLT` for `-fpie` and `-fpic`.

If a project passed with `-fno-function-sections` on aarch64/ppc/etc before 2020-01, we have some confidence that the project does not rely on function semantic interposition.

### In action

I started a thread: [-fpic ELF default: reclaim some -fno-semantic-interposition optimization opportunities?](https://lists.llvm.org/pipermail/llvm-dev/2021-May/150467.html)

I wanted a configure option `--enable-default-semantic-interposition` from GCC, so that `-fpic` compiles can default to `-fno-semantic-interposition`.
The feature request <https://gcc.gnu.org/PR100937> was rejected.

## Applications

Python. CPython 3.10 sets `-fno-semantic-interposition`.
[Red Hat Enterprise Linux 8.2 brings faster Python 3.8 run speeds](https://developers.redhat.com/blog/2020/06/25/red-hat-enterprise-linux-8-2-brings-faster-python-3-8-run-speeds/) says there is a huge improvement (up to 30%).
This is really an upper bound you can see from real world applications.
This suggests that the relevant code was not particularly tuned for the (unfortunate) ELF toolchain defaults.

A small single-digit performance boost (say, 4%) is what I'd normally expect.
In <https://bugs.archlinux.org/task/70697> and <https://bugzilla.redhat.com/show_bug.cgi?id=1956484>, I suggest that distributions consider `-fno-semantic-interposition` and `-Bsymbolic-functions` when building Clang.
My x86_64 defconfig build with a clang built with `-fPIC -fno-semantic-interposition` is 4% faster (total time, so the clang improvement is larger than 4%).
The number may be larger with GCC built -fPIC clang since GCC -fPIC disables interprocedural optimizations on default visibility definitions.
With `-fPIC -fno-semantic-interposition -Wl,-Bsymbolic-functions`, my clang is 15% faster (relocation processing takes too much time).

When built with GCC, `-fno-semantic-interposition` makes my `libclang-cpp.so` 2% smaller on top of `-Wl,-Bsymbolic-functions`

## Variable interposition examples

### libc++ locale

`libcxx/src/locale.cpp` calls `install(&make<_VSTD::ctype<char> >(nullptr, false, 1u));` to initialize `std::__1::ctype<char>::id`.
In `libcxx/include/__locale` (which may be transitively included by the executable), the `std::__1::ctype<char>::id` reference is expected to be bound to `std::__1::ctype<char>::id` defined in `libc++.so.1`.
If `std::__1::ctype<char>::id` is copy relocated by the executable, the `std::__1::ctype<char>::id` reference from the executable will be bound to the uninitialized copy in the executable, causing a no-facet failure.

```cpp
// _Facet is std::__1::ctype<char>
template <class _Facet>
inline _LIBCPP_INLINE_VISIBILITY
const _Facet&
use_facet(const locale& __l)
{
    return static_cast<const _Facet&>(*__l.use_facet(_Facet::id));
}
```
