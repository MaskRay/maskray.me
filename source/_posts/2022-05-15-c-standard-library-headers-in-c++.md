---
layout: post
title: C standard library headers in C++
author: MaskRay
tags: [c++,gcc,glibc]
---

Updated in 2024-01.

In ISO C++ standards, [support.c.headers.general] says:

> Source files that are not intended to also be valid ISO C should not use any of the C headers.

Then, [depr.c.headers] describes how a C header `name.h` is transformed to the corresponding C++ `cname` header. There is a helpful example:

> [ Example: The header <cstdlib> assuredly provides its declarations and definitions within the namespace std. It may also provide these names within the global namespace. The header <stdlib.h> assuredly provides the same declarations and definitions within the global namespace, much as in the C Standard. It may also provide these names within the namespace std. — end example ]

"may also" in the wording allows implementations to provide mix-and-match, e.g. `#include <stdlib.h>` may provide `std::exit` and `#include <cstdlib>` may provide `::exit`.

libstdc++ chooses to enable global namespace declarations with C++ `cname` header.
For example, `#include <cstdlib>` also includes the corresponding C header `stdlib.h` and we get declarations in both the global namespace and the namespace std.
```text
. /usr/include/c++/12/cstdlib
.. /usr/include/stdlib.h
```

The preprocessed output looks like:
```cpp
extern void exit (int __status) noexcept (true) __attribute__ ((__noreturn__));

extern "C++"
{
namespace std __attribute__ ((__visibility__ ("default")))
{
  using ::exit;
}
}
```

The compiler knows that the declarations in the namespace `std` are identical to the ones in the global namespace.
The compiler recognizes some library functions and can optimize them. By `using` the compiler can optimize some C library functions in the namespace `std` (e.g. many `std::mem*` and `std::str*` functions).

For some C standard library headers, libstdc++ provides wrappers (`libstdc++-v3/include/c_compatibility/`) which take precedence over the glibc headers.
The configuration of libstdc++ uses [`--enable-cheaders=c_global`](https://gcc.gnu.org/onlinedocs/libstdc++/manual/configure.html) by default.
`if GLIBCXX_C_HEADERS_C_GLOBAL` in `libstdc++-v3/include/Makefile.am` describes that the 6 wrappers (`complex.h, fenv.h, tgmath.h, math.h, stdatomic.h, stdlib.h`) shadow the C library headers of the same name.
For example, `#include <stdlib.h>` includes the wrapper `stdlib.h` which includes `cstdlib`, therefore bringing `exit` into the namespace std.
```text
. /usr/include/c++/12/stdlib.h
.. /usr/include/c++/12/cstdlib
... /usr/include/stdlib.h
```

<!-- more -->

In all `--enable-cheaders` modes, `#include <cstdlib>` also includes the corresponding C header, therefore we get declarations in both the global namespace and the namespace std.
```text
. /usr/include/c++/12/cstdlib
.. /usr/include/stdlib.h
```

The mix-and-match mechanism looks gross, but it has been needed for compatibility in the ancient days. Nowadays, I can imagine widespread breakage if we drop the mix-and-match.

## string.h and cstring

In ISO C standards before N3020, [String handling] lists:

```c
void *memchr(const void *s, int c, size_t n);
char *strchr(const char *s, int c);
char *strpbrk(const char *s1, const char *s2);
char *strstr(const char *s1, const char *s2);
```

Note that the return types do not have the const qualifier, e.g. `memchr` would better return a `const void *`.
Unfortunately that is not the case likely due to two reasons:

First, there hadn't been the `const` qualifier in The C Programming Language (1st edition, 1978).
The second edition of the C Programming Language (1988) introduced the const qualifier, added the const qualifier to some arguments, but did not change the return type, probably because that would break code.

Second, the const input non-const output signature provides the best usability in the absence of overloading. `strchr` works with both examples below:
```c
char *str = ...;
char *pos = strchr(str, 'a');

const char *str = ...;
const char *pos = strchr(str, 'a');
```

Anyhow, C++ decided to "correct" the signatures. I do not know when that decision was made.
In the current ISO C++ standard, [library.c] says

> The descriptions of many library functions rely on the C standard library for the semantics of those functions. In some cases, the signatures specified in this document may be different from the signatures in the C standard library, and additional overloads may be declared in this document, but the behavior and the preconditions (including any preconditions implied by the use of an ISO C restrict qualifier) are the same unless otherwise stated.

[cstring.syn] lists these functions with different signatures:

```cpp
namespace std {
  const void* memchr(const void* s, int c, size_t n);           // see [library.c]
  void* memchr(void* s, int c, size_t n);                       // see [library.c]
  const char* strchr(const char* s, int c);                     // see [library.c]
  char* strchr(char* s, int c);                                 // see [library.c]
  const char* strpbrk(const char* s1, const char* s2);          // see [library.c]
  char* strpbrk(char* s1, const char* s2);                      // see [library.c]
  const char* strstr(const char* s1, const char* s2);           // see [library.c]
  char* strstr(char* s1, const char* s2);                       // see [library.c]
}
```

There is an apparent attempt to correct the mistakes as evidenced by the note:

> [Note 1: The functions strchr, strpbrk, strrchr, strstr, and memchr, have different signatures in this document, but they have the same behavior as in the C standard library. — end note]

_Qualifier-preserving standard library functions_ (N3020) has changed these functions to be generic functions to preserve the qualifier.

In glibc `string.h`, some extensions have similar overloads: `rawmemchr, strchrnul, strcasestr, basename`.

## wchar.h and cwchar

Similar differences apply to `wchar.h` vs `cwchar`. C++ [cwchar.syn] lists the following functions with different signatures:

```cpp
namespace std {
  const wchar_t* wcschr(const wchar_t* s, wchar_t c);           // see [library.c]
  wchar_t* wcschr(wchar_t* s, wchar_t c);                       // see [library.c]
  const wchar_t* wcspbrk(const wchar_t* s1, const wchar_t* s2); // see [library.c]
  wchar_t* wcspbrk(wchar_t* s1, const wchar_t* s2);             // see [library.c]
  const wchar_t* wcsrchr(const wchar_t* s, wchar_t c);          // see [library.c]
  wchar_t* wcsrchr(wchar_t* s, wchar_t c);                      // see [library.c]
  const wchar_t* wcsstr(const wchar_t* s1, const wchar_t* s2);  // see [library.c]
  wchar_t* wcsstr(wchar_t* s1, const wchar_t* s2);              // see [library.c]
  const wchar_t* wmemchr(const wchar_t* s, wchar_t c, size_t n); // see [library.c]
  wchar_t* wmemchr(wchar_t* s, wchar_t c, size_t n);            // see [library.c]
}
```

> [Note 1: The functions wcschr, wcspbrk, wcsrchr, wcsstr, and wmemchr have different signatures in this document, but they have the same behavior as in the C standard library. — end note]

## glibc `__CORRECT_ISO_CPP_STRING_H_PROTO`

To fix `string.h` signatures for C++, in 2009, `__CORRECT_ISO_CPP_STRING_H_PROTO` [was added](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=d8387c7b7b1c9ae92f924c33ba05790c98464d19) to provide the correct overloads.
Nowadays `string.h` looks like:

```c
/* Tell the caller that we provide correct C++ prototypes.  */
#if defined __cplusplus && (__GNUC_PREREQ (4, 4) \
                            || __glibc_clang_prereq (3, 5))
# define __CORRECT_ISO_CPP_STRING_H_PROTO
#endif

#ifdef __CORRECT_ISO_CPP_STRING_H_PROTO
extern "C++"
{
extern void *memchr (void *__s, int __c, size_t __n)
      __THROW __asm ("memchr") __attribute_pure__ __nonnull ((1));
extern const void *memchr (const void *__s, int __c, size_t __n)
      __THROW __asm ("memchr") __attribute_pure__ __nonnull ((1));

# ifdef __OPTIMIZE__
__extern_always_inline void *
memchr (void *__s, int __c, size_t __n) __THROW
{
  return __builtin_memchr (__s, __c, __n);
}

__extern_always_inline const void *
memchr (const void *__s, int __c, size_t __n) __THROW
{
  return __builtin_memchr (__s, __c, __n);
}
# endif
}
#else
extern void *memchr (const void *__s, int __c, size_t __n)
      __THROW __attribute_pure__ __nonnull ((1));
#endif
```

In C++ mode, there are two `extern "C++"` overloads for `memchr`. To prevent C++ name mangling and redirect them to the symbol `memchr`, asm labels (`__asm ("memchr")`) are used.
(`__glibc_clang_prereq (3, 5)` was added in 2019 as the resolution to <https://sourceware.org/bugzilla/show_bug.cgi?id=25232>.)

The code fragment is used together with the following fragment in libstdc++ `cstring`:
```cpp
extern "C++"
{
namespace std __attribute__ ((__visibility__ ("default")))
{
  using ::memchr;
}
}
```

With the glibc and libstdc++ cooperation, the following code (adapted from `libstdc++-v3/testsuite/21_strings/c_strings/char/3_neg.cc`) will get a compile error.
```cpp
char *v1;
const char *cv2;

v1 = std::memchr (cv2, '/', 3);      // { dg-error "invalid conversion" }
```

If we modify glibc `string.h` by undefining `__CORRECT_ISO_CPP_STRING_H_PROTO`, the compile error will go away.

IMO it would look better if libstdc++ removed `using ::memchr` and declared the overloads in the namespace std.
libc++ took this approach in 2016. Actually it can use asm labels to avoid `__libcpp_memchr`:
```cpp
#if defined(__CORRECT_ISO_CPP_STRING_H_PROTO) || defined(_LIBCPP_MSVCRT) || \
    defined(__sun__) || defined(_STRING_H_CPLUSPLUS_98_CONFORMANCE_)
#define _LIBCPP_STRING_H_HAS_CONST_OVERLOADS
#endif

#if defined(__cplusplus) && !defined(_LIBCPP_STRING_H_HAS_CONST_OVERLOADS) && defined(_LIBCPP_PREFERRED_OVERLOAD)
extern "C++" {
inline _LIBCPP_INLINE_VISIBILITY
void* __libcpp_memchr(const void* __s, int __c, size_t __n) {return (void*)memchr(__s, __c, __n);}
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_PREFERRED_OVERLOAD
const void* memchr(const void* __s, int __c, size_t __n) {return __libcpp_memchr(__s, __c, __n);}
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_PREFERRED_OVERLOAD
      void* memchr(      void* __s, int __c, size_t __n) {return __libcpp_memchr(__s, __c, __n);}
#endif
```

## glibc `__CORRECT_ISO_CPP_WCHAR_H_PROTO`

There is a similar macro `__CORRECT_ISO_CPP_WCHAR_H_PROTO` for `wchar.h`.

A libc++ change <https://reviews.llvm.org/D148542> accidentally made `__CORRECT_ISO_CPP_WCHAR_H_PROTO` undefined when `iosfwd` is included first.
glibc before 2.26 did not provide [`bits/types/mbstate_t.h`](https://sourceware.org/git/?p=glibc.git;a=commit;h=199fc19d3aaaf57944ef036e15904febe877fc93).
If we include `iosfwd` before `cwchar`, we will include glibc `wchar.h` without defining `__CORRECT_ISO_CPP_WCHAR_H_PROTO` and will get the "incorrect" declarations.

```cpp
#ifdef X
#include <iosfwd>
#include <cwchar>
#else
#include <cwchar>
#include <iosfwd>
#endif
int main() {
  wchar_t *v1;
  const wchar_t *cv2 = L"/";
  v1 = wcsstr(cv2, L"/");  // should fail
}
```

```
% clang --sysroot=glibc-2.25 -stdlib=libc++ -c test.cc   # expected error
test.cc:11:8: error: assigning to 'wchar_t *' from 'const wchar_t *' discards qualifiers
  v1 = wcsstr(cv2, L"/");
       ^~~~~~~~~~~~~~~~~~
1 error generated.
% clang --sysroot=glibc-2.25 -stdlib=libc++ -c test.cc -DX  # unexpected success
```

If we change the libc++ `module.modulemap` file (in my build, `/tmp/Rel/include/c++/v1/module.modulemap`) to be a textual header, or just remove the entry from the modulemap file,
```
module __mbstate_t         { private textual header "__mbstate_t.h"       export * }
```

the following command will fail. (Yes, we use `-x c++` even when compiling a modulemap file, otherwise Clang will consider the modulemap file an object file for linking.)
```
% clang --sysroot=glibc-2.25 -stdlib=libc++ -c -Xclang -emit-module -fmodules -fmodule-name=std -Xclang=-fmodules-local-submodule-visibility -xc++ /tmp/Rel/include/c++/v1/module.modulemap -o std.pcm
...
In file included from /tmp/Rel/bin/../include/c++/v1/__mbstate_t.h:35:
glibc-2.25/include/wchar.h:227:17: error: functions that differ only in their return type cannot be overloaded
extern wchar_t *wcschr (const wchar_t *__wcs, wchar_t __wc)
       ~~~~~~~~~^
glibc-2.25/include/wchar.h:224:29: note: previous declaration is here
extern "C++" const wchar_t *wcschr (const wchar_t *__wcs, wchar_t __wc)
                   ~~~~~~~~~^
```

We can write a modulemap file that compiles just two headers.
```sh
mkdir stl
echo '#include_next <cwchar>' > stl/cwchar
echo '#include_next <iosfwd>' > stl/iosfwd
cat > stl.modulemap <<e
module "stl" {
  export *
  module "stl/cwchar" { export * header "stl/cwchar" }
  module "stl/iosfwd" { export * header "stl/iosfwd" }
}
e
```

```
% clang --sysroot=glibc-2.25 -stdlib=libc++ -c -Xclang -emit-module -fmodules -fmodule-name=stl -Xclang=-fmodules-local-submodule-visibility -xc++ stl.modulemap -o stl.pcm
...
glibc-2.25/include/wchar.h:227:17: error: functions that differ only in their return type cannot be overloaded
extern wchar_t *wcschr (const wchar_t *__wcs, wchar_t __wc)
       ~~~~~~~~~^
# This command will fail even without the change to /tmp/Rel/include/c++/v1/module.modulemap
% clang --sysroot=glibc-2.25 -stdlib=libc++ -c -Xclang -emit-module -fmodules -fmodule-name=stl -fno-implicit-module-maps -Xclang=-fmodules-local-submodule-visibility -xc++ stl.modulemap -o stl.pcm
...
glibc-2.25/include/wchar.h:227:17: error: functions that differ only in their return type cannot be overloaded
extern wchar_t *wcschr (const wchar_t *__wcs, wchar_t __wc)
       ~~~~~~~~~^
```

## stdatomic.h

C11 defines `stdatomic.h` which defines several macros and declares several types and functions for performing atomic operations on data shared between threads.
The header is shipped with the compiler (GCC, Clang).

<https://wg21.link/P0943> (C++23) defines the semantics including `stdatomic.h` in C++. Implementations:

* libstdc++: <https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d7f2a09e98520c4e4927f460ad96803b05a205b1>
* libc++: <https://reviews.llvm.org/D97044>

```cpp
#include <stdatomic.h>
#include <atomic>
```
This above code works with C++23. In older language modes, all of `g++`, `clang++ -stdlib=libstdc++`, and `clang++ -stdlib=libc++` report errors.

```cpp
#include <atomic>
#include <stdatomic.h>
```
This above code works with C++23. In older language modes, both `g++` and `clang++ -stdlib=libstdc++` report errors; `clang++ -stdlib=libc++` accepts the code.

## `using_if_exists`

libc++ [utilizes a Clang extension `using_if_exists`](https://reviews.llvm.org/D90257) to import a declaration only when the global namespace has such a declaration, avoiding an error in case the declaration does not exist.

```cpp
using ::sqrt _LIBCPP_USING_IF_EXISTS;
```
