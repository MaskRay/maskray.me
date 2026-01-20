---
layout: post
title: Weak symbol
author: MaskRay
tags: [binutils,linker]
---

Updated in 2023-10.

## C/C++

GCC and Clang support `__attribute__((weak))` which marks a symbol weak. The same effect can be achieved with a preprocessor directive `#pragma weak symbol`.

## Object file format

In ELF, there are three main symbol bindings. The ELF specification says:

* `STB_LOCAL`: Local symbols are not visible outside the object file containing their definition. Local symbols of the same name may exist in multiple files without interfering with each other.
* `STB_GLOBAL`: Global symbols are visible to all object files being combined. One file's definition of a global symbol will satisfy another file's undefined reference to the same global symbol.
* `STB_WEAK`: Weak symbols resemble global symbols, but their definitions have lower precedence.

<!-- more -->

In the GNU ABI, there is another binding `STB_GNU_UNIQUE`, which is like `STB_GLOBAL` with extra semantics (unique even with `RTLD_LOCAL`, nodelete).

In GNU as flavored assembly, you can set the binding of a symbol via `.weak sym`.

Since a symbol has only one binding, a symbol cannot be global and weak at the same time.
However, in GNU as, `.weak` overrides `.globl` since [1996](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=5ca547dc2399a0a5d9f20626d4bf5547c3ccfddd).
In the LLVM integrated assembler, the last directive wins.
Since LLVM 12 (<https://reviews.llvm.org/D90108>), the integrated assembler errors/warns for changed binding.
For `.globl sym; .weak sym`, it reports a warning instead of an error because the behavior actually matches GNU as, but relying on directive overridding is error-prone.

```asm
# error: local changed binding to STB_GLOBAL
local:
.local local
.globl local

## `.globl x; .weak x` matches the GNU as behavior. llvm-mc issues a warning.
# warning: global changed binding to STB_WEAK
global:
.global global
.weak global

# error: weak changed binding to STB_LOCAL
weak:
.weak weak
.local weak
```

A weak symbol can be either a definition or a reference (i.e. undefined). This is distinguished by the section index:

* `st_shndx==SHN_UNDEF`: weak reference
* `st_shndx!=SHN_UNDEF`: weak definition

## Semantics required by the ELF specification

The specification says very little about a weak symbol.

> When the link editor combines several relocatable object files, it does not allow multiple definitions of STB_GLOBAL symbols with the same name. On the other hand, if a defined global symbol exists, the appearance of a weak symbol with the same name will not cause an error. The link editor honors the global definition and ignores the weak ones. Similarly, if a common symbol exists (that is, a symbol whose st_shndx field holds SHN_COMMON), the appearance of a weak symbol with the same name will not cause an error. The link editor honors the common definition and ignores the weak ones.
>
> When the link editor searches archive libraries [see ``Archive File'' in Chapter 7], it extracts archive members that contain definitions of undefined global symbols. The member's definition may be either a global or a weak symbol. The link editor does not extract archive members to resolve undefined weak symbols. Unresolved weak symbols have a zero value.

Weak definitions allow multiple definitions. While a global definition can override a weak definition, it is unspecified what the linker should do when there are two weak definitions of the same name but no global definition.
In GNU ld/gold/ld.lld, the linker selects the first weak definition and resolves all references to it.

An undefined weak symbol does not extract archive members. (ld.lld uses a weak `LazyObject` to represent such a symbol.)

There is a remark

> The behavior of weak symbols in areas not specified by this document is implementation defined. Weak symbols are intended primarily for use in system software. Applications using weak symbols are unreliable since changes in the runtime environment might cause the execution to fail.

## Weak definitions

In C++, inline functions, template instantiations and a few other things can be defined in multiple object files but need deduplication at link time.

Before `.gnu.linkonce.*`/`GRP_COMDAT` were invented, the implementations used weak definitions to avoid linker multiple definition errors.
The weak definition convention remains post `GRP_COMDAT`. The weak definition can be used for compatibility for linkers which do not understand pre-COMDAT `.gnu.linkonce.*` or COMDAT.
Using `STB_GLOBAL` for COMDAT definitions can detect ODR violations caused by a COMDAT definition and a non-COMDAT definition. This will however be a significant behavior change.
gold has an option `--detect-odr-violations`: the option checks whether there are two weak definitions with different file:line debugging information.

A replaceable definition can be declared weak. A `STB_GLOBAL` definition from another translation unit can override it.
This is a great way providing a default/fallback definition in a library which can be redefined by applications.

```cpp
// lib.cc
__attribute__((weak)) void fun() {
  ...
}

void feature() {
  fun();
}

// app.cc - override the default implementation
void fun() {
  ...
}
```

A weak alias is a special form of weak definitions. It defines a weak symbol by reusing an existing definition.
A useful technique creates a weak symbol aliasing a local definition.
```cpp
static void impl() {}
__attribute__((weak,alias("impl"))) void fun();
```

(
An example interprocedural optimization bug introduced in GCC 4.8.2 and fixed in 4.9.2/5.0
```c
// https://gcc.gnu.org/bugzilla/show_bug.cgi?id=61144
// foo is an inexact definition but ipa misoptimizes bar() to always return 0.
static int dummy = 0;
extern int foo __attribute__((__weak__, __alias__("dummy")));
int bar() { if (foo) return 1; return 0; }
```
)

## Weak references

The ELF specification says "Unresolved weak symbols have a zero value." This property can be used to check whether a definition is provided.
A common pattern is to implement an optional hook.

```cpp
__attribute__((weak)) void undef_weak_fun();

  if (&undef_weak_fun)
    undef_weak_fun();
```

The compiler does not know whether the symbol ends up being defined or undefined.
It generally takes a conservative approach by emitting a code sequence loading the address from a GOT entry.

```asm
# AArch64
adrp    x0, :got:undef_weak_fun
ldr     x0, [x0, :got_lo12:undef_weak_fun]

# x86-64
movq    undef_weak_fun@GOTPCREL(%rip), %rax

# riscv64
.Lpcrel_hi0:
auipc   a0, %got_pcrel_hi(undef_weak_fun)
ld      a0, %pcrel_lo(.Lpcrel_hi0)(a0)
```

The code sequence has one or many GOT-generating relocations.

If the symbol ends up being undefined, the generated GOT entry has an initial value of zero and no corresponding dynamic relocation.
At run-time the code will get an address of zero.

If the symbol ends up being defined, at run-time the code will get the symbol address.
See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table).

Historically, toolchains, especially for the lesser-used architectures, tend to have more bugs with weak references, so musl refrains from using weak references.
The above pattern can be replaced with a weak alias.

```cpp
static void noop() {}
__attribute__((weak,alias("noop"))) void undef_weak_fun();

  undef_weak_fun();
```

In most cases weak references resolve to a GOT entry for the symbol.

For ELF `-fno-pic`, there is an optimization: the emitted code may use an absolute relocation to check whether the address is zero.
However, if the symbol turns out to be defined in a shared object and the linked image needs a dynamic section, there will be a [canonical PLT entry](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected).

PE-COFF can emulate this feature with an `IMAGE_SYM_CLASS_WEAK_EXTERNAL` `IMAGE_SYM_UNDEFINED` symbol with an `IMAGE_SYM_CLASS_EXTERNAL` `IMAGE_SYM_ABSOLUTE` auxiliary symbol (named `.weak.<weaksymbol>.<relatedstrongsymbol>` in GNU).

If a weak references turns out to be undefined, MinGW's `.refptr.` mechanism can ensure the value is 0.

### Weak references and shared objects

A weak reference can be satisfied by a shared object definition.
The weak reference is no different from a regular reference.

### Weak references and archives

This is a lesser-known rule.
When an ELF linker sees a weak reference, it does not extract an archive member to satisfy the weak reference.
Please make sure the archive member is extracted due to other symbols.

There is a related [longstanding problem in libstdc++](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=58909): because references to `pthread_*` are weak, the relevant members in `-lpthread` may not be extracted in a static link:

```text
% cat a.cc
#include <condition_variable>
int main() { std::condition_variable a; }
% g++ -static -pthread a.cc
% ./a.out
Segmentation fault
```

You can use `-Wl,-y` to understand why this happens.
GNU ld discards an archive if it does not need to be extracted, so I have to use ld.lld to show `lazy definition`.
```text
% g++ -fuse-ld=lld -static a.cc -lpthread -Wl,-y,pthread_cond_destroy
/usr/lib/gcc/x86_64-linux-gnu/10/../../../x86_64-linux-gnu/libpthread.a: lazy definition of pthread_cond_destroy
/usr/lib/gcc/x86_64-linux-gnu/10/libstdc++.a(condition_variable.o): reference to pthread_cond_destroy
```

### Binding of an undefined symbol in the linked image

If an object file has an undefined symbol not defined by other object files (for archives, we consider extracted archive members the same as object files), the symbol is undefined in the linked image.
The symbol may be defined by a shared object, but the linked image still has an undefined symbol.
If all relocations to the symbol are discarded by `--gc-sections`, the undefined symbol will be removed from the linked image.
If the undefined symbol is retained, we say it is unresolved.

The linker will usually report an undefined symbol error if the symbol is `STB_GLOBAL`.
(`-z undefs` and `--no-allow-shlib-undefined` are the default when linking an executable; see [Explain GNU style linker options](/blog/2020-11-15-explain-gnu-linker-options) for details).
If the symbol is unversioned and weak, the linker will suppress the diagnostic.
(If the symbol is versioned weak, the linker will still report an error. See [All about symbol versioning](/blog/2020-11-26-all-about-symbol-versioning))

A `STB_LOCAL` undefined symbol is not allowed, so the binding can be either `STB_GLOBAL` or `STB_WEAK`.
The binding is `STB_WEAK` if all undefined symbols in object files are `STB_WEAK`, otherwise the binding is `STB_GLOBAL`.
Note: symbols in shared objects do not affect the binding.

### Relocation types

#### `-fno-pic`

GCC and Clang `-fno-pic` for most targets emit absolute relocations.
ppc64 uses TOC-generating relocations.
aarch64 uses `.rodata.cst8` which is similar to GOT.

`clang -fno-pic -fno-direct-access-external-data` emits GOT-generating relocations even for hidden undefined weak symbols.

#### `-fpie` and `-fpic`

Compilers do not emit PC-relative relocations.

GCC and Clang `-fpie` and `-fpic` emit GOT-generating relocations, even for hidden undefined weak symbols.
ppc64 `-fno-pic` emits TOC-generating relocations.

Address taken when initializing static storage data may emit absolute relocations.

### Unresolved weak references and `R_*_GLOB_DAT`

#### Absolute relocations

In `-no-pie` mode, all of GNU ld, gold, and ld.lld statically resolve absolute relocations to 0.

`-pie` and `-shared` are complex. In GNU ld, different ports have different behaviors.
The x86 port tries to be smart (Observation, should be close to written rules): if there is at least one GOT-generating or PLT-generating relocation and `-z dynamic-undefined-weak` (enabled by default) is in effect, generate a dynamic relocation.
This may cause an `R_X86_64_64` text relocation which is disallowed by `-z text`.

Related issues:

GNU ld aarch64 before [2017-11](https://sourceware.org/bugzilla/show_bug.cgi?id=22269#c30) might emit a dynamic relocation `R_AARCH64_ABS64` for static PIE.
GNU ld arm before [2020-11](https://sourceware.org/bugzilla/show_bug.cgi?id=22269#c40) might emit a dynamic relocation `R_ARM_RELATIVE` for static PIE.
GNU ld ppc since [2021-05](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=4916030821bb0b052091bd1e29f1851e1512a056) emits a dynamic relocation by default if no text relocation is needed.

ppc supports `-z {,no}dynamic-undefined-weak` since 2021-05.

ld.lld just emits dynamic relocations unconditionally for `-pie` and `-shared`, since 13.0.0 ([D105164](https://reviews.llvm.org/D105164)).
Previous versions may suppress the relocation for `-pie`.

#### PC-relative relocations

Modern compilers do not emit non-branch PC-relative relocations.

GCC<5 (at least x86_64 and arm) may emit PC-relative relocations for hidden undefined weak symbols.
GCC<5 i386 may optimize `if (&foo) foo();` to unconditional `foo();`.

#### Branch relocations

Resolves to the current instruction, the next instruction, or relative zero address, depending on the architecture and somtimes the relocation type.

For `-no-pie`, there is no dynamic relocation. The behaviors are:

* aarch64: GNU ld: rewrite the instruction to a NOP; ld.lld: branch to the next instruction
* mips: GNU ld: jump to the start of the text segment (?)
* ppc32: GNU ld: rewrite the instruction to a NOP; ld.lld: branch to the current instruction
* ppc64: GNU ld: rewrite the instruction to a NOP; ld.lld: branch to the current instruction
* riscv: GNU ld: branch to the absolute zero address; ld.lld: branch to the current instruction ([D103001](https://reviews.llvm.org/D103001))
* i386/x86_64: GNU ld/ld.lld: branch to the link-time zero address

The aarch64 ABI says

> On platforms that do not support dynamic pre-emption of symbols, an unresolved weak reference to a symbol relocated by R_<CLS>_CALL26 shall be treated as a jump to the next instruction (the call becomes a no-op). The behaviour of R_<CLS>_JUMP26 and R_<CLS>_PLT32 in these conditions is not specified by this standard.

To the best of my knowledge, on other ABIs the behaviors are mostly unspecified.

#### GOT-generating relocations

The linker may or may not emit a dynamic relocation `R_*_GLOB_DAT` for the GOT entry.
If there is no `R_*_GLOB_DAT`, the GOT entry is always zero at runtime.
If there is an `R_*_GLOB_DAT`, the GOT entry may be non-zero if an immediately loaded shared object defines the symbol at runtime.

GNU ld's x86 port tries to be smart: if there is at least one GOT-generating or PLT-generating relocation and `-z dynamic-undefined-weak` (enabled by default) is in effect, generate an `R_*_GLOB_DAT`.
For `-fpie` and `-fpic` code there are generally `R_*_GLOB_DAT` in the linker output; for `-fno-pic` code there are no.

ppc supports `-z {,no}dynamic-undefined-weak` since 2021-05.

For static PIE (indicated by the option `--no-dynamic-linker`), they think there should be no dynamic relocations.
Actually glibc' `--enable-static-pie` support relies on this (fragile as I think) property, e.g. the `__pthread_mutex_lock` reference in `_dl_add_to_namespace_list`, the `__pthread_initialize_minimal` reference in `csu/libc-start.c`.

ld.lld generates `R_*_GLOB_DAT` in `-pie` and `-shared` modes.

#### Remarks

Targets have different opinions on whether dynamic relocations should be emitted.
GNU ld x86 can disable the relocation with `-z dynamic-undefined-weak` (<https://sourceware.org/bugzilla/show_bug.cgi?id=19636>).
I am on the fence of its usefullness.
I think the condition "if there is at least one GOT-generating or PLT-generating relocation" is unnecessary complexity.

ld.lld's approach is overall simpler.
ld.lld's dynamic `-pie` behavior is usually different from GNU ld. Portable code should not depend on whether there is an `R_*_GLOB_DAT`.
ld.lld's current behavior has inconsistency regarding absolute relocations and GOT-generating relocations.
For the example below, ld.lld generates an `R_*_GLOB_DAT` but suppresses the absolute relocation.

```c
// https://bugs.llvm.org/show_bug.cgi?id=50759
extern __attribute__((weak)) int weak_reference;
__attribute__((visibility("hidden"))) int* address_of_weak_reference = &weak_reference;

void _start() {
  if (&weak_reference)
    weak_reference = 1;
  if (address_of_weak_reference)
    *address_of_weak_reference = 1;
}
```

I think we may consider keeping the absolute relocations to improve consistency.
I do not know whether `-z dynamic-undefined-weak` may be of usefulness.

About `-z {,no}dynamic-undefined-weak`, I am of the opinion that it is not useful.
Its implication is much more complex than what the help message can explain, and users can unlikely use it correctly.

## ld.so

A `STB_GLOBAL` definition and `STB_WEAK` definition in the dynamic symbol table is equivalent in glibc ld.so and musl ld.so.
If the symbol lookup finds a `STB_WEAK` definition, it will stop and return that symbol, instead of searching shared objects.
glibc before 2.2 provided a different behavior: a `STB_WEAK` definition could be overridden by a subsequent `STB_GLOBAL` definition.

FreeBSD ld.so still uses the legacy glibc behavior.
<https://reviews.freebsd.org/D26352> implemented the Linux behavior under the environment variable `LD_DYNAMIC_WEAK=1`.

A weak reference in the dynamic symbol table does not cause a symbol lookup error if no definition is found.

## PDP-11 object file format

_PDP-11 MACRO-11 Language Reference Manual_ mentions the `.WEAK` directive.

> When the .WEAK directive specifies a symbol that is externally defined, it is
> considered a global symbol. If the linker finds the symbol's definition in
> another module, it uses that definition. If the linker does not find an
> external definition, the symbol is given a value of 0. The linker does not
> search a library for the global symbol, but if a module brought in from a
> library for another reason contains the symbol's definition, the linker uses
> that definition.
>
> If a symbol that is defined in the current module is specified by the
> .WEAK directive, the symbol is considered globally defined. However,
> if the current module is inserted in an object library, the symbol is
> not inserted in the library's symbol table. Consequently, the module
> is not found when the library is searched at link time to resolve the
> symbol.

Note that its weak directive for an externally defined symbol is very close to the semantics of ELF weak references.
It may be the origin of weak symbols.

I do not understand the semantics of its weak definition semantics, though.

## BSD a.out

In 1994, Paul Kranenburg (pk) added weak symbol support to NetBSD flavored a.out binary format.
See <https://github.com/NetBSD/src/commit/8e0a22a5fb5d6662c8c3aae3e09ea86cf621f82f>.
It was considered to be better than indirect symbols (`N_INDR`).

The previously unused field `n_other` is used in a way similar to ELF `st_other`: `N_AUX` takes the least significant 4 bits while `N_BIND` takes the remaining 4 bits.

## Mach-O

Unlike ELF, a weak reference can extract an archive member to satisfy the weak reference.

A weak dylib symbol can extract an archive member as well.

`.weak_definition` sets `N_WEAK_DEF`. This is used by weak definitions (common/linkonce/linkonce_any/weak/weak_any).
In ld64, `N_WEAK_DEF` is ignored for an absolute symbol (`N_ABS`).

When multiple weak definitions are combined, if all inputs are private extern, the output will be private extern only and not be exported.
If any symbol has the `N_NO_DEAD_STRIP` bit, the output will not be dead stripped.

`.weak_def_can_be_hidden` (Mac OS X 10.6) sets `N_WEAK_DEF` and `N_WEAK_REF`.
LLVMAsmPrinter sets this for `linkonce_odr` definitions which are `local_unnamed_addr` (except non-constant `GlobalVariable`) or `unname_addr`.
In the linker, this is similar to `N_PEXT` (PrivateExtern). If all instances are PrivateExtern or `.weak_def_can_be_hidden`, the symbol will not be exported.
The symbol will not be in the weak bind table.
In ld64, a `weak_def_can_be_hidden` definition can override a `N_PEXT` definition.

Weak binding happens in a flat namespace.

A weak definition in the executable takes precedence over a strong definition in a dylib.
A weak definition in a dylib may lose to a strong definition in a subsequent dylib.

## PE/COFF

PE/COFF does not have a direct counterpart but can emulate a weak definition with an `IMAGE_SYM_CLASS_WEAK_EXTERNAL` `IMAGE_SYM_UNDEFINED` symbol with a defined `IMAGE_SYM_CLASS_EXTERNAL` auxiliary symbol (named `.weak.<weaksymbol>.<relatedstrongsymbol>` in GNU).
If the symbol does not have a regular definition in another object file, the linker will select the auxiliary definition.

The auxiliary symbol is defined. Care is needed to reduce the possibility of causing duplicate definition errors.
([link.exe requires that the axuliary symbol to be external](https://reviews.llvm.org/D75989).)
MinGW and LLVM derive a unique name `.weak.$name.default.$something` to such a weak definition.
A defined external symbol in the same object file is a good candidate for `$something`.
LLVM prefers a non-comdat defined external symbol (<https://reviews.llvm.org/D75989>).

There are three main linkers: MSVC `link.exe`, GNU ld (PE/COFF port), and lld.
When named like "lld-link", lld acts like `link.exe`.
When named like `ld.lld`, lld acts like GNU ld's PE/COFF port with a PE emulation (e.g. `i386pe, i386pep, thumb2pe, arm64pe`).

`link.exe` reports a duplicate definition error with two weak definitions.
Both GNU ld and `lld-link -lld-allow-duplicate-weak` (or `-lldmingw`) allow this following the ELF behavior.

```sh
printf '__attribute__((weak)) int def() { return 0; } int main() { def(); }' > a.c
printf '__attribute__((weak)) int def() { return 1; } void unused() {}' > b.c
x86_64-w64-mingw32-gcc -c a.c b.c
x86_64-w64-mingw32-gcc a.o b.o     # ok, no "multiple definition" error
wine a.exe                         # exit code is 0
```

link.exe supports `/alternatename:` which can achieve similar effects. The option specifies a fallback definition for a symbol. If the symbol is otherwise undefined, the linker will pick up the fallback definition.

link.exe does not support linking together two weak definitions with the GNU `.weak.*` naming scheme: `fatal error LNK1227: conflicting weak extern definition for '?weak@@YAXXZ'.  New default '.weak.?weak@@YAXXZ.default.?foo@@YAXXZ' conflicts with old default '.weak.?wea....`

link.exe does not support overriding a weak definition with a strong definition if `.pdata` exists: `fatal error LNK1223: invalid or corrupt file: file contains invalid .pdata contributions`

```sh
printf '__attribute__((weak)) void undef_weak_fun(); int main() { if (&undef_weak_fun) undef_weak_fun(); }' > a.c
llvm-objdump -t a.o          # .weak.undef_weak_fun.main
x86_64-w64-mingw32-gcc a.c   # ok, no "undefined reference" error
wine a.exe
```

lld-link allows resolving an undefined reference to a weak definition. GNU ld rejects the case due to <https://sourceware.org/PR9687>.
```sh
printf 'void f(); int main() { f(); }' > a.c
printf '__attribute__((weak)) void f() {} void unused() {}' > b.c
x86_64-w64-mingw32-gcc -c a.c b.c
```
```
% x86_64-w64-mingw32-gcc a.o b.o
/usr/bin/x86_64-w64-mingw32-ld: /tmp/ccyRH2ap.o:a.c:(.text+0xe): undefined reference to `f'
collect2: error: ld returned 1 exit status
```

If we change `f` in `a.c` to a weak reference (add `__attribute__((weak))`), GNU ld will accept the link like lld-link.

## XCOFF

XCOFF supports weak definitions through weak external symbols (`C_WEAKEXT`).
Weak references are not supported.

`-qnoweakexp` asks AIX ld to not export weak external symbols. This achieves an effect similar to `-fvisibility-inlines-hidden`.
