---
layout: post
title: C++ standard library ABI compatibility
author: MaskRay
tags: [c++,gcc]
---

Updated in 2023-11.

For a user who only uses one C++ standard library, such as libc++, there are typically three compatibility goals, each with increasing compatibility requirements:

* Can the program, built with a specific version of libc++, work with an upgraded libc++ shared object (DSO)?
* Can an executable and its DSOs be compiled with different versions of libc++ headers?
* Can two relocatable object files, compiled with different versions of libc++ headers, be linked into the same executable or DSO?

If we replace "different libc++ versions" with a mixture of libc++ and libstdc++, we encounter additional goals:

* Can the program, built with a specific version of libstdc++, work with an upgraded libstdc++ DSO?
* Can an executable, built with libc++, link against DSOs that were built with libstdc++?
* Can two relocatable object files, compiled with libc++ and libstdc++, or two libstdc++ versions, be linked into the same executable or DSO?

Considering static linking raises another interesting question:

If libc++ is statically linked into `b.so`, can it be used with `a.out` that links against a different version of libc++?
Let's focus on the first three questions, which specifically pertain to libc++.

<!-- more -->

## libc++ ABI stability

libc++ is assigned a version number that corresponds to the major and minor releases of the llvm-project.
Additionally, libc++ offers a target ABI version (`LIBCXX_ABI_VERSION`) as a build-time option, which currently defaults to 1.
`LIBCXX_ABI_VERSION` is used to choose between the stable ABI and the unstable ABI, as explained in the official documentation [libc++ ABI stability](https://libcxx.llvm.org/DesignDocs/ABIVersioning.html).

If we build a program using a specific libc++ version with the stable ABI and link it against the libc++ DSO, upgrading the libc++ DSO should not break the program (assuming the program itself doesn't have any bugs).
However, there are rare cases where libc++ might remove symbols that technically have the potential to cause an ABI break.
These cases usually involve symbols such as debug mode symbols or symbols that only affect certain C++ 2003 programs, and their impact is limited.

In general, the answer to the first two questions (repeated below) is yes:

* Can the program, built with a specific version of libc++, work with an upgraded libc++ shared object (DSO)? Yes.
* Can an executable and its DSOs be compiled with different versions of libc++ headers? Yes, the linked libc++ must be the newest one.

However, certain unusual configurations, like `_LIBCPP_DISABLE_VISIBILITY_ANNOTATIONS`, need to be excluded.

Now, let's consider the problem: when does an ABI break occur?
An ABI break can happen when an executable or DSO uses a symbol that undergoes an ABI change due to a libc++ upgrade.
This symbol can be defined in the libc++ DSO itself or in another DSO that is rebuilt with a new version of libc++.

For each symbol affected by libc++, there is an intention regarding whether it should be exported or not.
Typically, the goal is to minimize the number of exported symbols. Let's discuss these symbols separately.

### Intended exported symbols

The symbols that are intended to be exported include numerous typeinfo/vtable symbols from `_LIBCPP_TEMPLATE_VIS` classes, many `_LIBCPP_EXPORTED_FROM_ABI` symbols, a few `_LIBCPP_CLASS_TEMPLATE_INSTANTIATION_VIS` explicit instantiation definitions, a few `_LIBCPP_METHOD_TEMPLATE_IMPLICIT_INSTANTIATION_VIS` and enum symbols, as well as a few miscellaneous symbols.

libc++ needs to provide ABI compatibility for these symbols within the stable ABI.

Most classes are marked with `_LIBCPP_TEMPLATE_VIS`, which allows the instantiated typeinfo symbols to have default visibility even when using `-fvisibility=hidden` or `-fvisibility-inlines-hidden`.
```
% cat typeid.cc
#include <string>
#include <typeinfo>
const char *foo() { return typeid(std::string).name(); }
% clang -stdlib=libc++ -c -fvisibility=hidden typeid.cc
% readelf -WsC c.o | grep basic_string
     5: 0000000000000000    16 OBJECT  WEAK   DEFAULT    9 typeinfo for std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >
     7: 0000000000000000    63 OBJECT  WEAK   DEFAULT    7 typeinfo name for std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >
```

For exported function symbols (e.g. `std::__1::thread::~thread()` (when `LIBCXX_ABI_VERSION=1`)), they are generally defined in `libcxx/src/**/*.cpp` files.

### Intended non-exported symbols

When our executable or DSO has an undefined symbol that is defined by the libc++ DSO, problems may arise if a new libc++ DSO defines the symbol with an ABI break or no longer defines the symbol.

In the case of DSOs, symbol interposition introduces another consideration: non-local default visibility symbols that are defined.
By default, such symbols are exported to the dynamic symbol table. When the DSO is used with another DSO, the runtime linker (rtld) binds the reference from the second DSO to the first DSO or executable that defines the symbol.

To enable the safe mixing of linked images built with different versions of libc++, libc++ utilizes the concept of "hiding symbols from the ABI."
Let's explore different categories of symbols and how to hide them from the ABI.

In short, libc++ provides the macro macro `_LIBCPP_HIDE_FROM_ABI` for such symbols. The macro consists of at least `__attribute__((__visibility__("hidden"))) __attribute__((__exclude_from_explicit_instantiation__))`.

#### Hide defined symbols

The majority of symbols impacted by libc++ are those generated during the compilation of libc++ headers.
Dealing with these symbols is relatively straightforward. We simply make them hidden using `__attribute__((__visibility__("hidden")))`.

#### Hide symbols due to explicit instantiation declarations

In the case of explicit instantiation declarations, the compiler assumes that the definition exists in another translation unit and may generate undefined symbols.
This behavior corresponds to the `available_externally` linkage in LLVM IR.
The presence of undefined symbols poses a problem because if the definition originates from a translation unit built with a different version of libc++, there may be a mismatch in the ABI.

To address this, starting from Clang 8, the attribute [`__attribute__((exclude_from_explicit_instantiation))`](https://reviews.llvm.org/D51789) is defined.
Applying this attribute to a symbol treats it as not being part of an explicit instantiation declaration.
Consequently, the symbol will exhibit the regular COMDAT behavior (the `linkonce_odr` or `weak_odr` linkage in LLVM IR), where it is either defined or optimized out, ensuring that it never results in an undefined symbol.

```cpp
% cat a.cc
#ifndef ATTR
#define ATTR
#endif
template <class T>
struct Foo {
  ATTR inline void non_static_member_function1();
  ATTR void non_static_member_function2();
  ATTR static int static_data_member;
};

template <class T> inline void Foo<T>::non_static_member_function1() { }
template <class T>        void Foo<T>::non_static_member_function2() { }
template <class T>        int Foo<T>::static_data_member;

extern template struct Foo<int>;

void use() {
  Foo<int> f;
  f.non_static_member_function1();
  f.non_static_member_function2();
  int &odr_use = Foo<int>::static_data_member;
}
% clang++ -c a.cc
% readelf -Ws a.o | grep Foo
     4: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN3FooIiE27non_static_member_function1Ev
     5: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN3FooIiE27non_static_member_function2Ev
     6: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN3FooIiE18static_data_memberE
% clang++ -c -DATTR='__attribute__((exclude_from_explicit_instantiation))' a.cc
% readelf -Ws a.o | grep Foo
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 .text._ZN3FooIiE27non_static_member_function1Ev
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 .text._ZN3FooIiE27non_static_member_function2Ev
     6: 0000000000000000    10 FUNC    WEAK   DEFAULT    5 _ZN3FooIiE27non_static_member_function1Ev
     7: 0000000000000000    10 FUNC    WEAK   DEFAULT    7 _ZN3FooIiE27non_static_member_function2Ev
     8: 0000000000000000     4 OBJECT  WEAK   DEFAULT    9 _ZN3FooIiE18static_data_memberE
```

In cases where `__attribute__((__exclude_from_explicit_instantiation__))` is unavailable, `__attribute__((__always_inline__))` serves as a fallback for older versions of Clang.
However, `always_inline` has certain issues:

* Clang needs to emit a definition regardless of the normal inline decision, leading to increased code size and longer compile times.
* The use of `always_inline` does not guarantee that the function will be inlined.

The primary concern with `always_inline` is that Clang is compelled to expand the large function at every call site, even when a translation unit contains multiple calls to the same member function of a specific instantiation.
On the other hand, with `exclude_from_explicit_instantiation`, the compiler can make a decision to emit a single definition that serves multiple call sites, thereby optimizing code size and compilation time.

Prior to the introduction of `exclude_from_explicit_instantiation`, [`__attribute__((internal_linkage))`](https://reviews.llvm.org/D127444) was used as a better alternative to `always_inline`.
The compiler can emit a single definition that serves multiple call sites, but the definitions in different translation units may not be deduplicated.

In addition, `exclude_from_explicit_instantiation` or `always_inline` serves another purpose in conjunction with the hidden visibility.
If a member function in an extern template has hidden visibility (due to an attribute, `-fvisibility-inlines-hidden`, or `-fvisibility=hidden`), a function call to it can compile to an hidden undefined symbol, which cannot be resolved to a definition in the libc++ DSO.

Some intended non-exported symbols were previously not marked as hidden from ABI. The compiled programs may reference the definition in the libc++ DSO.
Hidding these symbols for the stable ABI would cause an ABI break. These symbols can be annotated as `_LIBCPP_HIDE_FROM_ABI_AFTER_V1`:

* when building libc++, hide the symbol from ABI after ABI v1.
* when not building libc++, hide the symbol from ABI.

```cpp
#include <iostream>
int main() { return std::cout.tellp(); }
```

For example, the demangled symbol `std::__1::basic_ostream<char, std::__1::char_traits<char> >::tellp()` historically was undefined and now is defined with hidden visibility.
`libc++.so.1` exports the symbol to satisfy old executables and DSOs. `libc++.so.2` does not export the symbol.

## Relocatable object files compiled with different versions of libc++ headers

libc++ refers to the per-translation-unit ABI insulation as the ability to safely link two relocatable object files built with different versions of libc++ into the same executable or DSO.
Additionally, this requires that non-exported symbols from building relocatable object files with different libc++ versions do not conflict.

To achieve per-translation-unit ABI insulation, an intended non-exported symbol can be annotated with `__attribute__((__abi_tag__(...)))`. The ABI tag string is derived from `_LIBCPP_VERSION`.
In the default configuration, `_LIBCPP_HIDE_FROM_ABI` consists of `__attribute__((__visibility__("hidden"))) __attribute__((__exclude_from_explicit_instantiation__)) __attribute__((__abi_tag__(...)))`.

Let's consider an example.
```cpp
% cat a.cc
#include <vector>
int foo(const std::vector<int> &a) { return a[0]; }
% clang-17 -c -stdlib=libc++ a.cc
% readelf -WsC b.o | grep 'vector<'
     4: 0000000000000000    33 FUNC    GLOBAL DEFAULT    2 foo(std::__1::vector<int, std::__1::allocator<int> > const&)
     5: 0000000000000000    32 FUNC    WEAK   HIDDEN     5 std::__1::vector<int, std::__1::allocator<int> >::operator[][abi:v170000](unsigned long) const
% clang-17 -c -stdlib=libc++ -D_LIBCPP_NO_ABI_TAG a.cc
% readelf -WsC b.o | grep 'vector<'
     4: 0000000000000000    33 FUNC    GLOBAL DEFAULT    2 foo(std::__1::vector<int, std::__1::allocator<int> > const&)
     5: 0000000000000000    32 FUNC    WEAK   HIDDEN     5 std::__1::vector<int, std::__1::allocator<int> >::operator[](unsigned long) const
```

When compiling `a.c` with libc++ that defines `_LIBCPP_VERSION` to 170000, the `std::vector::operator[]` generates a symbol whose name `B7v170000`.
However, including this string for every `_LIBCPP_HIDE_FROM_ABI` symbol can significantly increase the size of relocatable object files, therefore `_LIBCPP_NO_ABI_TAG` is provided as an escape hatch when users do not require this strong ABI compatibility.

In libc++ 18, the ABI tag also encodes hardened/safe/debug/unchecked mode and whether exceptions are enabled.
```
% clang -c -stdlib=libc++ a.cc
% readelf -WsC a.o | grep vector\<
    12: 0000000000000000    33 FUNC    GLOBAL DEFAULT    2 foo(std::__2::vector<int, std::__2::allocator<int> > const&)
    13: 0000000000000000   135 FUNC    WEAK   HIDDEN     5 std::__2::vector<int, std::__2::allocator<int> >::operator[][abi:se180000](unsigned long) const
    14: 0000000000000000    28 FUNC    WEAK   HIDDEN     9 std::__2::vector<int, std::__2::allocator<int> >::size[abi:se180000]() const
```

Previously, libc++ used `__attribute__((internal_linkage))` to provide per-translation-unit ABI insulation.
As aforementioned, the main downside is the presence of duplidate copies of functions across translation units.

## Pitfalls

If a process has two different libc++ versions:

* there are multiple copies of some global variables and taking address of functions from the two copies can lead to different results.
* dynamic initialization (e.g. `libcxx/src/iostream_init.h:__start_std_streams`) may happen twice.

## libstdc++ ABI stability

If a program was built with an old libstdc++ version, it should work with an upgraded libstdc++ DSO (barring the program's own bugs).
libstdc++ is widely used as a system library for Linux distributions. It's fairly common to run a program with a newer version of libstdc++.
So, let's revisit the two questions.

* Can the program, built with a specific version of libstdc++, work with an upgraded libstdc++ shared object (DSO)? Yes.
* Can an executable and its DSOs be compiled with different versions of libstdc++ headers? Yes, the linked libstdc++ must be the newest one.

In libstdc++, there is no instance of the libc++-style `always_inline/internal_linkage/abi_tag`, suggesting that it doesn't provide per-TU ABI insulation in any configuration.
It appears that the documentation at <https://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html> should cover the implications for archives, but it currently does not.

In situations that a process an executable and its DSOs require two different libstdc++ copies, it is recommended to rename the conflicting symbols.

libstdc++ uses the `abi_tag` attribute as well but for limited purposes, e.g. the GCC 5 change to switch `std::basic_string` to use small size optimization.
See [GCC5 and the C++11 ABI](https://developers.redhat.com/blog/2015/02/05/gcc5-and-the-c11-abi) for detail.

```
% cat a.cc
#include <string>
void foo(std::string a) {}
% g++ -S a.cc -o - | grep 'globl'
        .globl  _Z3fooNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
% g++ -S a.cc -D_GLIBCXX_USE_CXX11_ABI=0 -o - | grep 'globl'
        .globl  _Z3fooSs
```

Statically linking libstdc++ is unpopular as it's difficult to ensure that there is no conflicting `libstdc++.so.6`.

### Symbol versioning

While libstdc++ utilizes [symbol versioning](/blog/2020-11-26-all-about-symbol-versioning), I think the primary use case is so that glibc rtld will report an error (version `GLIBCXX_XXX' not found (required by YYY)` when a DSO built with new libstdc++ is loaded by an old `libstdc++.so.6`.

Non-default symbols are limited to the implementation files and not the headers.
If I run `llvm-nm -gjDU /usr/lib/gcc/x86_64-linux-gnu/12/libstdc++.so | grep '[^@]@[^@]'`, I can only find very few symbols.

GCC defines a `symver` attribute, but it cannot be used with a template function, making it not a substitute for the `abi_tag` attribute.

## C++ ABI library

The above discussion has bypassed another important component, C++ ABI library.
Common implementations are:

* libsupc++, part of libstdc++
* libc++abi in llvm-project
* libcxxrt, primarily used by FreeBSD

The C++ standard library implementation in llvm-project, libc++, can leverage libc++abi, libcxxrt or libsupc++, but libc++abi is recommended.

These libraries have to provide certain symbols as required by the ABI. They do not use inline namespaces.
Mixing the C++ ABI libraries would lead to ODR violation and possibly mysterious behaviors.

To link two libc++ copies, we can do:

* libc++ X => `b.so`
* `b.so`, libc++ Y, libc++abi => `a.out`

With some efforts, you can mix libstdc++ and libc++. This requires compiling the non-libsupc++ part of libstdc++ to get a homebrew `libstdc++.so.6`.
To mix libstdc++ and libc++, we can do:

* libstdc++ without libsupc++ => `b.so`
* `b.so`, libc++, libc++abi => `a.out`

Normally, if you want to link a library using libstdc++ into an executable using libc++, the best way is to ensure that the library only exposes C and C++ `extern "C"` symbols.

<!--
## Interesting changes

https://lists.llvm.org/pipermail/cfe-dev/2018-August/059024.html "Adding a new attribute: no_extern_template "
https://lists.llvm.org/pipermail/cfe-dev/2018-July/058419.html "A proposal for getting rid of __always_inline__ in _LIBCPP_INLINE_VISIBILITY"
commit cb3eb30636c8d9136c8a01126963eded45617531 added `_LIBCPP_HIDE_FROM_ABI` to replace `_LIBCPP_INLINE_VISIBILITY`.
-->
