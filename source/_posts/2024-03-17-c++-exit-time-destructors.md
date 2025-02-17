layout: post
title: C++ exit-time destructors
author: MaskRay
tags: [c++]
---

In ISO C++ standards, [basic.start.term] specifies that:

> Constructed objects ([dcl.init]) with static storage duration are destroyed and functions registered with std::atexit are called as part of a call to std::exit ([support.start.term]). The call to std::exit is sequenced before the destructions and the registered functions.
> [Note 1: Returning from main invokes std::exit ([basic.start.main]). — end note]

For example, consider the following code:

```cpp
struct A { ~A(); } a;
```

The destructor for object a will be registered for execution at program termination.

<!-- more -->

## `__cxa_atexit`

The Itanium C++ ABI employs `__cxa_atexit` rather than atexit for object destructor registration for two primary reasons:

* Limited `atexit` guarantee: ISO C (up to C23) guarantees support for 32 registered functions, although most implementations support many more.
* Dynamic library unloading: `__cxa_atexit` provides a mechanism for handling destructors when dynamic libraries are unloaded via `dlclose` before program termination.

Several standard libraries, including glibc, musl, and FreeBSD libc, implement `atexit` using `__cxa_atexit`.

* In glibc, `atexit` returns `__cxa_atexit ((void (*) (void *)) func, NULL, __dso_handle)`, where `__dso_handle` is part of libc itself.
* musl uses 0 instead of `__dso_handle`.

<https://itanium-cxx-abi.github.io/cxx-abi/abi.html#dso-dtor-runtime-api> provides detailed documentation on object destruction mechanisms.
Let's illustrate this with a GCC and glibc example:

```sh
cat > a.cc <<'eof'
#include <dlfcn.h>
int main() {
  void *h = dlopen("./b.so", RTLD_NOW);
  ((void (*)())dlsym(h, "foo"))();
  dlclose(h);
}
eof
cat > b.cc <<'eof'
#include <stdio.h>
struct A { ~A(); } ga;
A::~A() { printf("~A %p\n", this); }
extern "C" void foo() {
  static A a;
  puts("foo");
}
eof
g++ -fpic -shared b.cc -o b.so
g++ a.cc -o a
```

An invocation yields:

```
foo
~A 0x7f70d66c4c79  // for the static-local variable
~A 0x7f70d66c4c78  // for the global variable
```

Key points:

* The compiler registers destructors with `__cxa_atexit` using the `__dso_handle` symbol as an argument.
* `crtbeginS.o` defines the `.fini_array` section (triggering `__do_global_dtors_aux`) and the hidden symbol `__dso_handle`.
* Since 2017, lld [defines `__dso_handle` as a hidden symbol](https://reviews.llvm.org/D33856) if crtbegin does not.
* `dlclose` invokes `.fini_array` functions. `__cxa_finalize(d)` iterates through the termination function list, calling matching destructors based on the DSO handle.
* `__cxa_atexit` implementations typically allocate memory dynamically and may fail. The failures are simply ignored.

Note: In glibc, the `DF_1_NODELETE` flag marks a shared object as unloadable. Additionally, symbol lookups with `STB_GNU_UNIQUE` automatically set this flag.

musl provides a [no-op implementation for `dlclose` and `__cxa_finalize`](https://wiki.musl-libc.org/functional-differences-from-glibc.html#Unloading_libraries).

## Thread storage duration variables

Objects with thread storage duration that have non-trivial destructors will register those destructors using `__cxa_thread_atexit` during construction.

## When exit-time destructors are undesired

Exit-time destructors for static and thread storage duration variables can be undesired due to

* Unnecessary overhead and complexity: This includes operating system kernels and memory-constrained systems.
* Potential race conditions: Destructors might execute during thread termination, while other threads still attempt to access the object. Examples: [webkit](https://bugs.webkit.org/show_bug.cgi?id=21810)

Clang provides `-Wexit-time-destructors` (disabled by default) to warn about exit-time destructors.

```
% clang++ -c -Wexit-time-destructors g.cc
g.cc:1:20: warning: declaration requires an exit-time destructor [-Wexit-time-destructors]
    1 | struct A { ~A(); } a;
      |                    ^
1 warning generated.
```

## Disabling exit-time destructors

Then, I will describe some approaches to disable exit-time destructors.

### Pointer/reference to a dynamically-allocated object

We can use a reference or pointer that refers to a dynamically-allocated object.

```cpp
struct A { int v; ~A(); };
A &g = *new A; // or A *const g = new A;
A &foo() {
  static A &a = *new A;
  return a; // or static A *a = new A; return *a
}
```

This approach prevents the destructor from running at program exit, as pointers and references have a trivial destructor.
Note that this does not create a memory leak, since the pointer/reference is part of the root set.

The primary downside is unnecessary pointer indirection when accessing the object.
Additionally, this approach uses a mutable pointer in the data segment and requires a memory allocation.

```asm
# %bb.2:                                 // initializer
        movl    $4, %edi
        callq   _Znwm@PLT
        movq    %rax, _ZZ3foovE1a(%rip)  // store pointer of the heap-allocated object to _ZZ3foovE1a
...
        movq    _ZZ3foovE1a(%rip), %rax  // load a pointer from _ZZ3foovE1a
```

## Class template with an empty destructor

A common approach, as outlined in [P1247](https://wg21.link/p1247), is to use a class template with an empty destructor to prevent exit-time destruction:

```cpp
template <class T> class no_destroy {
  alignas(T) std::byte data[sizeof(T)];
public:
  template <class... Ts> no_destroy(Ts&&... ts) { new (data) T(std::forward<Ts>(ts)...); }
  T &get() { return *reinterpret_cast<T *>(data); }
};

no_destroy<widget> my_widget;
```

libstdc++ employs a variant that uses a union member.

```cpp
struct A { ~A(); };

namespace {
  struct constant_init {
    union { A obj; };
    constexpr constant_init() : obj() { }
    ~constant_init() { /* do nothing, union member is not destroyed */ }
  };
  constinit constant_init global;
}

A* get() { return &global.obj; }
```

C++20 will support constexpr destructor:

```cpp
template <class T> union no_destroy {
  template <typename... Ts>
  explicit constexpr no_destroy(Ts&&... args) : obj(std::forward(args)...) {}
  constexpr ~no_destroy() {}
  T obj;
};
```

Libraries like [`absl::NoDestructor`](https://github.com/abseil/abseil-cpp/blob/master/absl/base/no_destructor.h) and [`folly::Indestructible`](https://github.com/facebook/folly/blob/main/folly/Indestructible.h) offer similar functionality.
The absl version optimizes for trivially destructible types.

### Compiler optimization for no-op destructors

Ideally, compilers should optimize out exit-time destructors for empty user-provided destructors:

```cpp
struct C { C(); ~C() {} };
void foo() { static C c; }
```

LLVM has addressed this [since 2011](https://github.com/llvm/llvm-project/commit/ee6bc70d2f1c2434ca9ca8092216bdeab322c7e5).
Its GlobalOpt pass eliminates `__cxa_atexit` calls related to empty destructors, along with other global variable optimizations.

In contrast, GCC has an [open feature request](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=19661) for this optimization since 2005.

### `no_destroy` attribute

Clang supports `[[clang::no_destroy]]` (alternative form: `__attribute__((no_destroy))`) to disable exit-time destructors for variables of static or thread storage duration.
Its `-fno-c++-static-destructors` option allows disabling exit-time destructors globally.

* July 2018 discussion: <https://discourse.llvm.org/t/rfc-suppress-c-static-destructor-registration/49128>
* Patch: <https://reviews.llvm.org/D50994> with follow-up <https://reviews.llvm.org/D54344>
* Documentation: <https://clang.llvm.org/docs/AttributeReference.html#no-destroy>

Standardization efforts for this attribute are underway [P1247R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1247r0.html).

I recently encountered a scenario where the `no_destroy` attribute would have been beneficial.
I've filed a GCC feature request ([PR114357](https://gcc.gnu.org/PR114357)) after I learned that GCC doesn't have the attribute.

## Case study

LLVM provides `ManagedStatic` to construct an object on-demand (good for reducing startup time) and make destruction explicitly through `llvm_shutdown`.
`ManagedStatic` is intended to be used at namespace scope. A prime example is LLVM's statistics mechanisms (`-stats` and `-time-passes`).

Programs using LLVM can strategically avoid calling `llvm_shutdown` for fast teardown by skipping some destructors.
The lld linker employs this approach unless the `LLD_IN_TEST` environment variable is set to a non-zero integer.

DSO plugin users requiring library unloading may [find `ManagedStatic` unsuitable](https://discourse.llvm.org/t/making-llvm-play-nice-r-when-used-as-a-shared-library-in-a-plugin-setting/63306).
This is because:

* A DSO may not be able to determine if other active LLVM users exist within the process, making it unsafe to call `llvm_shutdown`.
* If `llvm_shutdown` is deferred until around program exit, executing destructors becomes unsafe once the DSO's code has been removed.

The mold linker improves perceived linking speed by spawning a separate process for the linking task.
This allows the parent process (the one launched from the shell or other programs) to exit early.
This approach eliminates overhead associated with static destructors and other operations.
