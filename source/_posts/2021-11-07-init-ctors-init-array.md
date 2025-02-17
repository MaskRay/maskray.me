layout: post
title: .init, .ctors, and .init_array
author: MaskRay
tags: [binutils,glibc,linker]
---

Updated in 2023-03.

In C++, dynamic initializations for non-local variables happen before the first statement of the `main` function.
All (most?) implementations just ensure such dynamic initializations happen before `main`.

As an extension, GCC supports `__attribute__((constructor))` which can make an arbitrary function run before `main`.
A constructor function can have an optional priority (`__attribute__((constructor(N)))`).

<!-- more -->

Priorities from 0 to 100 are reserved for the implementation (`-Wprio-ctor-dtor` catches violation), e.g. gcov uses `__attribute__((destructor(100)))`.
Applications can use 101 to 65535. 65535 (`.init_array` or `.ctors`, without a suffix) has the same priority as a non-local variable's dynamic initialization in C++.

```cpp
struct S { S(); };
S& s = *new S;
pid_t pid = getpid();

__attribute__((constructor(101))) void init101() { puts("init101"); }
__attribute__((constructor(102))) void init102() { puts("init102"); }
__attribute__((constructor)) void init() { puts("init65535"); }

int main(void) { return 0; }
```

Under the hood, on ELF platforms, the initialization functions or constructors are implemented in two schemes.
The legacy one uses `.init`/`.ctors` while the new one uses `.init_array`.

```asm
.section	.text.startup,"ax",@progbits
_GLOBAL__sub_I_a.cc:
  callq	_Znwm
  callq	_ZN1SC1Ev
  callq	getpid

.section	.init_array.101,"aw",@init_array
## legacy: .section .ctors.65434,"aw",@progbits
.p2align	3
.quad	_Z7init101v

.section	.init_array.102,"aw",@init_array
## legacy: .section .ctors.65433,"aw",@progbits
.p2align	3
.quad	_Z7init102v

.section	.init_array,"aw",@init_array
## legacy: .section .ctors,"aw",@progbits
.p2align	3
.quad	_Z4initv
.quad	_GLOBAL__sub_I_a.cc
```

## `.init` and `.fini`

System V release 4 introduced the dynamic tags `DT_INIT` and `DT_FINI` to implement ELF initialization and termination functions.
Today it is difficult to figure out what it actually did, but it was likely similar to the GCC scheme described below.

On a GCC+glibc system, traditionally the section `.init` in an executable/shared object consisted of four fragments:

```text
glibc crti.o:(.init) _init
GCC crtbegin.o:(.init)    # not existent on modern systems
GCC crtend.o:(.init)      # not existent on modern systems
glibc crtn.o:(.init)
```

The linker combines `.init` input sections and places the fragments into the `.init` output section.
`_init` is defined at offset 0 in the first input section, so its address equals the address of the `.init` output section.
The linker defines `DT_INIT` according to the value of `_init` (which can be changed by the `-init` linker option).
In the absence of `.dynamic`, `DT_INIT` does not exist. The runtime references the symbol `_init`.

`.fini` is similar:
```text
glibc crti.o:(.fini) _fini
GCC crtbegin.o:(.fini)    # not existent on modern systems
GCC crtend.o:(.fini)      # not existent on modern systems
glibc crtn.o:(.fini)
```

The linker defines `DT_FINI` according to the value of `_fini` (which can be changed by the `-fini` linker option).

In glibc x86-64, `sysdeps/x86_64/crti.S` and `sysdeps/x86_64/crtn.S` provide the definitions for `crti.o` and `crtn.o`:

```text
# crti.o
Disassembly of section .init:

0000000000000000 <_init>:
       0: 48 83 ec 08                   subq    $8, %rsp
       4: 48 8b 05 00 00 00 00          movq    (%rip), %rax  # b <_init+0xb>
                0000000000000007:  R_X86_64_REX_GOTPCRELX       __gmon_start__-0x4
       b: 48 85 c0                      testq   %rax, %rax
       e: 74 02                         je      0x12 <_init+0x12>
      10: ff d0                         callq   *%rax

Disassembly of section .fini:

0000000000000000 <_fini>:
       0: 48 83 ec 08                   subq    $8, %rsp

# crtn.o
Disassembly of section .init:

0000000000000000 <.init>:
       0:       addq    $8, %rsp
       4:       retq

Disassembly of section .fini:

0000000000000000 <.fini>:
       0:       addq    $8, %rsp
       4:       retq
```

`crti.o` calls `__gmon_start__` (gmon profiling system) if defined. This is used by `gcc -pg`.

musl just provides empty `crti.o` and `crtn.o`.

## `.ctors` and `.dtors`

In GCC `libgcc/crtstuff.c`, when `__LIBGCC_INIT_ARRAY_SECTION_ASM_OP__` is not defined and `__LIBGCC_INIT_SECTION_ASM_OP__` is defined (`HAVE_INITFINI_ARRAY_SUPPORT` is 1 in `$builddir/gcc/auto-host.h`), the following scheme is used.
Note: the condition is not satisfied on modern systems.

C++ dynamic initializations and `__attribute__((constructor))` do not use `_init` directly.
They are implemented as ELF functions. The addresses are collected in the `.ctors` section which will be called by the runtime.
Assume that we have one object files `a.o` and `b.o` with `.ctors` sections with different priorities, the layout of the `.ctors` output section is:

```text
crtbegin.o:(.ctors) __CTOR_LIST__
a.o:(.ctors) b.o:(.ctors)
a.o:(.ctors.00001) b.o:(.ctors.00001)
a.o:(.ctors.00002) b.o:(.ctors.00002)
...
a.o:(.ctors.65533) b.o:(.ctors.65533)
a.o:(.ctors.65534) b.o:(.ctors.65534)
...
crtend.o:(.ctors) __CTOR_LIST_END__
```

`.dtors` is similar:
```text
crtbegin.o:(.dtors) __DTOR_LIST__
a.o:(.dtors) b.o:(.dtors)
a.o:(.dtors.00001) b.o:(.dtors.00001)
a.o:(.dtors.00002) b.o:(.dtors.00002)
...
a.o:(.dtors.65533) b.o:(.dtors.65533)
a.o:(.dtors.65534) b.o:(.dtors.65534)
...
crtend.o:(.dtors) __DTOR_LIST_END__
```

* `crtbegin.o` defines `.ctors` and `.dtors` with one element, -1 (0xffffffff on 32-bit platforms and 0xffffffffffffffff on 64-bit platforms).
* `crtend.o` defines `.ctors` and `.dtors` with one element, 0.
* `crtend.o` defines a `.init` section which calls `__do_global_ctors_aux`. `__do_global_ctors_aux` calls the static constructors in the `.ctors` section. The -1 and 0 sentinels are skipped.
* `crtbegin.o` defines a `.fini` section which calls `__do_global_dtors_aux`. `__do_global_dtors_aux` calls the static constructors in the `.dtors` section. The -1 and 0 sentinels are skipped.

### Reversed execution order

Here is an interesting property: `.ctors` elements are run in the reversed order and `.dtors` elements are run in the regular order.
E.g. for `a.o:(.ctors) b.o:(.ctors)`, b.o's constructor runs before a.o's.

This is to make dynamic linking similar to static linking for `.ctors` sections without a suffix (having the lowest priority).

The origin may be related to a generic ABI promise: if a.so depends on b.so, then b.so's constructors run first.
If we only look at `.ctors` sections without a suffix, the behavior of `ld main.o a.so b.so` may be quite similar to the static linking `ld main.o a.a b.a`.

`.dtors` can be seen as undoing `.ctors`, so its order is the reverse of `.ctors`, which is the regular order.

## `.init_array` and `.fini_array`

HP-UX developers noticed that the `.init`/`.ctors` scheme have multiple problems:

* Fragmented `_init` function is ugly and error-prone.
* Sentinel values in `.ctors` are ugly.
* `.init` and `.ctors` use magic names instead of dedicated section types.

They invented `DT_INIT_ARRAY` as an alternative.
glibc implemented the scheme in [1999](https://sourceware.org/git/?p=glibc.git;a=commit;h=fcf70d4114db9ff7923f5dfeb3fea6e2d623e5c2). The GCC and binutils implementations were also quite old.

FreeBSD added support in [2012-03](https://github.com/freebsd/freebsd-src/commit/95d1e3d11bebe5f0d75da30af98fbc94ee0e5233).
OpenBSD added support in [2016-08](https://github.com/openbsd/src/commit/86fa57a2792c6374b0849dd7b818a11e676e60ba).
NetBSD made `DT_INIT_ARRAY` available for all ports in [2018-12](https://github.com/NetBSD/src/commit/17c588ef35d9a33496f4b96ea6a68802c10e0028).

glibc and BSD implementations call the constructors with `argc, argv, environ` while musl's calls the constructors with no argument.

In this scheme, `.init_array` and `.init_array.N` sections have a dedicated type `SHT_INIT_ARRAY`.
`crtbegin.o` and `crtend.o` do not provide fragments.

Below is a layout.

```text
a.o:(.init_array.1) b.o:(.init_array.1)
a.o:(.init_array.2) b.o:(.init_array.2)
...
a.o:(.init_array.65533) b.o:(.init_array.65533)
a.o:(.init_array.65534) b.o:(.init_array.65534)
a.o:(.init_array) b.o:(.init_array)
```

Note: `ctors_priority = 65535-init_array_priority`

The linker defines `DT_INIT_ARRAY` and `DT_INIT_ARRAYSZ` according to the address and size of `.init_array`.
The linker also defines `__init_array_start` and `__init_array_end` if referenced.
The pair of symbols can be used by a statically linked position dependent executable which may not have `.dynamic`.

Unlike `.ctors`, the execution order of `.init_array` is forward (follows `.init`).
`a.o:(.init_array) b.o:(.init_array)` has a different order from `a.o:(.ctors) b.o:(.ctors)`.
In a future section we will discuss that this difference can expose a type of very subtle bugs called "static initialization order fiasco".

In GCC, newer ABI implementations like AArch64 and RISC-V only use `.init_array` and don't provide `.ctors`.

### `.preinit_array`

The linker defines `DT_PREINIT_ARRAY` and `DT_PREINIT_ARRAYSZ` according to the address and size of `.preinit_array`.
The linker also defines `__preinit_array_start` and `__preinit_array_end` if referenced.

The generic ABI says:

> DT_PREINIT_ARRAY: This element holds the address of the array of pointers to pre-initialization functions, discussed in ``Initialization and Termination Functions'' below. The DT_PREINIT_ARRAY table is processed only in an executable file; it is ignored if contained in a shared object.

`DT_PREINIT_ARRAY` only applies to the executable.
This feature gives the executable a way to run initialization functions before shared object dependencies.

There is no `.postfini_array`.

Most ld.so implementations support `DT_PREINIT_ARRAY`.
musl does not support the feature. See [add preinit_array support](https://www.openwall.com/lists/musl/2016/05/17/2).

## Runtime behavior

The generic ABI says:

> If an object contains both DT_INIT and DT_INIT_ARRAY entries, the function referenced by the DT_INIT entry is processed before those referenced by the DT_INIT_ARRAY entry for that object. If an object contains both DT_FINI and DT_FINI_ARRAY entries, the functions referenced by the DT_FINI_ARRAY entry are processed before the one referenced by the DT_FINI entry for that object.

If the executable `a` depends on `b.so` and `c.so` (in order), the glibc ld.so and libc behavior is:

* If any DSO specifies `DT_1_INITFIRST`, the last such DSO to be loaded has its `DT_INIT`/`DT_INIT_ARRAY` run and skips the associated execution below.
* `ld.so` runs `a:DT_PREINIT_ARRAY`
* `ld.so` runs `c.so:DT_INIT`. The crtbegin.o fragment of `_init` calls `.ctors`
* `ld.so` runs `c.so:DT_INIT_ARRAY`
* `ld.so` runs `b.so:DT_INIT`. The crtbegin.o fragment of `_init` calls `.ctors`
* `ld.so` runs `b.so:DT_INIT_ARRAY`
* `libc_nonshared.a` runs `a:DT_INIT`. The crtbegin.o fragment of `_init` calls `.ctors`
* `libc_nonshared.a` runs `a:DT_INIT_ARRAY`

As a new ABI, glibc's RISC-V port doesn't define `ELF_INIT_FINI`, so `DT_INIT` does not run.

Here is a test for the execution order of `atexit` and `DT_FINI_ARRAY`.
```sh
printf > a.c %s '
#include <stdio.h>
#include <stdlib.h>

void hook() { puts("atexit"); }
void fini() { puts("fini"); }
__attribute__((section(".fini_array"), used)) static void *use_fini = &fini;

__attribute__((constructor)) void ctor() { atexit(hook); }
int main() {}
'
printf > b.c %s '
#include <stdio.h>
#include <stdlib.h>

void hook_b() { puts("atexit b"); }
__attribute__((constructor)) void ctor_b() { atexit(hook_b); }

__attribute__((used,retain)) void fini_b() { puts("fini b"); }
asm(".pushsection .fini_array,\"aw\",@fini_array; .quad fini_b; .popsection");
'
printf %s '
.MAKE.MODE := meta curdirOk=1

a: a.c b.so
        ${CC} -Wl,--no-as-needed -Wl,-rpath=. $> -o $@

b.so: b.c
        ${CC} -fpic -shared $> -o $@
' | sed 's/        /\t/' > ./Makefile
```

musl ensures that atexit registered hooks run before `DT_FINI_ARRAY`.
```text
# musl
% ./a
atexit
atexit b
fini
fini b

# glibc, FreeBSD
% ./a
atexit
fini
fini b
atexit b
```

## `.ctors` to `.init_array` transition

In 2010-12, Mike Hommey filed [Replace .ctors/.dtors with .init_array/.fini_array on targets supporting them](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=46770) which I believe was related to his [ELF hack](https://glandium.org/blog/?p=1177) work for Firefox.

Switching sections needed to consider backward compatibility: how to handle old object files using `.ctors` sections.
H.J. Lu proposed that the internal linker script of GNU ld could be changed to place `.ctors .ctors.N .init_array .init_array.N` input sections into the `.init_array` output section in [`RFC: Support mixing .init_array.* and .ctors.* input sections`](https://sourceware.org/pipermail/binutils/2010-December/070524.html).

With this GNU ld support, GCC 4.7 made the switch.

---

```cpp
//--- a.cc
#include <stdio.h>
__attribute__((constructor)) void init() { puts("init"); }
extern "C" void ctors() { puts("ctors"); }
__attribute__((section(".ctors"), used)) static auto *use_ctors = ctors;
void preinit() { puts("preinit"); }
__attribute__((section(".preinit_array"), used)) static auto *use_preinit = &preinit;

int main() {}

//--- b.cc
#include <stdio.h>
__attribute__((constructor)) void init_b() { puts("init b"); }
extern "C" void ctors_b() { puts("ctors b"); }
__attribute__((section(".ctors"), used)) static auto *use_ctors_b = ctors_b;

//--- c.cc
#include <stdio.h>
__attribute__((constructor)) void init_c() { puts("init c"); }
extern "C" void ctors_c() { puts("ctors c"); }
__attribute__((section(".ctors"), used)) static auto *use_ctors_c = ctors_c;

//--- d.cc
#include <stdio.h>
__attribute__((constructor)) void init_d() { puts("init d"); }
extern "C" void ctors_d() { puts("ctors d"); }
__attribute__((section(".ctors"), used)) static auto *use_dtors_d = ctors_d;

//--- run
clang -fpic -shared b.cc -o b.so
clang -fpic -shared c.cc -o c.so
clang -fpic -shared d.cc -o d.so
clang -fuse-ld=bfd a.cc ./b.so ./c.so ./d.so -o a
```

```sh
% split-file a.txt dir && cd dir
% sh run
% readelf -d a
... (NEEDED)  Shared library: [./b.so]
... (NEEDED)  Shared library: [./c.so]
... (NEEDED)  Shared library: [./d.so]
% ./a
preinit
ctors d
init d
ctors c
init c
ctors b
init b
ctors
init

% clang -fpic -shared d.cc ./b.so -o d.so  # make d.so depend on b.so
% ./a
preinit
ctors b
init b
ctors d
init d
ctors c
init c
ctors
init
```

split-file is a LLVM utility.
You may link one or more DSOs with `-z initfirst` for `DF_1_INITFIRST` and observe the new output.

gold doesn't have the concept of an internal linker script.
Ian Lance Taylor added the enabled-by-default linker option [`--ctors-in-init-array`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=487b39dfdd6cdfcac1124f2bcacd70a2da92f242) to emulate the GNU ld behavior

Since `.ctors` is rare, ld.lld does not implement converting `.ctors` into `.init_array`.

You can link the output with a linker script fragment:
```
OVERWRITE_SECTIONS {
  .init_array : { *(SORT_BY_INIT_PRIORITY(.init_array.*)) *(.init_array) *(SORT_BY_INIT_PRIORITY(.ctors.*)) *(.ctors) }
}
```

## GCC vs Clang

GCC's `.ctors`/`.init_array` choice is a configure option `--enable-initfini-array`.

Clang uses a CC1 option `-fno-use-init-array`.
This makes cross compilation and testing multiple targets in one build convenient.

In the llvm-project supported toolchains, only MinGW and PlayStation 4 still use `.ctors` for the latest version.
For MinGW, this is related to the fact that PE/COFF does not have section types and the MinGW runtime doesn't have the `.init` pain, so there isn't motivation for a switch.
For PlayStation 4, it is presumably related to the fact that PlayStation 4 uses a modified FreeBSD 9 image. I saw `.ctors` patches to llvm-project in 2021.

## Linux remnant of `.ctors` in 2021

If you don't use prebuilt object files from GCC<4.7, it is difficult to see `.ctors` on Linux in 2021.
However, I found two exceptions.

First, a libgcc file for the split stack implementation had `.ctors.65535` assembly code.
I filed [`morestack.S should support .init_array.0 besides .ctors.65535`](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99759) which was fixed in 2021-10.

Second, GCC cross compilers targeting Linux did not enable `--enable-initfini-array`.
H.J. Lu reported [`--enable-initfini-array should be enabled for cross compiler to Linux`](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100896) and fixed it for GCC 12.
This affected GCC 11 builds by `scripts/build-many-glibcs.py`

## Manually convert `.ctors` to `.init_array`

Run `objcopy --rename-section .ctors=.init_array --rename-section .dtors=.fini_array $file`.

Rarely, `.ctors.$x` may be present. Convert such a section to `.init_array.$((65535-$x))`.

## C++ dynamic initialization

In a typical C++ object, most `.init_array` elements are dynamic initializations, so I will spend some paragraphs describing it.

The standard defines the order for various initializations.

* Constant initialization and zero initialization
* Dynamic initialization
* `main`
* Deferred dynamic initialization (e.g. optimized out, on-demand shared library)

Dynamic initialization has three types with different degrees of order guarantee:

* Unordered dynamic initialization (static data members and variable templates not explicitly specialized)
* Partially-ordered dynamic initialization (inline variables that are not an implicitly or explicitly instantiated specialization)
* Ordered dynamic initialization (other non-local variables)

Basically, in one translation unit, the order of dynamic initializations usually matches the intuition, e.g. `a`'s initializations happen before `b`'s below.

```cpp
struct A { A(); };

namespace testing {
A& a = *new A;
}

static A b;
```

### C++ static initialization order fiasco

> [basic.start.dynamic]: If V and W have ordered initialization and the definition of V is appearance-ordered before the definition of W, or if V has partially-ordered initialization, W does not have unordered initialization, and for every definition E of W there exists a definition D of V such that D is appearance-ordered before E, then ... V is sequenced before the initialization of W  ... otherwise <b>the initializations of V and W are indeterminately sequenced.</b>

If no _appearance-ordered_ relationship is defined, we say that two initializations are indeterminately sequenced.
Relying on a particular order is called "static initialization order fiasco".
(I don't know what "static" refers to. Perhaps it refers to static variables or static storage duration.)

Below is a registry example.
The order that a, b, and C are registered depends on the link order.
If somehow only one order works, than the program may be brittle.

```cpp
// registry.cc
Registry registry;
// a.cc
Register a;
// b.cc
Register b;
// c.cc
Register c;
```

Fixing such bugs requires thoughts on the initialization order.
Basically one needs to do one of the following:

* constant initialization
* lazy initialization (dynamic initialization of function-locale static, llvm::ManagedStatic, etc)
* manual initialization
* Nifty Counter idiom

Some ways to prevent static initialization order fiasco:

* constexpr
* constinit (constexpr - const)
* `clang -Wglobal-constructors`: `warning: declaration requires a global constructor [-Wglobal-constructors]`

AddressSanitizer `check_initialization_order` is enabled by default due to `strict_init_order`.
It enforces that a dynamic initialization does not touch memory regions of other global variables.
Unfortunately in practice it misses many many cases.

### `ld.lld --shuffle-sections=.init_array=-1`

In ld.lld, Rafael Espindola added `--shuffle-sections` motivated by making tests stabilized.
I changed the option to apply to `.init_array`/`.init_array` as well and later changed it to the current form:
`--shuffle-sections=<section-glob>=<seed>`: shuffle matched input sections using the given seed before mapping them to the output sections.
I specialized the seed value -1 to mean the deterministic reversed order.

You can specify `--shuffle-sections=.init_array=-1 --shuffle-sections=.fini_array=-1` to reverse the input section order.
It's unclear whether `.fini_array` needs to be reversed as well, but it is safe to do so.
This does not change `.init_array.N` and `.fini_array.N`, but in practice static initialization order fiasco from prioritized sections are rare.

In a linker script, you can use the `REVERSE` keyword (<https://reviews.llvm.org/D145381>) in ld.lld.

In practice, most static initialization order fiasco bugs are due to the order between two translation units.
For a mostly statically linked executable, testing the regular order and the reversed order is sufficient to catch such bugs.

However, for a program with many shared objects, the checks may be insufficient.
An executable and all its `DT_NEEDED` transitive dependencies form a dependency graph.
The generic ABI requirement "if a.so depends on b.so, then b.so's initialization functions run first" imposes some orders in the graph.

If the executable `a` depends on `b.so` and `c.so`, and `b.so` and `c.so` are unrelated.
We may consider it a bug if `c.so`'s initialization functions need to run before `b.so`, but the linker cannot do anything to improve the checks.
There is also no ld.so feature altering the order.
A manual way is to change the link order of `b.so` and `c.so`, but it may be difficult to do so in a build system.
