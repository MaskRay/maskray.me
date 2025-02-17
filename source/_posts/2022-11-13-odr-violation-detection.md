layout: post
title: ODR violation detection
author: MaskRay
tags: [linker,llvm]
---

This article describes how to detect C++ [One Definition Rule](https://en.cppreference.com/w/cpp/language/definition) (ODR) violations.
There are many good resources on the Internet about how ODR violations can introduce subtle bugs, so I will not repeat that here.

<!-- more -->

## Debug information based

### gold `--detect-odr-violations`

In [2007](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=a55ce7febfaa52670ce3d9c236d3033de80ac091), the gold linker implemented an option `--detect-odr-violations` to detect ODR violation based on debug information.
This option collects symbols starting with `_Z` and finds two `STB_WEAK` definitions with different `st_size` or different `st_type` values.
These symbols are candidates of ODR violation.

gold parses DWARF line tables.
For a candidate, if both its definitions have associated line table information (if any definition does not have debug info, no warning) and disjoint file:line sets. If yes, gold issues a warning.

The check uses source locations as a proxy as an ODR violation. The proxy is usually good but not precisely ODR violation.
The first line of a function may change among relocatable object files due to different optimization behaviors. And we may see spurious ODR violations.
The check does not find differing class/enum definitions and templates.

The option uncovered some Chromium bugs, see [https://crbug.com/449754](https://bugs.chromium.org/p/chromium/issues/detail?id=449754).

The feature is not implemented in other linkers.
This idea of using debug information is interesting. See "Future direction" for a non-debug-information alternative, which may possibly be better.

A debug information based approach is potentially slow. As an analogy, constructing `.gdb_index` is easily the slowest path in the linker.

### adobe/orc

<https://github.com/adobe/orc> is a tool for finding violations of C++'s One Definition Rule on the OSX toolchain.
It wraps libtool/ld on macOS to collect input object files and parses debugging information entries and verifies that certain attributes match.

As a standalone program, it can focus on user interface.

[Finding ODR Violations with ORC](https://accu.org/video/spring-2022-day-2/brereton-thomason/) was a talk on ACCU 2022.

## LTO based

### GCC `-Wlto-type-mismatch` and `-Wodr`

`-Wlto-type-mismatch` is an enabled-by-default warning for LTO about mismatched types of types (structs, function signatures, etc).
See `gcc/lto/lto-symtab.cc:warn_type_compatibility_p`.

```sh
echo 'struct A { int x; } a; int main(void) {}' > a.c
echo 'struct A {}; extern struct A a; struct A *b = &a;' > b.c
```
```
% gcc -flto -fuse-ld=bfd a.c b.c
b.c:1:30: warning: type of ‘a’ does not match original declaration [-Wlto-type-mismatch]
    1 | struct A {}; extern struct A a; struct A *b = &a;
      |                              ^
a.c:1:21: note: ‘a’ was previously declared here
    1 | struct A { int x; } a; int main(void) {}
      |                     ^
a.c:1:21: note: code may be misoptimized unless ‘-fno-strict-aliasing’ is used
```

C++ has an additional diagnostic `-Wodr` about mismatched types of C++ global declarations.

GCC also reports `-Wodr` diagnostics for mismatched external linkage enumeration types.
([lld/ELF example](https://github.com/llvm/llvm-project/issues/83529))
```sh
echo 'enum A {X} a; int main() {}' > a.cc
echo 'enum A {Y} b;' > b.cc
```
```
% g++ -fuse-ld=bfd -flto a.cc b.cc
a.cc:1:8: warning: type ‘struct A’ violates the C++ One Definition Rule [-Wodr]
    1 | struct A {int x;} a; int main() {}
      |        ^
b.cc:1:8: note: a different type is defined in another translation unit
    1 | struct A {} b;
      |        ^
a.cc:1:15: note: the first difference of corresponding definitions is field ‘x’
    1 | struct A {int x;} a; int main() {}
      |               ^
b.cc:1:8: note: a type with different number of fields is defined in another translation unit
    1 | struct A {} b;
      |        ^
```

```sh
echo 'enum A {X} a; int main() {}' > a.cc
echo 'enum A {Y} b;' > b.cc
```

```
% g++ -fuse-ld=bfd -flto a.cc b.cc
a.cc:1:6: warning: type ‘A’ violates the C++ One Definition Rule [-Wodr]
    1 | enum A {X} a; int main() {}
      |      ^
b.cc:1:6: note: an enum with different value name is defined in another translation unit
    1 | enum A {Y} b;
      |      ^
a.cc:1:9: note: name ‘X’ differs from name ‘Y’ defined in another translation unit
    1 | enum A {X} a; int main() {}
      |         ^
b.cc:1:9: note: mismatching definition
    1 | enum A {Y} b;
      |         ^
```

```
% g++ -fuse-ld=bfd -flto a.cc b.cc
a.cc:1:8: warning: type ‘struct A’ violates the C++ One Definition Rule [-Wodr]
    1 | struct A {int x;} a; int main() {}
      |        ^
b.cc:1:8: note: a different type is defined in another translation unit
    1 | struct A {} b;
      |        ^
a.cc:1:15: note: the first difference of corresponding definitions is field ‘x’
    1 | struct A {int x;} a; int main() {}
      |               ^
b.cc:1:8: note: a type with different number of fields is defined in another translation unit
    1 | struct A {} b;
      |        ^
```

It cannot detect the case when a member function is changed from inline to non-inline.

```sh
cat > a.cc <<'eof'
struct A { virtual ~A() {} };
A *fa() { return new A; }
int main() {}
eof
cat > b.cc <<'eof'
struct A { virtual ~A(); };
A::~A() {}
A *fb() { return new A; }
eof
g++ -flto -fuse-ld=bfd a.cc b.cc
```

`-Wodr` does not detect `inline int foo() { return 1; }` vs `inline int foo() { return 2; }`.

See [block:618550 ODR](https://bugs.gentoo.org/buglist.cgi?quicksearch=block%3A618550%20ODR&list_id=6592901) for Gentoo's bug list.

[Implement -Wlto-type-mismatch](https://github.com/llvm/llvm-project/issues/56487) is an llvm-project feature request.

### ThinLTO

[ThinLTO](https://clang.llvm.org/docs/ThinLTO.html) computes a module summary for functions, global variables, aliases, and ifuncs.
Technically the module summary can be overloaded to record ODR hashes, but coupling this with an optimization-targeted feature seems weird and adds size overhead even when the feature is not used.

## Clang ODR hash

In [2017](https://reviews.llvm.org/D21675), Clang implemented an AST-based ODR hash feature. Each definition is given a hash value.
When definitions are merged, the hash values are compared and an error is reported if mismatching.

This feature works with both Clang header modules and C++ modules.

```sh
echo 'module B { header "B.h" } module C { header "C.h" }' > module.modulemap
echo '#include "B.h"\n#include "C.h"\nint main() { return foo(); }' > A.cc
echo 'inline int foo() { return 1; }' > B.h
echo 'inline int foo() { return 2; }' > C.h

clang++ -c -fmodules A.cc
```

`-fmodules` implies `-fimplicit-modules` to load `module.modulemap`. The two `#include` directives are translated to module loads.
When `foo` in B and C are merged, an error is issued.

```text
In module 'C' imported from A.cc:2:
./C.h:1:12: error: 'foo' has different definitions in different modules; definition in module 'C' first difference is function body
inline int foo() { return 2; }
~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~
./B.h:1:12: note: but in 'B' found a different body
inline int foo() { return 1; }
~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~
```

Let's see an example of C++ modules.

```sh
echo 'import B; import C; int main() { return foo(); }' > A.cc
echo 'export module B; export inline int foo() { return 1; }' > B.ccm
echo 'export module C; export inline int foo() { return 2; }' > C.ccm

clang++ -std=c++20 --precompile B.ccm -o B.pcm
clang++ -std=c++20 --precompile C.ccm -o C.pcm
clang++ -std=c++20 -fprebuilt-module-path=. A.cc B.pcm C.pcm -o A
```

```text
In file included from A.cc:1:
/tmp/d/C.ccm:1:36: error: 'foo' has different definitions in different modules; definition in module 'C' first difference is function body
export module C; export inline int foo() { return 2; }
                        ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~
/tmp/d/B.ccm:1:36: note: but in 'B' found a different body
export module B; export inline int foo() { return 1; }
                        ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~
```

## Compiler instrumentation based

### `#pragma detect_mismatch`

This pragma is supported by MSVC and Clang.
See <https://devblogs.microsoft.com/oldnewthing/20160803-00/?p=94015>.
This is implemented using the linker option `/failifmismatch:`.

In ELF, this feature can be emulated with `SHN_ABS` symbols.
GNU ld and lld do not report a duplicate definition error when two `SHN_ABS` symbols have the same value.

### AddressSanitizer `detect_odr_violation`

For an instrumented translation unit, there is a global constructor which calls `__asan_register_globals` to register some types of global variables (non-thread-local, defined, `external/private/internal` LLVM linkage, and a few other conditions).
This can be used to check whether two global variables of the same name are defined in different modules.
In [2014](https://github.com/llvm/llvm-project/commit/e91930a7e6786e0f3fa314c66203996210ea2929), `detect_odr_violation` was implemented for this idea.
Note: functions and vague linkage symbols are not instrumented, so the interesting case is skipped.

#### Poisoning based detection

The runtime poisons the red zone of a to-be-registered global variable (`compiler-rt/lib/asan/asan_globals.cpp`).
If the variable was poisoned when attempting a registration, it means that the variable has been registered by another linkage unit.
The runtime will report an ODR violation error.

```sh
echo 'int var; int main() { return var; }' > a.cc
echo 'long var;' > b.cc
clang++ -fpic -fsanitize=address -fno-sanitize-address-use-odr-indicator -shared b.cc -o b.so
clang++ -fsanitize=address -fno-sanitize-address-use-odr-indicator a.cc ./b.so -o a
```

```
% ./a
=================================================================
==2573423==ERROR: AddressSanitizer: odr-violation (0x55ddebde3200):
  [1] size=4 'var' a.cc
  [2] size=8 'var' b.cc
These globals were registered at these points:
  [1]:
    #0 0x55ddeb348f9a in __asan_register_globals /b/s/w/ir/cache/builder/src/third_party/llvm/compiler-rt/lib/asan/asan_globals.cpp:356:3
    #1 0x55ddeb409b4e in asan.module_ctor a.cc
    #2 0x7f7f27029332 in call_init csu/../csu/libc-start.c:145:3
    #3 0x7f7f27029332 in __libc_start_main csu/../csu/libc-start.c:376:5

  [2]:
    #0 0x55ddeb348f9a in __asan_register_globals /b/s/w/ir/cache/builder/src/third_party/llvm/compiler-rt/lib/asan/asan_globals.cpp:356:3
    #1 0x7f7f27776cfe in asan.module_ctor b.cc
    #2 0x7f7f27781ced in call_init elf/dl-init.c:70:3
    #3 0x7f7f27781ced in call_init elf/dl-init.c:26:1

==2573423==HINT: if you don't care about these errors you may set ASAN_OPTIONS=detect_odr_violation=0
SUMMARY: AddressSanitizer: odr-violation: global 'var' at a.cc
==2573423==ABORTING
```

The default `detect_odr_violation=2` mode additionally disallows symbol interposition on variables.
Change `long` in `b.cc` to `int` and we will still see an `odr-violation` error.
`detect_odr_violation=1` suppresses errors if the registered variable is of the same size.

```
% ASAN_OPTIONS=detect_odr_violation=1 ./a
% ASAN_OPTIONS=detect_odr_violation=2 ./a
=================================================================
==2574052==ERROR: AddressSanitizer: odr-violation (0x562d39db1200):
...
```

This approach has a drawback when a global variable is defined in a non-instrumented TU and an instrumented TU, and the linker selects the non-instrumented TU.

The variable metadata references the interposable variable symbol.
If an instrumented global variable is interposed by an uninstrumented one, the runtime may poison bytes not belonging to the global variable.
Since poisoning writes to shadow memory, this is usually benign.
However, global variable instrumentation increases the alignment of a global variable (to at least 32) and checks that the metadata-referenced variable symbol has an alignment of at least shadow granularity (8).
If the referenced variable symbol resolves to a non-instrumented module, the alignment check may fail (if the symbol is less aligned) and in this case the runtime reports a bogus odr-violation error as well.

Let's see an example. I add a dummy variable to make `var` not aligned by 8 in `a.o` (no guarantee but working in practice).

```sh
echo 'char pad, var; int main() { return pad + var; }' > a.cc
echo 'char var;' > b.cc
clang++ -fpic -fsanitize=address -fno-sanitize-address-use-odr-indicator -shared b.cc -o b.so
clang++ -c a.cc -o a.o
clang++ -fsanitize=address a.o ./b.so -o a
```

```
% ./a
==2890036==The following global variable is not properly aligned.
==2890036==This may happen if another global with the same name
==2890036==resides in another non-instrumented module.
==2890036==Or the global comes from a C file built w/o -fno-common.
==2890036==In either case this is likely an ODR violation bug,
==2890036==but AddressSanitizer can not provide more details.
=================================================================
==2890036==ERROR: AddressSanitizer: odr-violation (0x000000cf3869):
  [1] size=1 'var' /tmp/d/a.cc:1
  [2] size=1 'var' /tmp/d/a.cc:1
...
```

#### ODR indicator

<http://reviews.llvm.org/D15642> introduced a new mode: for a variable `var`, a one-byte variable `__odr_asan_gen_var` is created with the original linkage (essentially only `external`).
If `var` is defined in two instrumented modules, their `__odr_asan_gen_var` symbols reference to the same copy due to symbol interposition.
When registering `var`, set the associated `__odr_asan_gen_var` to 1. The runtime checks whether `__odr_asan_gen_var` is already 1, and if yes, the variable has an ODR violation.

To prevent the metadata-referenced symbol from interposed to another linkage unit, create a private alias for `var` to be referenced in the metadata.
This ensures that the metadata refers to the self copy.

```sh
echo 'int var; int main() { return var; }' > a.cc
echo 'long var;' > b.cc
clang++ -fpic -fsanitize=address -fsanitize-address-use-odr-indicator -shared b.cc -o b.so
clang++ -fsanitize=address -fsanitize-address-use-odr-indicator a.cc ./b.so -o a
```

For Clang 16, I landed <https://reviews.llvm.org/D137227> to use `-fsanitize-address-use-odr-indicator` by default for non-Windows targets.

<https://reviews.llvm.org/D127911> changed the ODR indicator symbol name to `__odr_asan_gen_$demangled`.

### KCFI

Clang has recently implemented a indirect call control flow integrity instrumentation which does not require link-time optimization: KCFI.
One side product of this feature is related to ODR violation detection.

For an address-taken function, a weak absolute symbol `__kcfi_typeid_<function>` is defined.
The symbol is weak. But imagine we use a `STB_GLOBAL` symbol, a linker can find differing values.
GNU ld has a hack that duplicate absolute definitions do not trigger an error and ld.lld has ported the behavior.
While such a scheme would work, using magic symbols is not proper usage of a linker and I would object to such an attempt.

### MSVC LNK2022

In MSVC, the `/clr` switch seems to insert some metadata. The linker is able to report `LNK2022: metadata operation failed`.

## lld --no-allow-shlib-undefined

[DSO undef and non-exported definition](/blog/2023-10-31-dso-undef-and-non-exported-definition) describes a case when a relocatable object file provides a non-exported definition and a DSO provides another definition.

## lld COMDAT resolution

While implementing parallel section initialization for ld.lld, I [changed ld.lld symbol resolution](https://reviews.llvm.org/D120626) to disregard COMDAT resolution (COMDAT resolution was moved to a later pass).
This combined with AddressSanitizer `-fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator` turns out to catch some violations of classes with vtables.
This is an interesting side product I did not anticipate.

With `-fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator`, a global variable is placed in a COMDAT group so that the global variable (`var`) along with the metadata (`__asan_gen_var`) can be discarded by the linker if unreferenced.
The associated ODR indicator is not in a COMDAT.

```sh
cat > a.cc <<'eof'
struct A { virtual ~A() {} };
A *fa() { return new A; }
int main() {}
eof
cat > b.cc <<'eof'
struct A { virtual ~A(); };
A::~A() {}
A *fb() { return new A; }
eof
clang++ -c -fsanitize=address -fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator a.cc b.cc
```

```llvm
; b.ll
$_ZTV1A = comdat any
@_ZTV1A = dso_local constant { { [4 x ptr] }, [32 x i8] } { { [4 x ptr] } { [4 x ptr] [ptr null, ptr @_ZTI1A, ptr @_ZN1AD1Ev, ptr @_ZN1AD0Ev] }, [32 x i8] zeroinitializer }, comdat, align 32
@___asan_gen_.1 = private unnamed_addr constant [13 x i8] c"vtable for A\00", align 1
@__asan_global__ZTV1A = private global { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @0 to i64), i64 32, i64 64, i64 ptrtoint (ptr @___asan_gen_.1 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 ptrtoint (ptr @"__odr_asan_gen_vtable for A" to i64) }, section "asan_globals", comdat($_ZTV1A), !associated !0
```

```
% clang++ -fsanitize=address -fuse-ld=lld a.o b.o
ld.lld: error: relocation refers to a symbol in a discarded section: vtable for A
>>> defined in /tmp/b-8ed614.o
>>> section group signature: _ZTV1A
>>> prevailing definition is in /tmp/a-2f04ec.o
>>> or the symbol in the prevailing group had STB_WEAK binding and the symbol in a non-prevailing group had STB_GLOBAL binding. Mixing groups with STB_WEAK and STB_GLOBAL binding signature is not supported
>>> referenced by a.cc
>>>               /tmp/a-2f04ec.o:(A::A())
>>> the vtable symbol may be undefined because the class is missing its key function (see https://lld.llvm.org/missingkeyfunction)
...
% readelf -Ws a.o | grep _ZTV1A
    12: 0000000000000000    32 OBJECT  WEAK   DEFAULT   16 _ZTV1A
% readelf -g a.o
...
COMDAT group section [   15] `.group' [_ZTV1A] contains 2 sections:
   [Index]    Name
   [   16]   .data.rel.ro._ZTV1A
   [   17]   .rela.data.rel.ro._ZTV1A
...
% readelf -Ws b.o | grep _ZTV1A
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT   14 .data.rel.ro._ZTV1A
    24: 0000000000000000    64 OBJECT  GLOBAL DEFAULT   14 _ZTV1A
% readelf -g b.o
...
COMDAT group section [   13] `.group' [_ZTV1A] contains 4 sections:
   [Index]    Name
   [   14]   .data.rel.ro._ZTV1A
   [   15]   .rela.data.rel.ro._ZTV1A
   [   24]   asan_globals
   [   25]   .relaasan_globals
...
```

In `a.o`, the vtable for `A` has vague linkage and compiles to a weak symbol `_ZTV1A` in a COMDAT.
In `b.o`, the vtable for `A` has regular external linkage and compiles to a `STB_GLOBAL` symbol `_ZTV1A`.
Normally `b.o:_ZTV1A` is not in a COMDAT but in `-fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator` mode the symbol is in a COMDAT.

During linking, as `a.o` precedes `b.o`, the COMDAT group in `a.o` is prevailing while the COMDAT group in `b.o` is non-prevailing.
With lld's new symbol resolution rule, `b.o:_ZTV1A` overrides `a.o:_ZTV1A` but later the definition is discarded (the symbol becomes undefined) because it is in a non-prevailing COMDAT.
A relocation referencing the noew undefined `_ZTV1A` will cause an error.

There are similar errors for `typeinfo for A` and `typeinfo name for A` unless `-fno-rtti` is specified.

If we make the COMDAT with the `STB_GLOBAL` symbol prevailing, the errors will be suppressed.
```
clang++ -fsanitize=address -fuse-ld=lld b.o a.o
```

Since `typeinfo` and `typeinfo name` can trigger this error, we can change `a.cc` to not have a vtable. Just use `typeid`.

```sh
cat > a.cc <<'eof'
#include <stdio.h>
#include <typeinfo>
struct A {};
void ext(const char *);
int main() { puts(typeid(A).name()); }
eof
echo 'struct A { virtual ~A(); }; A::~A() {}' > b.cc
clang++ -c -fsanitize=address -fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator a.cc b.cc
```

```
% clang++ -fuse-ld=lld -fsanitize=address a.o b.o
ld.lld: error: relocation refers to a symbol in a discarded section: typeinfo for A
>>> defined in b.o
>>> section group signature: _ZTI1A
>>> prevailing definition is in a.o
>>> or the symbol in the prevailing group had STB_WEAK binding and the symbol in a non-prevailing group had STB_GLOBAL binding. Mixing groups with STB_WEAK and STB_GLOBAL binding signature is not supported
>>> referenced by a.cc
>>>               a.o:(main)
>>> referenced by b.cc
>>>               b.o:(vtable for A)

ld.lld: error: relocation refers to a symbol in a discarded section: typeinfo name for A
>>> defined in b.o
>>> section group signature: _ZTS1A
>>> prevailing definition is in a.o
>>> or the symbol in the prevailing group had STB_WEAK binding and the symbol in a non-prevailing group had STB_GLOBAL binding. Mixing groups with STB_WEAK and STB_GLOBAL binding signature is not supported
>>> referenced by a.cc
>>>               a.o:(.data.rel.ro._ZTI1A+0x8)
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

## Summary

There are several ways we can categorize these tools.

* run-time analysis: AddressSanitizer `detect_odr_violation`
* static analysis: the others

By scope:

* single translation unit: Clang ODR hash
* linkage unit: most tools
* cross linkage unit: AddressSanitizer `detect_odr_violation`

By entity:

* GCC `-Wlto-type-mismatch` and `-Wodr`: type
* gold `--detect-odr-violations`: C++ vague linkage functions
* lld with `-fsanitize-address-globals-dead-stripping -fsanitize-address-use-odr-indicator`: C++ vague linkage functions and variables: (partial)
* adobe/orc: many
* Clang ODR hash: almost all

## Future direction

Having an approach for LTO is nice, but we probably want an approach usable without LTO or debug information.

As Clang has implemented the heavylifting work of ODR hashes, we can implement a feature to collect the hashes into a custom section.
We can change lld to scan this section and find differing values.

See [RFC: ODR checker for Clang and LLD](https://discourse.llvm.org/t/rfc-odr-checker-for-clang-and-lld/45148) for a 2017 RFC.
The effort added `SHT_LLVM_ODRTAB` and was not upstreamed.

I think the section can be a table holding ODR hash values. Each value is associated with a `R_*_NONE` relocation referencing the associated symbol table entry.
For classes which do not produce a symbol (i.e. no vtable), the compiler can generate a symbol solely for ODR violation detection.

I will probably implement this feature if people find this useful.
