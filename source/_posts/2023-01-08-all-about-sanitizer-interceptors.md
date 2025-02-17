layout: post
title: All about sanitizer interceptors
author: MaskRay
tags: [llvm,sanitizer]
---

Many sanitizers want to know every function in the program.
User functions are instrumented and therefore known by the sanitizer runtime.
For library functions, some (e.g. mmap, munmap, memory allocation/deallocation functions, longjmp, vfork) need special treatment.
Sanitizers leverage symbol interposition to redirect such function calls to its own implementation: interceptors.
Other library functions can be treated as normal user code. Either instrumenting the function or providing an interceptor is fine.

In some cases instrumenting is infeasible:

* Assembly source files usually do not (or are inconvenient to) call sanitizer callbacks
* Many libc implementations cannot be instrumented. [When can glibc be built with Clang?](/blog/2021-10-10-when-can-glibc-be-built-with-clang#asm-label-after-first-use)
* Some functions have performance issues if instrumented instead of intercepted (mostly `mem*` and `str*`)

And interceptors may be the practical choice.

This article talks about how interceptors work and the requirements of sanitizer interceptors.

<!-- more -->

## How interceptors work

Here is the short summary and I will elaborate.

* ELF platforms: `__interceptor_$name` has a weak alias `$name` of the same name as the intercepted function.
* Apple platforms: `__DATA,__interpose` holds the address of the interceptor (`wrap_$name`) and the intercepted function.
* Windows: hot patching the intercepted function

### ELF platforms

In Clang, sanitizer runtime files are named `$resource_dir/lib/$triple/libclang_rt.*.{a,so}` on ELF platforms.
In the older `LLVM_ENABLE_PER_TARGET_RUNTIME_DIR=off` configuration (default before LLVM 15.0.0), the files are named `$resource_dir/lib/libclang_rt.*-$arch.{a,so}`.
The `.a` files are called static runtime while the `.so` files are called shared runtime or dynamic runtime.
As of 2023-01, Android and Fuchsia default to shared runtime while other ELF platforms (Linux, \*BSD, etc) default to static runtime.

In GCC, sanitizer runtime files are named `lib*san.{a,so}`. Shared runtime is the default.
Specify `-static-libasan` to use static runtime.

### Static runtime

Most static runtime files are only used when linking executables.
In the `-static-libsan` mode, when linking an executable, Clang Driver passes `--whole-archive $resource_dir/lib/libclang_rt.$name.a --no-whole-archive` to the linker.
For the following example, if `libclang_rt.$name.a` defines `malloc` and `free`, the executable will get the definitions.

```sh
printf '#include <stdlib.h>\nint main() { void *p = malloc(42); free(p); }' > a.c
clang -fsanitize=address a.c -o a
```

`malloc`, `free`, and actually all libc interceptors are exported to `.dynsym` because they are defined/referenced by a link-time shared object (glibc `libc.so.6`), even if `-Wl,--export-dynamic` is not specified.
See [Explain GNU style linker options#--export-dynamic](/blog/2020-11-15-explain-gnu-linker-options#export-dynamic) for detail.
`-Wl,--gc-sections` cannot discard these interceptors as `.dynsym` symbols are considered GC roots.

Also, note that the definitions are weak and unversioned.

```text
% nm -D a | grep -w 'malloc\|free'
00000000000ead10 W free
00000000000eb010 W malloc
```

When linking a shared object, the static runtime is not used. The linker option `-z defs` (`--no-undefined`) may lead to undefined symbol errors.

On Linux glibc, the shared object has a versioned reference.

```sh
printf '#include <stdlib.h>\nvoid *foo() { return malloc(42); }' > b.c
clang -fsanitize=address -fpic -shared b.c -o b.so
```
```text
% nm -D b.so | grep 'malloc\|free'
                 U malloc@GLIBC_2.2.5
```

If we make `b.so` a link-time dependency of the executable `a` or dlopen `b.so` at run-time, the `malloc@GLIBC_2.2.5` reference from `b.so` will be bound to the definition in `a`.
The dynamic loader computes a breadth-first symbol search list (`executable, needed0, needed1, needed2, needed0_of_needed0, needed1_of_needed0, ...`).
For each symbol reference, the dynamic loader iterates over the list and finds the first component which provides a definition.
The executable provides a definition and the search stops at the executable.
See the first few paragraphs of [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic).

Actually we also use a rule that a `malloc@GLIBC_2.2.5` reference can be bound to a `malloc` definition of `VER_NDX_GLOBAL`.
See [All about symbol versioning#rtld-behavior](https://maskray.me/blog/2020-11-26-all-about-symbol-versioning#rtld-behavior).

Since the executable defines `malloc` and `free`, its calls to the two symbols do not use PLT.
This is an advantage over shared runtime. Calls from the shared object still need PLT.
Preloading malloc functions does not work since the executable's definitions take priority.

From LLVM 17 onwards, [an interceptor involves four symbols](https://reviews.llvm.org/D151085).
`func` has a `STB_WEAK` binding and aliases a thunk (`__interceptor_trampoline_func`) that jumps to a weak function `__interceptor_func`.
`__interceptor_func` aliases a `STB_GLOBAL` function `___interceptor_func`, which provides the sanitizer interceptor implementation.

```
% readelf -Ws $(clang --print-file-name=libclang_rt.asan.a) | grep -E 'malloc$'
  3638: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND ___interceptor_malloc
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT   11 .text.___interceptor_malloc
    51: 000000000000000a     5 FUNC    WEAK   DEFAULT    2 malloc
    52: 000000000000000a     5 FUNC    GLOBAL DEFAULT    2 __interceptor_trampoline_malloc
    53: 0000000000000000   270 FUNC    WEAK   DEFAULT   11 __interceptor_malloc
   107: 0000000000000000   270 FUNC    GLOBAL DEFAULT   11 ___interceptor_malloc
```

```asm
malloc:
__interceptor_trampoline_malloc:
  jmp __interceptor_malloc

__interceptor_malloc:
___interceptor_malloc:
  // sanitizer interceptor implementation
```

User code can define a `STB_GLOBAL` `func` to [override](https://github.com/llvm/llvm-project/commit/7fb7330469af52ae1313b2b47c273e62c61a4dd5) the interceptor and call `__interceptor_func` to get the sanitizer definition.

`__interceptor_func` is weak so that it can be overridden by an out-of-tree sampling interceptor.
That out-of-tree sampling interceptor runtime may define a `STB_GLOBAL` `__interceptor_func` to call the real `malloc` for most times and sometimes call the sanitizer interceptor `___interceptor_malloc`. 

Before LLVM 17, an interceptor is available as two symbols, one named `__interceptor_$name` has a `STB_GLOBAL` binding and the other named `$name` has a `STB_WEAK` binding.

```
% readelf -Ws $(clang-16 --print-file-name=libclang_rt.asan.a) | grep -E ' (malloc|__interceptor_malloc)$'
  2196: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND __interceptor_malloc
    57: 0000000000000000   295 FUNC    GLOBAL DEFAULT   10 __interceptor_malloc
   121: 0000000000000000   295 FUNC    WEAK   DEFAULT   10 malloc
```

### Shared runtime

In Clang, Android, Fuchsia, and Darwin uses shared runtime by default. GCC defaults to this mode for Linux.
On targets that default to static runtime, use `-shared-libsan` to select this configuration.
Both executables and shared objects will link against `libclang_rt.$name.so`.
Here is an example on Linux glibc.

```text
% clang -fsanitize=address -shared-libsan a.c -o a
% readelf -Wd a | grep 'clang_rt\|libc'
 0x0000000000000001 (NEEDED)             Shared library: [libclang_rt.asan.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
% readelf -d b.so | grep 'clang_rt\|libc'
 0x0000000000000001 (NEEDED)             Shared library: [libclang_rt.asan.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
% readelf -W --dyn-syms $(clang --print-file-name=libclang_rt.asan.so) | grep -w malloc
  1295: 000000000011a2d0   295 FUNC    WEAK   DEFAULT   11 malloc
```

In the symbol search list, `libclang_rt.asan.so` appears before `libc.so.6`, so `malloc` references from `a` and `b.so` will be bound to the definition `malloc` in `libclang_rt.asan.so`.
The rule that a versioned reference can be bound to a definition of `VER_NDX_GLOBAL` kicks in again.

Preloading a shared object is dangerous and the asan runtime warns about it.

```
% clang -fsanitize=address -shared-libsan -Wl,-rpath=$(dirname $(clang --print-file-name=libclang_rt.asan.so)) a.c -o a
% LD_PRELOAD=/lib/x86_64-linux-gnu/libjemalloc.so.2 ./a
==1650190==ASan runtime does not come first in initial library list; you should either link runtime to your application or manually preload it with LD_PRELOAD.
```

The `dlopen` interceptor [does not work properly](https://github.com/llvm/llvm-project/issues/28164) when the `DT_RUNPATH` tag is used.

### `dlsym RTLD_NEXT`

An interceptor needs to call the intercepted library function.
During initialization, the runtime calls `dlsym(RTLD_NEXT, $name)` for all intercepted functions and save the addresses.
An interceptor `__interceptor_$name` calls the saved return value of `dlsym(RTLD_NEXT, $name)`.

In the previous examples, whether the definition is in the executable or `libclang_rt.asan.so`, the component is prior to `libc.so.6` in the symbol search list.
Therefore `dlsym(RTLD, $name)` will retrieve the symbol address in `libc.so.6`.

Before glibc 2.36, `dlsym(RTLD_NEXT, name)` returned the address of the oldest version definition of _name_ in `libc.so.6`. And we got to use the old semantics.
This is usually benign but not ideal (see <https://github.com/google/sanitizers/issues/1371> for a `regexec` issue).
I fixed glibc 2.36 ([BZ #14932](https://sourceware.org/bugzilla/show_bug.cgi?id=14932)) so that `dlsym(RTLD_NEXT, name)` returns the default version definition now.

This fix led to an interesting issue.
glibc [made](https://sourceware.org/git/?p=glibc.git;a=commit;h=99f841c441feeaa9a3d97fd91bb3d6ec8073c982) `__pthread_mutex_lock` a non-default version definition in 2.34, so `dlsym(RTLD_NEXT, "__pthread_mutex_lock")` would return NULL.
I fixed it by [disabling the interceptor](https://github.com/llvm/llvm-project/commit/fb32a69855341630a7084364c66083264e1d18bf) if glibc>=2.34 at build time.

Since `dlsym RTLD_NEXT` is called so many times, the rtld overhead is a significant bottleneck when an executable loads O(1000) DSOs.
Most rtld implementations iterate over all the DSOs and will only find a symbol at very end of the search list (e.g. `libc.so.6` for glibc).

### Apple platforms

On Apple platforms, sanitizers use a dyld feature to intercept symbols. Only dynamic runtime is supported.

The runtime defines a special section `__DATA,__interpose` which contains a list of pairs holding the addresses of the interceptor (named `wrap_$name`) and the intercepted function.
dyld will redirect calls to the interceptor, as long as the call is not coming from the library defining the interceptor.

Let's say we have an interceptor `wrap_strcmp`. It calls the real definition `strcmp`.
dyld will bind the reference to the real definition in `libsystem_kernel.dylib`, instead of `wrap_strcmp`.

```cpp
// compiler-rt/lib/interception/interception.h
// For a function foo() and a wrapper function bar() create a global pair
// of pointers { bar, foo } in the __DATA,__interpose section.
// As a result all the calls to foo() will be routed to bar() at runtime.
#define INTERPOSER_2(func_name, wrapper_name) __attribute__((used)) \
const interpose_substitution substitution_##func_name[] \
    __attribute__((section("__DATA, __interpose"))) = { \
    { reinterpret_cast<const uptr>(wrapper_name), \
      reinterpret_cast<const uptr>(func_name) } \
}
```

Let's use a reduced example to get a feeling.
```sh
cat > a.c <<eof
#include <string.h>

int initialized;
int main(int argc, char *argv[]) {
  initialized = 1;
  return strcmp(argv[1], argv[2]);
}
eof
cat > b.c <<eof
#include <stdio.h>
#include <string.h>

extern int initialized;
int wrap_strcmp(const char *a, const char *b) {
  if (initialized)
    printf("strcmp: %s %s\n", a, b);
  return strcmp(a, b);
}

__attribute__((used, section("__DATA, __interpose")))
static void *const interpose[] = {(void *)wrap_strcmp, (void *)strcmp};
eof
clang -dynamiclib -Wl,-U,_initialized b.c -o b.dylib  # or use -Wl,-undefined,dynamic_lookup
clang a.c b.dylib -o a
./a a a
```

### Windows

See [AddressSanitizer for Windows, 2014 LLVM Developers' Meeting](https://llvm.org/devmtg/2014-10/Slides/ASan%20for%20Windows.pdf).
The current implementation is at `compiler-rt/lib/interception/interception_win.cpp`.
Interceptors are implemented by hot patching the entry block of the intercepted function.

Linking runtime has multiple ways.

For `/MT`,

* Some functions (e.g. `malloc`) are defined by the lib. The runtime just defines the interceptor as the same name.
* Many functions are dllimported. The runtime calls `__interception::OverrideFunction` to try multiple hot patching techniques.

For `/MD`, the runtime maintains a list of interesting DLLs, checks which DLL defines the intercepted function, and calls `__interception::OverrideFunction` for hot patching.

After hot patching, the first instruction of the intercepted function will jump to the interceptor either directly or through a trampoline.
The interceptor will jump back.

`compiler-rt/lib/asan/asan_win_dll_thunk.cpp` needs to be updated for new asan interceptors.

## Interceptor requirements

### AddressSanitizer

AddressSanitizer detects addressability bugs.
For a mapped memory region, AddressSanitizer uses shadow memory to track whether user bytes are unaddressable (poisoned): accesses are considered a bug (`heap-buffer-overflow, heap-use-after-free, stack-buffer-overflow, stack-use-after-{return,scope}`, etc).
8 (granule) aligned user bytes are mapped to one shadow memory byte.
(This can be patched to support other granules, e.g. now-deleted Myriad RTEMS used 32 as the granule.)

A shadow memory byte is 0 (_unpoisoned_, all 8 bytes are addressable) or a non-zero integer (_poisoned_, not all 8 bytes are addressable).
The non-zero integer may be smaller than 8 (the first X bytes are addressable) or a predefined special value (to indicate a bug category).

At the start of an interceptor, `AsanInitFromRtl` is called if the runtime hasn't been initialized yet.

mmap and munmap do not need special treatment.

When a chunk of memory is reserved from a mapped region for heap allocation, the associated shadow memory is poisoned with 0xfa (`kAsanHeapLeftRedzoneMagic`).
For a malloc-family function, its interceptor records the allocation information (thread ID, requested size, stack trace, allocation type (`malloc, new, new[]`), etc) and unpoisons the shadow (sets to zeros) which may be 0xfa if unallocated previously.

For a free-family function, its interceptor detects double free and alloc-dealloc type mismatch bugs, records the deallocation information (thread ID, stack trace), and poisons the shadow with 0xfd (`kAsanHeapFreeMagic`).
Instrumented/intercepted accesses to the deallocated memory will cause an error.

For a library function which performs memory reads or writes, its interceptor emulates an instrumented memory read/write: check the shadow memory and report an error in case of a poisoned byte.

Some library functions allocate memory internally. An implementation typically carefully uses an interposable symbol `malloc` instead of a private alias, so the allocation/deallocation will be known by AddressSanitizer and be unpoisoned/poisoned properly.

An non-special function which is neither instrumented nor intercepted just leads to fewer detected errors.

#### Stack use after scope

An instrumented function poisons stack variables to catch stack use after scope bugs.
Instrumentation unpoisons stack variables before an epilogue.
If a process creates a subprocess with shared memory, and the subprocess exits due to a noreturn function (including throw expressions), the unpoisoned shadow may cause a [false positive](https://github.com/google/sanitizers/issues/37) to the first process.
This is fixed by calling `__asan_handle_no_return` before calling noreturn functions to conservatively unpoison the whole stack.

`vfork` has [similar](https://github.com/google/sanitizers/issues/925) issues and needs an interceptor.

longjmp-family functions have similar issues. The stack memory may be reused causing a false positive.
These functions are intercepted in the runtime to call `__asan_handle_no_return`.

https://github.com/android/ndk/issues/988

### HWAddressSanitizer

HWAddressSanitizer detects addressability bugs (the same class of errors as the main feature of AddressSanitizer) using a different algorithm (software memory tagging).
16 (granule) aligned user bytes are associated with a non-zero tag. The tag is implemented as one byte and stored in the shadow memory.

A memory allocation chooses a random non-zero tag and sets it in the high bits of the returned pointer.
The shadow memory of the allocated chunk is filled with the tag.
To support accessing the pointer with non-zero high bits, hardware features (ARM Top Byte Ignore, Intel Linear Address Masking, RISC-V Pointer Masking) or page aliases are needed.

The interceptor behavior is similar to AddressSanitizer.

mmap and munmap do not need special treatment.

For an instrumented memory read or write operation, its interceptor emulates an instrumented memory read/write: check the pointer tag against the tag stored in the shadow memory, and report an error in case of a mismatch.

To detect use-after-free, a memory deallocation needs to clear the associated shadow memory.

For an interceptor, do something similar to AddressSanitizer: check the pointer tag against the shadow memory and report an error in case of a mismatch.

HWAddressSanitizer is [deployed](https://source.android.com/docs/security/test/hwasan) on Android.
Its C library bionic is instrumented so that very few interceptors are needed.

To use HWAddressSanitizer with glibc in the future, either interceptors need to be provided or glibc can be instrumented.

longjmp-family functions have issues similar to AddressSanitizer. The stack memory may be reused causing a false positive.
These functions are intercepted to call `__hwasan_handle_longjmp` to clear the shadow memory.
`vfork` needs an interceptor similar to AddressSanitizer.

### ThreadSanitizer

In the old runtime (tsan v2), 8 aligned user bytes are mapped to a shadow cell of 32 bytes, which contains 4 shadow values.
The representation uses 13 bits to record a thread ID (up to 8192 threads are supported), and 42 bits to record a vector clock timestamp.

In the new runtime (tsan v3), 8 aligned user bytes are mapped to a shadow cell of 16 bytes, which contains 4 shadow values.
A shadow value records the bitmask of accessed bytes (8 bites), a thread slot ID (8 bits), a vector clock timestamp (14 bites), is\_read (1 bit), is\_atomic (1 bit).
The shrinking of time is made available because the timestamp increments more slowly (only on atomic releases, mutex unlocks, thread creation/destruction).

At the start of an interceptor, the runtime calls `cur_thread_init` and retrieves the thread state and the return address of the current function.
For a library function which performs memory reads or writes, its interceptor emulates an instrumented memory read/write: record the access as a thread event (`EventAccess`), form a new shadow value, and check existing shadow values (at most 4) in the shadow cell.
If the current shadow value and a previous shadow value interact in the bitmask of accessed bytes, have different thread slot IDs, have at least one write, and have at least one non-atomic access, report a data race.
Otherwise replace one shadow value with the new one.

pthread mutex functions such as `pthread_mutex_{init,destroy,lock,trylock,timedlock,unlock}` (and `pthread_{rwlock,spin,cond,barrier}_*` `pthread_once`) are intercepted to record mutex lifetime and synchronization points.

Most libc functions do not have synchronization semantics.

An non-special function which is neither instrumented nor intercepted just leads to fewer detected errors.

### MemorySanitizer

MemorySanitizer uses shadow memory to track whether a memory region has uninitialized values.

One user byte is mapped to one shadow memory byte.
A shadow memory byte is 0 (_unpoisoned_, all 8 bits are initialized) or a non-zero integer (_poisoned_, some bits are uninitialized).

At the start of an interceptor, `__msan_init` is called if the runtime hasn't been initialized yet.
Then `__errno_location()` is unpoisoned.
To support `-fsanitize=memory,fuzzer`, interceptors introduce a small overhead by [checking](https://reviews.llvm.org/D48891) whether interceptors are disabled due to libFuzzer.
This is so that the libFuzzer runtime does not need to be instrumented by MemorySanitizer.

For a library function which performs memory reads: if the shadow memory is poisoned, report a use-of-uninitialized-value error.
For a library function which performs memory writes: unpoison the shadow memory, i.e. mark the memory region as initialized.

If an uninstrumented function which performs memory writes does not have an interceptor, the lack of unpoisoning may lead to false positives when the memory is subsequently read.
This property is different from AddressSanitizer/ThreadSanitizer where a missing interceptor usually just leads to fewer detected errors.

### DataFlowSanitizer

[DataFlowSanitizer](https://clang.llvm.org/docs/DataFlowSanitizer.html) is a dynamic data flow analysis (taint analysis) tool.
It allows tagging a user byte with up to 8 labels. Compiler instrumentation propagates labels when a user byte affects the computation of another one.
This process is similar to uninitialized value propagation in MemorySanitizer.
A user byte is mapped to a shadow memory byte which supports 8 labels.

At the start of an interceptor, `dfsan_init` is called if the runtime hasn't been initialized yet.
Then the label of `__errno_location()` is cleared.

For a malloc-family or free-family function, its interceptor clears labels (assuming the bytes are unaffected by other values) by default.

For a library function which performs memory writes or returns a value, its interceptor propagates the label of the source value.

An non-special function which performs memory writes and is neither instrumented nor intercepted misses label propagation.
This may cause false negatives.

### NumericalSanitizer

NumericalSanitizer uses shadow memory to track floating point types and values.
The usage is similar to MemorySanitizer. However, false positive rates are significantly lower.
This is because the false positive scenarios are not likely utilized:

* Allocate, store a floating point, and deallocate
* calloc/mmap, which actually reuses the previous allocation with a non-zero floating point value
* Perform a read-write operation like `*p += f`. The shadow memory holds a non-zero floating point value, mismatching the actual zero value.

### Standalone LeakSanitizer

For most major 64-bit platforms (except Apple), AddressSanitizer integrates and enables LeakSanitizer by default.
LeakSanitizer can be used standalone as well to just detect memory leak bugs.
See [All about LeakSanitizer](/blog/2023-02-12-all-about-leak-sanitizer).

Standalone LeakSanitizer intercepts very few functions: malloc-family and free-family functions (like a preloaded memory allocator), and a few functions like `pthread_create`.

At the start of an interceptor, `__lsan_init` is called if the runtime hasn't been initialized yet.

For a malloc-family function, its interceptor records the allocation information (requested size, stack trace).

For `pthread_create`, its interceptor [ensures](https://github.com/llvm/llvm-project/commit/bdeff959a166594abf76206d0eae397e4cc17747) correct thread ID, ignores allocations from the real `pthread_create`, and registers the new thread.

At exit time, LeakSanitizer performs a GC style stop-the-world and scans all reachable memory chunks.
For an unreachable chunk, report an error using the recorded information.

### Scudo hardened allocator and GWP-ASan

Scudo is a user-mode memory allocator designed to be resilient against heap-related vulnerabilities.
[GWP-ASan](https://llvm.org/docs/GwpAsan.html) uses Scudo as the main allocator and lets Scudo hand over sampled memory allocations to its own memory allocator.

In Scudo, only memory allocator related functions are intercepted.

### MemProf

TODO MemProf isn't a sanitizer, but it uses sanitizer interceptors.

## Portability and maintainability

Sanitizers support many operating systems and architectures.

* <https://clang.llvm.org/docs/AddressSanitizer.html#supported-platforms>
* <https://clang.llvm.org/docs/MemorySanitizer.html#supported-platforms>
* <https://clang.llvm.org/docs/ThreadSanitizer.html#supported-platforms>
* ...

Implementing interceptors (though the code can be shared) may be the most challenging part porting a sanitizer to a new platform.

* A libc implementation more or less supports some extensions. These functions need to be intercepted.
* Type definitions from a newer standard may not be supported by an implementation. A version dispatch is needed or a shim may be provided.

Sanitier runtime is therefore scattered with `#if` conditional inclusions.
Runtime tests with combinatorial explosions sometimes make debugging tricky.

Therefore, supporting a new operating system should be taken very carefully.
I believe sanitizer maintainers tend to focus on the ability to instrument libc and provide sanitizer callbacks if a new OS is ever considered.
One may ask whether this improses pressure when sanitizer runtime gets updated and needs to work with older libc versions.
It seems that the intercepted read/write behavior part is quite stable in the sanitizer runtime, so working with multiple libc versions seems fine.

### glibc

As mentioned at the beginning of this article, glibc [cannot be built wglibc ith Clang](/blog/2021-10-10-when-can-glibc-be-built-with-clang#asm-label-after-first-use).
I hope that this will change in the next few years.

OK, an observation: new releases of glibc oftentimes require adaptation from sanitizer interceptors.

The Linux kernel user-space API and glibc traditionally did not play well.
The discord can occasionally cause portibility issues to projects.
Sanitizer interceptors use `#include <linux/*.h>` a lot and are subject to such issues.
E.g. I landed <https://reviews.llvm.org/D129471> to remove `#include <linux/fs.h>` to resolve `fsconfig_command/mount_attr` conflict with glibc 2.36.
TODO mention why it is difficult to pull `#include <linux/*.h>` out of `compiler-rt/lib/sanitizer_common/sanitizer_platform_limits_posix.cpp`.

### musl

I [ported](https://github.com/llvm/llvm-project/commit/7afdc89c2054d6fa3a6c2f2f24411becfc764676) `asan, cfi, lsan, msan, tsan, ubsan` to Linux musl in 2021-01 and learned a lot in the process.
It seemed that code readability improved in some places (where glibc-ism was refined with a better condition, e.g. change from "Linux but not Android" or "all OSes but not X/Y/Z" (where X/Y/Z basically enumerates all non-glibc platforms) to "Linux glibc").
Maintaining the port required low efforts. I applied very few fixes in 2021 and 2022.

musl provides `_LARGEFILE64_SOURCE` symbols (`mmap64, pread64, readdir64, stat64`, etc) for glibc ABI compatibility which are not intended for linking.
musl 1.2.4 will disallow linking against LFS64 symbols[ therefore these interceptors have to be [removed](https://reviews.llvm.org/D141186).

## Interesting problems

### Non-intercepted library function calls an intercepted library function

#### MemorySanitizer/DataFlowSanitizer and ptsname

`ptsname` used not to be intercepted and this caused false positives with MemorySanitizer.

In glibc, `ptsname` calls `__ptsname_r` with a static variable (unknown to MemorySanitizer, having a zero value shadow).
Since `__ptsname_r` is not instrumented, the shadow memory of `numbuf` is whatever the last shadow value for the memory region that `numbuf` happens to occupy.
If `memcpy` is interposable, its interceptor copies the incorrect (poisoned) shadow to `buf`, which may cause a false positive if `buf` is accessed by the caller.
```c
int
__ptsname_r (int fd, char *buf, size_t buflen)
{
  ...
      char numbuf[21];
  ...
      numbuf[sizeof (numbuf) - 1] = '\0';
      p = _itoa_word (ptyno, &numbuf[sizeof (numbuf) - 1], 10, 0);
  ...
      memcpy (__stpcpy (buf, devpts), p, &numbuf[sizeof (numbuf)] - p);
  ...
}
```

The fix is to [instrument](https://reviews.llvm.org/D88547) `ptsname` and `ptsname_r`.

In glibc before 2018 ([BZ #18822](https://sourceware.org/bugzilla/show_bug.cgi?id=18822)), many functions had PLT calls.
There were many lurking issues like the above.

### `_FORTIFY_SOURCE`

In short, disable `_FORTIFY_SOURCE` when a sanitizer is used.
If `-D_FORTIFY_SOURCE=2` is appended after your specified options, use `-Wp,-U_FORTIFY_SOURCE` to override it.

When `_FORTIFY_SOURCE` is enabled, some library functions are redirected to `*_chk`.
Interceptors don't provide `*_chk` (with an exception <https://reviews.llvm.org/D40951>), so we just end up with fewer detected errors (e.g. asan, tsan) or false positives (msan).

See <https://github.com/google/sanitizers/issues/247>.

### Android linker namespace and libc++ interceptor

<https://github.com/android/ndk/issues/988>
