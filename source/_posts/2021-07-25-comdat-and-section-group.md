layout: post
title: COMDAT and section group
author: MaskRay
tags: [binutils,llvm,linker]
---

## Vague linkage

In C++, inline functions, template instantiations, and a few other things can be defined in multiple object files but need deduplication at link time.
In the dark ages the functionality was implemented by weak definitions: the linker does not report duplicate definition errors and resolves the references to the first definition.
The downside is that unneeded copies remained in the linked image.

<!-- more -->

## COMDAT

In the Microsoft PE file format, COMDAT was invented. It has two major features: (1) enabling deduplication of a set of sections (2) allowing a set of sections to be retained or discarded as a unit.

The section flag (`IMAGE_SCN_LNK_COMDAT`) marks a section COMDAT and enables deduplication on a per-section basis.

The first symbol that has the section value of the COMDAT section must be the section symbol.
The second symbol is called "the COMDAT symbol" and is used by the linker in conjunction with the `Selection` field in its section definition auxiliary record.
The `Selection` field describes how the associated section should be deduplicated.

`IMAGE_COMDAT_SELECT_ANY` is the most common selection value. "Any section that defines the same COMDAT symbol can be linked; the rest are removed."
link.exe and ld.lld just pick the first seen section.
Similar selection kinds include: `IMAGE_COMDAT_SELECT_SAME_SIZE`, `IMAGE_COMDAT_SELECT_EXACT_MATCH`, and `IMAGE_COMDAT_SELECT_LARGEST`.

`IMAGE_COMDAT_SELECT_ASSOCIATIVE` makes a section linked to another section. The interrelated sections will be retained or discarded as a unit.
Conceptually there is a leader and an arbitrary number of followers.
We will see that in ELF section groups the sections are equal.

If a text section needs a data section and deduplication is needed for both sections, you have two choices:

* Use two COMDAT symbols. There is the drawback that deduplication happens independently for the interconnected sections.
* Make the data section link to the text section via `IMAGE_COMDAT_SELECT_ASSOCIATIVE`. Whether an `IMAGE_COMDAT_SELECT_ASSOCIATIVE` section is retained is dependent on its referenced section.

`IMAGE_COMDAT_SELECT_NODUPLICATES` is interesting.
The implementation just does not perform deduplication.
If a section defines an external symbol and the section is duplicated in another object file, there will be a duplicate definition linker error.

### link.exe limitation

There is a limitation: MSVC link.exe may report a duplicate symbol error (`error LNK2005`) for an external symbol defined in a `IMAGE_COMDAT_SELECT_ASSOCIATIVE` section, even if it would be discarded after handling the leader symbol.

## ELF section groups

In the GNU world, `.gnu.linkonce.` was invented to deduplicate groups with just one member.
`.gnu.linkonce.` has been long obsoleted in favor of section groups but the usage has been daunting until 2020.
Adhemerval Zanella removed the last live glibc use case for `.gnu.linkonce.` [BZ #20543](http://sourceware.org/PR20543).

Cary Coutant et al implemented section groups and `GRP_COMDAT` on HP-UX before pushing the idea to the generic ABI committee.
The design was a generalization of PE COMDAT.
You can find the document _IA-64 gABI Issue 72: COMDAT_ on the Internet. It lists some design consideration.

> Some sections occur in interrelated groups. For example, an out-of-line definition of an inline function might require, in addition to the section containing its executable instructions, a read-only data section containing literals referenced, one or more debugging information sections and other informational sections. Furthermore, there may be internal references among these sections that would not make sense if one of the sections were removed or replaced by a duplicate from another object. Therefore, such groups must be included or omitted from the linked object as a unit. A section cannot be a member of more than one group.

You may want to read the description before reading the following paragraphs.

According to "such groups must be included or omitted from the linked object as a unit",
a linker's garbage collection feature must retain or discard the sections as a unit.
This property can be leveraged by interrelated metadata sections.

Say `__llvm_prf_data` and `__llvm_prf_cnts` are used as interrelated metadata sections describing the text section `.text.a`.
`__llvm_prf_data` has a relocation to `__llvm_prf_cnts`. `.text.a` and other text sections reference `__llvm_prf_cnts`.
Without the "as a unit" semantics, `ld --gc-sections` may retain `__llvm_prf_cnts` while discarding `__llvm_prf_data`, since `__llvm_prf_data` is not referenced by a live section.
The "as a unit" semantics allow `__llvm_prf_data` to be retained due to its interrelation with `__llvm_prf_cnts`.

A section group can contain more than 2 members.
E.g. if the function in `.text.a` uses value profiling, there will be a third metadata section `__llvm_prf_vnds`, which is not referenced.
The "as a unit" semantics allow `__llvm_prf_vnds` to be retained if `__llvm_prf_data` or `__llvm_prf_cnts` is retained.

> The section data of a SHT_GROUP section is an array of Elf32_Word entries. The first entry is a flag word. The remaining entries are a sequence of section header indices.

### `GRP_COMDAT`

The most common section group flag is `GRP_COMDAT`, which makes the member sections similar to COMDAT in Microsoft PE file format, but can apply to multiple sections.
(The committee borrowed the name "COMDAT" from PE.)

> `GRP_COMDAT` - This is a COMDAT group. It may duplicate another COMDAT group in another object file, where duplication is defined as having the same group signature. In such cases, only one of the duplicate groups may be retained by the linker, and the members of the remaining groups must be discarded.

The signature symbol provides a unique name for deduplication.
The binding (local/global status) of the group signature is not part of the equation.
This is not clear in the specification and has been clarified by [GRP_COMDAT group with STB_LOCAL signature](https://groups.google.com/g/generic-abi/c/2X6mR-s2zoc).

In [D106782](https://reviews.llvm.org/D106782), I changed llvm-objcopy to clear the `GRP_COMDAT` bit if the signature symbol becomes local.
`--localize-hidden` and `--keep-global-symbol` are sometimes used to localize symbols.
If the signature symbol of a section group is localized, the intention is likely to localize the section group and suppress deduplication.
GNU objcopy feature request: [PR27931](https://sourceware.org/bugzilla/show_bug.cgi?id=27931).

I want to highlight one thing GCC does (and Clang inherits) for backward compatibility: definitions relative to a COMDAT section are usually `STB_WEAK` instead of `STB_GLOBAL`.
The idea is that an old toolchain which does not recognize COMDAT groups can still operate correctly, just in a degraded manner.

The section group flag can be 0: no signature based deduplication should happen.
In LLVM IR, use `comdat nodeduplicate` to generate such a COMDAT group.

In LLVM, `llvm/lib/Linker` performs COMDAT deduplication for regular LTO. The deduplication is only performed if the COMDAT key has a non-local linkage.

#### Referencing a local symbol from outside the group

The generic ABI specifies:

> A symbol table entry with STB_LOCAL binding that is defined relative to one of a group's sections, and that is contained in a symbol table section that is not part of the group, must be discarded if the group members are discarded. References to this symbol table entry from outside the group are not allowed.

In the following example, the local symbol `foo` is used as the COMDAT group signature. The group in `b.o` is discarded per the COMDAT deduplication rule. `b.o:.metadata` is a reference outside the prevailing group (the group within `a.o`), which is rejected by LLD.

```sh
cat > ./a.s <<'eof'
.globl _start
_start:
  call .text.foo

.section .text.foo,"axG",@progbits,foo,comdat
foo:
  ret
eof
cat > ./b.s <<'eof'
.section .text.foo,"axG",@progbits,foo,comdat
foo:
  nop
  ret

.section .metadata,"a"
  .quad foo
eof
clang -c a.s b.s
```

```
% ld.lld a.o b.o
ld.lld: error: relocation refers to a symbol in a discarded section: foo
>>> defined in b.o
>>> referenced by b.o:(.metadata+0x0)
```

This rule detects misuse of local symbols that are discarded during COMDAT deduplication.
When `b.o:.text.foo` is discarded, the `.metadata` relocation loses its meaningful target.
Since undefined local symbols are invalid, we cannot preserve an undefined `foo` in the output.

Moreover, this rule serves as a one-definition-rule violation detector.
The symbols `a.o:foo` and `b.o:foo` may behave differently, and `a.o:.text.foo` might not even define `foo` at all.
The `.metadata` relocation depends on local translation unit information for `foo`, which becomes invalid when `b.o:.text.foo` is discarded.

Note that if `foo` is `STB_WEAK` or `STB_GLOBAL`, it would remain defined in the output file, and the `.metadata` relocation would bind to the `foo` defined in the prevailing `.text.foo` section in `a.o`. 

GCC's `-fpatchable-function-entry` was fixed to emit multiple sections. See (Chinese) [从-fpatchable-function-entry=N[,M]说起](https://maskray.me/blog/2020-02-01-fpatchable-function-entry) for detail.

(The error will go away if `.metadata` changes to a non-SHF_ALLOC section as LLD doesn't scan relocations from non-SHF_ALLOC sections.)

#### Group signature localization

If a non-local symbol (e.g. `__llvm_retpoline_r11`) is defined relative to a section in a group, and referenced from outside the section group, it cannot be localized.

The compiler may generate some functions in a COMDAT group, e.g. `__x86.get_pc_thunk.bx` and `__x86.get_pc_thunk.cx` by GCC and `__llvm_retpoline_r11` by Clang x86-64 with retpoline.
Let's see an example of `__llvm_retpoline_r11`.

```sh
cat > ./a.s <<'eof'
.globl _start
_start:
  call __llvm_retpoline_r11
  call foo

.section .text.__llvm_retpoline_r11,"axG",@progbits,__llvm_retpoline_r11,comdat
.hidden __llvm_retpoline_r11
.weak __llvm_retpoline_r11
__llvm_retpoline_r11:
  ret
eof
cat > ./b.s <<'eof'
.globl foo
foo:
  call __llvm_retpoline_r11

.section .text.__llvm_retpoline_r11,"axG",@progbits,__llvm_retpoline_r11,comdat
.hidden __llvm_retpoline_r11
.weak __llvm_retpoline_r11
__llvm_retpoline_r11:
  ret
eof
clang -c a.s b.s
ld.lld a.o b.o  # good
```

If we localize `__llvm_retpoline_r11` in `b.o`, the COMDAT group should drop the `GRP_COMDAT` flag to avoid an error.
As GNU objcopy keeps the `GRP_COMDAT` flag, there is a reference (from `foo` in `.text`) from outside the group, leading to an error.
```
% objcopy -G foo b.o bb.o
% ld.lld a.o bb.o
ld.lld: error: relocation refers to a symbol in a discarded section: __llvm_retpoline_r11
>>> defined in bb.o
>>> referenced by bb.o:(.text+0x1)
% llvm-objcopy -G foo b.o bb.o
% ld.lld a.o bb.o
```

The `llvm-objcopy` output works because the `GRP_COMDAT` flag is dropped.
However, the linker output has two copies of `__llvm_retpoline_r11`, which is benign but not great.
The fault falls on the user side: when using `llvm-objcopy --keep-global-symbol` to localize most symbols, be aware to keep all COMDAT signature symbols global.

```
% llvm-objcopy -G foo -G __llvm_retpoline_r11 b.o bb.o
% ld.lld a.o bb.o
```

## LLVM IR

In LLVM IR, [comdat](https://llvm.org/docs/LangRef.html#comdats) provides access to object file COMDAT/section group functionality which represents interrelated sections.
The syntax is closer to ELF section groups in spirit.
I recently added clarification that "a comdat must be included or omitted as a unit."

For example, the following IR defines a comdat named `foo` and its two members `foo` and `bar`.

```llvm
$foo = comdat any
@foo = global i32 2, comdat
@bar = global i32 3, comdat($foo)
```

In PE/COFF, the comdat must contain a member which has the same name as the group.
The member (leader) will encode the IR selection kind and the other members will use the `IMAGE_COMDAT_SELECT_ASSOCIATIVE` selection.

### `llvm.global_ctors` and `llvm.global_dtors`

On ELF, `llvm.global_ctors` elements are usually placed in `.init_array` and `.init_array.*` sections.

> If the third field is non-null, and points to a global variable or function, the initializer function will only run if the associated data from the current module is not discarded. On ELF the referenced global variable or function must be in a comdat.

The third field was initially added for PE/COFF so that linker garbage collection can discard an initialization section if its referenced text section can be discarded.
If the referenced text section is in a `comdat any`, the initialization section can be deduplicated with other translation units.

On ELF, the third field is implemented as a section group to enable deduplication.

As an example, `-fsanitize-coverage=` instrumentation generates:
```llvm
$sancov.module_ctor_trace_pc_guard = comdat any

@llvm.global_ctors = appending global [1 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 2, void ()* @sancov.module_ctor_trace_pc_guard, i8* bitcast (void ()* @sancov.module_ctor_trace_pc_guard to i8*) }]
@llvm.used = appending global [1 x i8*] [i8* bitcast (void ()* @sancov.module_ctor_trace_pc_guard to i8*)], section "llvm.metadata"

define internal void @sancov.module_ctor_trace_pc_guard() #1 comdat {
  call void @__sanitizer_cov_trace_pc_guard_init(i32* @__start___sancov_guards, i32* @__stop___sancov_guards)
  ret void
}
```

The module constructor `sancov.module_ctor_trace_pc_guard` is in an IR comdat.
The `llvm.global_ctors` element compiles to a `.init_array.2` in the same section group as `.text.sancov.module_ctor_trace_pc_guard`.
```text
% llvm-readelf -g a.o
COMDAT group section [    6] `.group' [sancov.module_ctor_trace_pc_guard] contains 4 sections:
   [Index]    Name
   [    7]   .text.sancov.module_ctor_trace_pc_guard
   [    8]   .rela.text.sancov.module_ctor_trace_pc_guard
   [   11]   .init_array.2
   [   12]   .rela.init_array.2
```

`.init_array.2` has the type `SHT_INIT_ARRAY`. Such a section within a group is unusual.
To the best of my knowledge, such sanitizer usage was not leveraged by GCC/Clang before.

Before Clang 13.0.0, the `SHT_INIT_ARRAY` member in a group was a GC root.
It ensured the module constructor could not be discarded by linker GC.
This is considered as an abuse (see [GC-ability of SHT_INIT_ARRAY](https://groups.google.com/g/generic-abi/c/TpleUEkNoQI)).
I placed the module constructor in `llvm.used` so that `.text.sancov.module_ctor_trace_pc_guard` has the `SHF_GNU_RETAIN` flag.

ld.lld doesn't treat `SHT_INIT_ARRAY` sections with the `SHF_LINK_ORDER` flag as GC roots.
Linkers do not treat `SHT_NOTE` sections within a group as GC roots.
For consistency, I made `SHT_INIT_ARRAY` in a section group GCable for ld.lld 13.0.0.

### WebAssembly

`comdat nodeduplicate` is not implemented. I filed a feature request <https://github.com/llvm/llvm-project/issues/49875>.

It strikes me that WebAssembly need `WASM_COMDAT_DATA`, `WASM_COMDAT_FUNCTION`, and `WASM_COMDAT_SECTION` to encode COMDAT.
It is difficult for an elf to understand the complexity:)

### Suppression of non-inline interprocedural optimizations

Global variables and functions in a COMDAT typically almost always have the `linkonce_odr` or `weak_odr` linkage.
See <https://github.com/llvm/llvm-project/issues/27148>: non-inline interprocedural optimizations generally need to be suppressed.

```cpp
//--- common.h
#include <stdint.h>
bool always_false(int);
__attribute__((noinline)) inline int maybe_divide(int *ptr) {
  int val = 500 / ptr[0];
  return always_false(val) ? val : int(intptr_t(ptr));
}

//--- a.cc
#include "common.h"
bool always_false(int) { return false; }  // Make maybe_divide optimized
int fa(int *ptr) {
  ptr[0] = 10; int k = maybe_divide(ptr); ptr[0] = 20;
  return k;
}

//--- b.cc
#include "common.h"
int fb(int *ptr) {  // Unused
  ptr[0] = 10; int k = maybe_divide(ptr); ptr[0] = 20;
  return k;
}

//--- main.cc
#include <stdio.h>
int fa(int *ptr);
int main() {
  int *ptr = new int[1];
  ptr[0] = 0;
  printf("%d\n", fa(ptr));
  delete[] ptr;
}
```


In `a.cc`, the function `maybe_divide` could be optimized to `__attribute__((noinline)) inline int maybe_divide(int *ptr) { return int(intptr_t(ptr)); }`.
This optimization would eliminate a memory load and a division, but it would not be safe to infer the `readnone nounwind` attributes for `maybe_divide`.
If we did so, in `fa`, `int k = maybe_divide(ptr);` could potentially be reordered before `ptr[0] = 10;`.
If `b.cc`'s unoptimized version is prevailing (e.g. `clang++ main.cc b.cc a.cc`), this reordering would result in a division-by-zero exception at run-time.

To avoid the issue, the resolution states that a global value with one of the linkages `{linkonce,weak}{,_odr}, available_externally, external_weak, common`, as well as non-dsolocal `external` linkage, is considered to have a non-exact definition and will suppress inter-procedural optimizations.

Additionally, the resolution to [llvm-project#48366](https://github.com/llvm/llvm-project/issues/48366) states that a `nobuiltin` `external` function is also considered to have a non-exact definition: as if a `builtin` call site may call the `nobuiltin` function or an unnamed library function, suppressing assumptions the call site can make (if the call site does resolve to the `nobuiltin` function, its behavior may be different from a library implementation).

We can enable non-inline inter-procedural optimizations for `linkonce_odr` or `weak_odr` linkage functions by first internalizing the profitable ones, and then attempting to merge them during LTO.

## C inline

`-fgnu89-inline` inline and C99 inline are different from C++ inline.
Non-static non-extern inline has weird semantics.
I think it may be because the designers wanted to control the defining translation unit and avoid vague linkage.
The choices did avoid the need for COMDAT but made the non-static non-extern inline inconvenient to use.
In practice I think most projects avoid it.

## Case study - PGO/coverage

A function and its data/counter variables must use separate IR comdat.
(a)
Instrumentation runs before inlining.
More functions may reference the function's data/counter.
Using one comdat can lead to dangling references.

(b)
Using separate comdat can allow benign ODR violation where the prevailing copy is from a non-instrumented module.

### Make `FunctionPointer` relative

Using a private alias to the function does not allow this scenario: both a non-instrumented TU and a instrumented TU have a function in a comdat any. If the non-instrumented TU is prevailing, a relocation referencing the discarded private alias will cause a linker error.

Q: How to handle an `available_externally` function? (e.g. `extern __inline __attribute__((gnu_inline)) int stat(const char *path, struct stat *statbuf)`)

An alias cannot reference an `available_externally` function.

### Static variable inside inline function emitted in a different section group than the function

It seems that GCC only places a function in a COMDAT of a different name for aliases (`symtab_node::add_to_same_comdat_group` is only called for aliases).

For the following code (adapted from <https://github.com/llvm/llvm-project/issues/72361>),
```cpp
struct A {
  char data[24];
#if __cplusplus >= 202002L
  constexpr
#endif
    A() noexcept : data() {}
};

inline const A &get() {
  static const A empty;
  return empty;
}

const A &(*emitGet)() = &get;

// clang++ -c -O2 -std=c++17 a.cc -o a17.o
// clang++ -c -O2 -std=c++20 a.cc -o a20.o
```

The function `get` and the variable `empty` are in different COMDAT groups.
This design choice allows `get` and `empty` to be discarded separately.
Usually, if all references to `get` are inlined (except in this address-taken case), `get` can be discarded while `empty` is retained.

In a C++17 object file, Clang generates LLVM IR `@_ZZ3getvE5empty = linkonce_odr dso_local global %struct.A zeroinitializer, comdat, align 1` and LLVM places variable in the section `.bss._ZZ3getvE5empty`.
`get` initializes `_ZZ3getvE5empty` to all zeros (although the modification can be optimized out, given that this is the only modification and BSS implies all zeros).

In a C++20 object file, the constructor is `constexpr` and Clang generates LLVM IR `@_ZZ3getvE5empty = linkonce_odr dso_local constant %struct.A zeroinitializer, comdat, align 1`.
LLVM places the variable in the section `.rodata._ZZ3getvE5empty`.

The primary difference is that the section containing the variable is mutable without `constexpr` and read-only with `constexpr`.
When mixing C++17 and C++20 object files, if C++17 `get` is selected and C++20 `empty` is selected, C++17 `get` will attempt to write to `.rodata._ZZ3getB5cxx11vE5emptyB5cxx11`, resulting in a segmentation fault.
