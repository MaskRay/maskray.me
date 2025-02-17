layout: post
title: "AddressSanitizer: global variable instrumentation"
author: MaskRay
tags: [llvm,clang,sanitizer]
---

AddressSanitizer (ASan) is a compiler technology that detects addressability-related memory errors with some additional checks.
It consists of two components: compiler instrumentation and a runtime library. To put it simply,

* The compiler instruments global variables, stack frames, and heap allocations to monitor shadow memory.
* The compiler also instruments memory access instructions to verify shadow memory.
* In case of an error, the inserted code invokes a callback (implemented in the runtime library) to report the error along with a stack trace. Typically, the program will terminate after displaying the error message.

This article describes global variable instrumentation.

## Global variable instrumentation

AddressSanitizer instruments certain defined global variables of LLVM external or internal linkage.
To be instrumented, the variable must satisfy a bunch of conditions.

* It is not thread-local.
* It has a smaller alignment.
* It is not synthesized by LLVM.
* It does not have the `no_sanitize_address` attribute in LLVM IR. Variables receive this attribute when annotated as `__attribute__((no_sanitize("address")))` or `__attribute__((disable_sanitizer_instrumentation))` in C/C++.

```c
int g0;
const long g1 = 42;
```

Each instrumented global variable is padded with a right redzone to detect out-of-bounds accesses.
```llvm
@g0 = dso_local global { i32, [28 x i8] } zeroinitializer, comdat, align 32
@g1 = dso_local constant { i64, [24 x i8] } zeroinitializer, comdat, align 32
```

On ELF platforms, by default (since Clang 17.0) each instrumented global variable receives an associated `__asan_global_$name` variable, which is located within the `asan_globals` section.
Additionally, there are several related variables, including some unnamed ones (`@0` and `@1`), as well as `__odr_asan_gen_g0` and `__odr_asan_gen_g1`, along with metadata nodes (`!0` and `!1`), which we will discuss in more detail later."

```llvm
@___asan_gen_.1 = private unnamed_addr constant [3 x i8] c"g0\00", align 1
@___asan_gen_.2 = private unnamed_addr constant [3 x i8] c"g1\00", align 1
@__asan_global_g0 = private global { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @0 to i64), i64 4, i64 32, i64 ptrtoint (ptr @___asan_gen_.1 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 ptrtoint (ptr @__odr_asan_gen_g0 to i64) }, section "asan_globals", comdat($g0), !associated !0
@__asan_global_g1 = private global { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @1 to i64), i64 4, i64 32, i64 ptrtoint (ptr @___asan_gen_.2 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 ptrtoint (ptr @__odr_asan_gen_g1 to i64) }, section "asan_globals", comdat($g1), !associated !1
@llvm.compiler.used = appending global [4 x ptr] [ptr @g0, ptr @g1, ptr @__asan_global_g0, ptr @__asan_global_g1], section "llvm.metadata"

!0 = !{ptr @g0}
!1 = !{ptr @g1}
```

The module constructor `asan.module_ctor` processes garbage-collectable `asan_globals` input sections.
This constructor invokes a runtime callback to register the instrumented global variables, which involves poisoning the redzone and conducting ODR violation checks.
I will discuss ODR violation checking later.
```llvm
define internal void @asan.module_ctor() #0 comdat {
  call void @__asan_init()
  call void @__asan_version_mismatch_check_v8()
  call void @__asan_register_elf_globals(i64 ptrtoint (ptr @___asan_globals_registered to i64), i64 ptrtoint (ptr @__start_asan_globals to i64), i64 ptrtoint (ptr @__stop_asan_globals to i64))
  ret void
}
```

The runtime poisons the redzone of each instrumented global variable.
```cpp
void __asan_register_elf_globals(uptr *flag, void *start, void *stop) {
  if (*flag) return;
  if (!start) return;
  CHECK_EQ(0, ((uptr)stop - (uptr)start) % sizeof(__asan_global));
  __asan_global *globals_start = (__asan_global*)start;
  __asan_global *globals_stop = (__asan_global*)stop;
  __asan_register_globals(globals_start, globals_stop - globals_start);
  *flag = 1;
}

void __asan_register_globals(__asan_global *globals, uptr n) {
  if (!flags()->report_globals) return;
  ...
  for (uptr i = 0; i < n; i++)
    RegisterGlobal(&globals[i]);

  // Poison the metadata. It should not be accessible to user code.
  PoisonShadow(reinterpret_cast<uptr>(globals), n * sizeof(__asan_global),
               kAsanGlobalRedzoneMagic);
}

static void RegisterGlobal(const Global *g) {
  ...
  if (CanPoisonMemory())
    PoisonRedZones(*g);
}
```

Every full granule in the shadow of the redzone is filled with 0xf9 (`kAsanGlobalRedzoneMagic`) while a partial granule is filled in a manner similar to partially-addressable stack memory.
```cpp
ALWAYS_INLINE void PoisonRedZones(const Global &g) {
  uptr aligned_size = RoundUpTo(g.size, ASAN_SHADOW_GRANULARITY);
  FastPoisonShadow(g.beg + aligned_size, g.size_with_redzone - aligned_size,
                   kAsanGlobalRedzoneMagic);
  if (g.size != aligned_size) {
    FastPoisonShadowPartialRightRedzone(
        g.beg + RoundDownTo(g.size, ASAN_SHADOW_GRANULARITY),
        g.size % ASAN_SHADOW_GRANULARITY, ASAN_SHADOW_GRANULARITY,
        kAsanGlobalRedzoneMagic);
  }
}
```

## global-buffer-overflow example

If an access occurs within a redzone byte poisoned by 0xf9 or within a partial redzone preceding 0xf9, the runtime will report a `global-buffer-overflow` error.
Here is an example:

```c
cat > a.c <<e
#include <string.h>
int main(int argc, char **argv) {
  static char a[10];
  memset(a, 0, 10);
  return a[argc * 5];
}
e
clang -fsanitize=address a.c -o a
```

```
% ./a 1  # a[argc * 5] == a[10] is out-of-bounds
=================================================================
==240472==ERROR: AddressSanitizer: global-buffer-overflow on address 0x5592092356aa at pc 0x5592088dc38f bp 0x7ffd457ab520 sp 0x7ffd457ab518
READ of size 1 at 0x5592092356aa thread T0
    #0 0x5592088dc38e  (/tmp/c/a+0x14238e)
    #1 0x7fd59d38f6c9  (/lib/x86_64-linux-gnu/libc.so.6+0x276c9) (BuildId: 2ac5fa07c22f99cfd5dc47c70cd5f0e78b974269)
    #2 0x7fd59d38f784  (/lib/x86_64-linux-gnu/libc.so.6+0x27784) (BuildId: 2ac5fa07c22f99cfd5dc47c70cd5f0e78b974269)
    #3 0x559208800f80  (/tmp/c/a+0x66f80)

0x5592092356aa is located 0 bytes after global variable 'main.a' defined in 'a.c' (0x5592092356a0) of size 10
SUMMARY: AddressSanitizer: global-buffer-overflow (/tmp/c/a+0x14238e)
Shadow bytes around the buggy address:
  0x559209235400: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235480: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235500: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235580: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235600: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x559209235680: 00 00 00 00 00[02]f9 f9 00 00 00 00 00 00 00 00
  0x559209235700: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235780: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235800: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235880: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x559209235900: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
...
```

## ODR violation checker

The global variable poisoning mechanism offers a straightforward means to detect differences in variable definitions between two components, such as between the main executable and a shared object, or between two shared objects.
This can be considered a category of ODR violations.

```sh
echo 'int var; int main() { return var; }' > a.cc
echo 'long var;' > b.cc
clang++ -fpic -fsanitize=address -shared b.cc -o b.so
clang++ -fsanitize=address a.cc ./b.so -o a
```
```
% ./a
=================================================================
==1299789==ERROR: AddressSanitizer: odr-violation (0x56107ea3f500):
  [1] size=4 'var' a.cc in /tmp/c/a
  [2] size=8 'var' b.cc in ./b.so
These globals were registered at these points:
  [1]:
    #0 0x56107df99996  (/tmp/c/a+0x7b996)
    #1 0x56107df9aab9  (/tmp/c/a+0x7cab9)
    #2 0x7f72e5a457f5  (/lib/x86_64-linux-gnu/libc.so.6+0x277f5) (BuildId: 2ac5fa07c22f99cfd5dc47c70cd5f0e78b974269)

  [2]:
    #0 0x56107df99996  (/tmp/c/a+0x7b996)
    #1 0x56107df9aab9  (/tmp/c/a+0x7cab9)
    #2 0x7f72e604dd2d  (/lib64/ld-linux-x86-64.so.2+0x4d2d) (BuildId: accffc5784c4a469d09348e3f7ec53a74096fbd3)

==1299789==HINT: if you don't care about these errors you may set ASAN_OPTIONS=detect_odr_violation=0
SUMMARY: AddressSanitizer: odr-violation: global 'var' at a.cc in /tmp/c/a
==1299789==ABORTING
```

The default mode, `detect_odr_violation=2`, also prohibits symbol interposition on variables.
If you change `long` to `int` in `b.cc`, you will still encounter an `odr-violation` error.
In contrast, with `detect_odr_violation=1`, errors are suppressed if the registered variables are of the same size.
```
% ASAN_OPTIONS=detect_odr_violation=1 ./a
% ASAN_OPTIONS=detect_odr_violation=2 ./a
=================================================================
==2574052==ERROR: AddressSanitizer: odr-violation (0x562d39db1200):
...
```

For a variable named `$var`, a one-byte variable, `__odr_asan_gen_$var`, is created with the original linkage (essentially must be `external`) and visibility.

If `$var` is defined in two instrumented modules, their `__odr_asan_gen_$var` symbols reference to the same copy due to symbol interposition.
When registering `$var`, the runtime checks whether `__odr_asan_gen_$var` is already 1, and if yes, the program has an ODR violation; otherwise `__odr_asan_gen_$var` is set to 1.

```llvm
@__odr_asan_gen_g0 = global i8 0, align 1
@__odr_asan_gen_g1 = global i8 0, align 1

@0 = private alias { i32, [28 x i8] }, ptr @g0
@1 = private alias { i32, [28 x i8] }, ptr @g1
```

The private aliases @0 and @1 were due to <http://reviews.llvm.org/D15642>.

If `a.supp` contains the following text, running the program with the environment variable `ASAN_OPTIONS=suppressions=a.supp` suppresses errors due to the variable name var.
```
odr_violation:^var$
```

An ODR violation is reported for two different linked units, say, `exe` and `b.so`.
With static linking, the issue can be suppressed due to archive member extraction semantics if the `b.a` member is not extracted.

### ODR indicator

The previous example uses `-fsanitize-address-use-odr-indicator`.

Prior to Clang 16,  `-fno-sanitize-address-use-odr-indicator` was the default for non-Windows platforms.
The runtime checks checks whether a variable has been registered by verifying whether its redzone has been poisoned, and reports an ODR violation when the redzone has been poisoned.
```llvm
@___asan_gen_.1 = private unnamed_addr constant [3 x i8] c"g0\00", align 1
@___asan_gen_.2 = private unnamed_addr constant [3 x i8] c"g1\00", align 1
@__asan_global_g0 = private global { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @g0 to i64), i64 4, i64 32, i64 ptrtoint (ptr @___asan_gen_.1 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 0 }, section "asan_globals", !associated !0
@__asan_global_g1 = private global { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @g1 to i64), i64 8, i64 32, i64 ptrtoint (ptr @___asan_gen_.2 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 0 }, section "asan_globals", !associated !1
@llvm.compiler.used = appending global [4 x ptr] [ptr @g0, ptr @g1, ptr @__asan_global_g0, ptr @__asan_global_g1], section "llvm.metadata"
```

This mode eliminates the need for an additional variable like `__odr_asan_gen_$var`, but it can lead to interaction issues when mixing instrumented and uninstrumented components.
In the case of a shared object, if the reference to `$var` in `__asan_global_$var` is interposed with an uninstrumented variable due to symbol interposition, it may result in a spurious error stating, "The following global variable is not properly aligned."

For Clang 16, I introduced the use of `-fsanitize-address-use-odr-indicator` by default for non-Windows targets (see https://reviews.llvm.org/D137227).

(Additionally, <https://reviews.llvm.org/D127911> changed the ODR indicator symbol name to `__odr_asan_gen_$demangled`.)

### Copy relocations

Private aliases have an interest interaction with copy relocations.
This issue is reported at <https://gcc.gnu.org/PR68016>.

The default `-fsanitize-address-use-odr-indicator` in Clang 16 and later cannot detect the `global-buffer-overflow` error below:

```sh
echo 'int f[5] = {1};' > foo.cc
echo 'extern int f[5]; int main() { return f[5]; }' > a.cc
clang++ -fpic -fsanitize=address -mllvm -asan-use-private-alias=1 -shared foo.cc -o foo1.so
clang++ -fno-pic -fsanitize=address -mllvm -asan-use-private-alias=1 -no-pie a.cc ./foo1.so -o a1
./a1 # no error

clang++ -fpic -fsanitize=address -mllvm -asan-use-private-alias=0 -shared foo.cc -o foo0.so
clang++ -fno-pic -fsanitize=address -mllvm -asan-use-private-alias=0 -no-pie a.cc ./foo0.so -o a0
./a0 # error
```

The definition of `f` in `foo.cc` is instrumented, resulting in the creation of `__asan_global_f`.
However, the executable actually accesses the copy created by the linker due to copy relocation.

When `-asan-use-private-alias=1` is in effect (the default since Clang 16), the `__asan_global_f` variable references the unused copy inside the shared object.
The executable accesses the copy-relocated variable, whose redzone is not poisoned, resulting in no error.

Conversely, when `-asan-use-private-alias=0` is in effect, the `__asan_global_f` variable references the copy-relocated variable and poisons the redzone within the executable. Consequently, accessing `f[5]` leads to the expected error.

## Garbage collection

Since Clang 17, `asan.module_ctor` is, by default, placed in a COMDAT group.
When multiple instrumented relocatable object files are linked together, only one `asan.module_ctor` is retained.

`__asan_global_g0` is positioned in a section that links to the section defining `g0` using the `SHF_LINK_ORDER` flag. During linking, if the linker discards the section defining `g0`, the `asan_globals` section containing `__asan_global_g0` will also be discarded.
For more detail on `SHF_LINK_ORDER`, you can refer to [Metadata sections, COMDAT and SHF_LINK_ORDER](/blog/2021-01-31-metadata-sections-comdat-and-shf-link-order).

Before Clang 17, the default behavior was to use `-fno-sanitize-address-globals-dead-stripping`.
In this mode, the instrumentation places pointers to instrumented global variables in a metadata array and calls `__asan_register_globals`. `__asan_register_globals` then iterates over the array and registers each global variable.

```llvm
@g0 = dso_local global { i32, [28 x i8] } zeroinitializer, align 32
@g1 = dso_local global { i64, [24 x i8] } zeroinitializer, align 32

@___asan_gen_.1 = private unnamed_addr constant [3 x i8] c"g0\00", align 1
@___asan_gen_.2 = private unnamed_addr constant [3 x i8] c"g1\00", align 1

@llvm.compiler.used = appending global [2 x ptr] [ptr @g0, ptr @g1], section "llvm.metadata"
@0 = internal global [2 x { i64, i64, i64, i64, i64, i64, i64, i64 }] [{ i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @1 to i64), i64 4, i64 32, i64 ptrtoint (ptr @___asan_gen_.1 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 ptrtoint (ptr @__odr_asan_gen_g0 to i64) }, { i64, i64, i64, i64, i64, i64, i64, i64 } { i64 ptrtoint (ptr @2 to i64), i64 4, i64 32, i64 ptrtoint (ptr @___asan_gen_.2 to i64), i64 ptrtoint (ptr @___asan_gen_ to i64), i64 0, i64 0, i64 ptrtoint (ptr @__odr_asan_gen_g1 to i64) }]

@1 = private alias { i32, [28 x i8] }, ptr @g0
@2 = private alias { i32, [28 x i8] }, ptr @g1

define internal void @asan.module_ctor() #0 {
  call void @__asan_init()
  call void @__asan_version_mismatch_check_v8()
  call void @__asan_register_globals(i64 ptrtoint (ptr @0 to i64), i64 2)
  ret void
}
```

`asan.module_ctor` references the metadata array `@0`, which, in turn, references `@1` and `@2`. `@1` and `@2` reference the global variables `g0` and `g1`, respectively.
This unfortunately indicates that `g0` and `g1` cannot be discarded by section-based garbage collection.

It's important to note that this version of `asan.module_ctor` is not placed within a COMDAT group.
In another compile unit, a separate `asan.module_ctor` references a different metadata array.
As a result, these `asan.module_ctor` functions cannot share the same implementation.

In a linked component, both `__asan_init` and `__asan_version_mismatch_check_v8` will be called multiple times, incurring a small overhead.

Regrettably, the default setting of `-fsanitize-address-globals-dead-stripping` in Clang 17 had a bug. Specifically, when there are no global variables, and the unique module ID is non-empty, a COMDAT `asan.module_ctor` is created without any `__asan_register_elf_globals` calls. If this COMDAT is selected as the prevailing copy by the linker, the linkage unit will lack a `__asan_register_elf_globals` call, resulting in an unpoisoned redzone and a non-functional ODR violation checker.

I have fixed this in the main branch ([#67745](https://github.com/llvm/llvm-project/pull/67745)) but LLVM 17.0.2 does not contain the fix.

### Global variable metadata

Before Clang 15, Clang's instrumentation included `llvm.asan.globals`, and the AddressSanitizer runtime required its object file feature for symbolization.

<https://reviews.llvm.org/D127552> enabled debug information for symbolization and <https://reviews.llvm.org/D127911> deleted the metadata node `llvm.asan.globals`.

## initialization-order-fiasco

AddressSanitizer provides a check to detect whether a dynamic initializer for one global variable accesses dynamically initialized global variables defined in another compile unit, which helps identify certain initialization order issues.
This catches certain initialization order fiasco issues.

Here is an example:
```sh
cat > a0.cc <<'eof'
#include <stdio.h>
extern int a1;
static int fa0() { return 1; }
int a0 = fa0();
int main() { printf("%d %d\n", a0, a1); }
eof
cat > a1.cc <<'eof'
extern int a0;
static int fa1() { return a0+1; }
int a1 = fa1();
eof
clang++ -fsanitize=address a0.cc a1.cc -o a
```
```
% ASAN_OPTIONS=strict_init_order=1 ./a
=================================================================
==124921==ERROR: AddressSanitizer: initialization-order-fiasco on address 0x5577b1cd6b00 at pc 0x5577b12fbbca bp 0x7ffe75a0a280 sp 0x7ffe75a0a260
READ of size 4 at 0x5577b1cd6b00 thread T0
    #0 0x5577b12fbbc9 in fa1() /tmp/t/d/a1.cc:2:27
    #1 0x5577b12fbbec in __cxx_global_var_init /tmp/t/d/a1.cc:3:10
    #2 0x5577b12fbc64 in _GLOBAL__sub_I_a1.cc /tmp/t/d/a1.cc
    #3 0x7ff44e0107f5 in call_init csu/../csu/libc-start.c:145:3
    #4 0x7ff44e0107f5 in __libc_start_main csu/../csu/libc-start.c:347:5
    #5 0x5577b11b46d0 in _start (/tmp/t/d/a+0x766d0)

0x5577b1cd6b00 is located 0 bytes inside of global variable 'a0' defined in '/tmp/t/d/a0.cc:4' (0x5577b1cd6b00) of size 4
  registered at:
    #0 0x5577b11d1da4 in __asan_register_globals /usr/local/google/home/maskray/llvm/compiler-rt/lib/asan/asan_globals.cpp:363:3
    #1 0x5577b11d2181 in __asan_register_elf_globals /usr/local/google/home/maskray/llvm/compiler-rt/lib/asan/asan_globals.cpp:346:3
    #2 0x5577b12fbb57 in asan.module_ctor a0.cc
    #3 0x7ff44e0107f5 in call_init csu/../csu/libc-start.c:145:3
    #4 0x7ff44e0107f5 in __libc_start_main csu/../csu/libc-start.c:347:5

SUMMARY: AddressSanitizer: initialization-order-fiasco /tmp/t/d/a1.cc:2:27 in fa1()
...
```

When `check_initialization_order` is enabled, while `strict_init_order` is disabled, AddressSanitizer performs a weak check allowing a compile unit that is about to be initialized to access global variables in an already initialized compile unit.
In this scenario, the previous example does not result in an error:
```
% ASAN_OPTIONS=check_initialization_order=1:strict_init_order=0 ./a
1 2
```

For the following case, the weak check can still catch the initialization order fiasco:
```
cat > a0.cc <<'eof'
#include <stdio.h>
extern int a1;
int a0 = []() { return a1-1; }();
int main() { printf("%d %d\n", a0, a1); }
eof
cat > a1.cc <<'eof'
extern int a0;
static int fa1() { return 2; }
int a1 = fa1();
eof
clang++ -g -fsanitize=address a0.cc a1.cc -o a
ASAN_OPTIONS=check_initialization_order=1:strict_init_order=0 ./a
```

Clang translates C++ dynamic initialization into a global initialization function within the `llvm.global_ctors` list. AddressSanitizer augments this global initialization function with `__asan_before_dynamic_init` and `__asan_after_dynamic_init`. These two functions work together to check for initialization order issues when `check_initialization_order` is enabled.

For instrumented global variables with initializers, the `has_dynamic_init` variable in the `__asan_global` metadata is set to true. These variables are collected into the `dynamic_init_globals` array.

`__asan_before_dynamic_init` is called for each compile unit. This function iterates over `dynamic_init_globals` and poisons those whose `DynInitGlobal::initialized` value is false.
Subsequently, the global initialization function is executed. If it accesses the poisoned memory, it triggers a report for an initialization order issue.
Following this, `__asan_after_dynamic_init` processes these global variables, unpoisoning them.

```cpp
void __asan_before_dynamic_init(const char *module_name) {
  ...
  for (uptr i = 0, n = dynamic_init_globals->size(); i < n; ++i) {
    DynInitGlobal &dyn_g = (*dynamic_init_globals)[i];
    const Global *g = &dyn_g.g;
    if (dyn_g.initialized)
      continue;
    if (g->module_name != module_name)
      PoisonShadowForGlobal(g, kAsanInitializationOrderMagic);
    else if (!strict_init_order)
      dyn_g.initialized = true;
  }
}

void __asan_after_dynamic_init() {
  ...
  for (uptr i = 0, n = dynamic_init_globals->size(); i < n; ++i) {
    DynInitGlobal &dyn_g = (*dynamic_init_globals)[i];
    const Global *g = &dyn_g.g;
    if (!dyn_g.initialized) {
      // Unpoison the whole global.
      PoisonShadowForGlobal(g, 0);
      // Poison redzones back.
      PoisonRedZones(*g);
    }
  }
}
```

The check is applicable when the accessed variable resides in another linked unit.

For example, consider that `b.so` consists of `b0.cc` and `b1.cc`, while the main executable `a` contains `a0.cc` and `a1.cc`.
```sh
cat > a0.cc <<'eof'
#include <stdio.h>
extern int a1, b0, b1;
static int fa0() { return 1; }
int a0 = fa0();
int main() { printf("%d %d %d %d\n", a0, a1, b0, b1); }
eof
echo 'static int fa1() { return 2; } int a1 = fa1();' > a1.cc
echo 'static int fb0() { return 3; } int b0 = fb0();' > b0.cc
echo 'static int fb1() { return 4; } int b1 = fb1();' > b1.cc
sed 's/^        /\t/' > Makefile <<'eof'
.MAKE.MODE := meta curDirOk=true
CXX := clang++
CXXFLAGS := -g -fsanitize=address
a: a0.cc a1.cc b.so
        ${LINK.cc} -Wl,-rpath=. $> -o $@
b.so: b0.cc b1.cc
        ${LINK.cc} -fpic -shared $> -o $@
clean:
        rm -f *.meta a b.so
eof
bmake
```

In `check_initialization_order=1,strict_init_order=0` mode,

* globals in `b0.cc` and `b1.cc` are registered
* b0.cc: `__asan_before_dynamic_init` marks `b0` as initialized and poisons `b1`. Global initialization is run. `__asan_register_globals` unpoisons `b1`
* b1.cc: `__asan_before_dynamic_init` marks `b1` as initialized and poisons `b0`. Global initialization is run. `__asan_register_globals` unpoisons `b0`
* globals in `a0.cc` and `a1.cc` are registered
* a0.cc: `__asan_before_dynamic_init` marks `a0` as initialized and poisons `a1`. Global initialization is run. `__asan_register_globals` unpoisons `a1`
* a1.cc: `__asan_before_dynamic_init` marks `a1` as initialized and poisons `a0`. Global initialization is run. `__asan_register_globals` unpoisons `a0`

In `check_initialization_order=1,strict_init_order=1` mode,

* globals in `b0.cc` and `b1.cc` are registered
* b0.cc: `__asan_before_dynamic_init` poisons `b1`. Global initialization is run
* b1.cc: `__asan_before_dynamic_init` poisons `b0`. Global initialization is run
* globals in `a0.cc` and `a1.cc` are registered
* a0.cc: `__asan_before_dynamic_init` poisons `b0,b1,a1`. Global initialization is run. `__asan_register_globals` unpoisons `b0,b1,a1`
* a1.cc: `__asan_before_dynamic_init` poisons `b0,b1,a0`. Global initialization is run. `__asan_register_globals` unpoisons `b0,b1,a0`

Note, violations due to `b.so` accessing `a` cannot be detected.

The instrumentation can be disabled with an entry in `asan_ignorelist.txt`:
```
global:var=init
```

An initialization-order-fiasco error cannot be suppressed using `ASAN_OPTIONS=suppressions=a.supp`.
