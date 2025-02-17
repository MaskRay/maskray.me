layout: post
title: All about LeakSanitizer
author: MaskRay
tags: [llvm,sanitizer]
---

Clang and [GCC 4.9](https://gcc.gnu.org/PR59061) implemented LeakSanitizer in 2013.
[LeakSanitizer](https://clang.llvm.org/docs/LeakSanitizer.html) (LSan) is a memory leak detector.
It intercepts memory allocation functions and by default detects memory leaks at `atexit` time.
The implementation is purely in the runtime (`compiler-rt/lib/lsan`) and no instrumentation is needed.

LSan has very little architecture-specific code and supports many 64-bit targets.
Some 32-bit targets (e.g. Linux arm/x86-32) are supported as well, but there may be high false negatives because pointers with fewer bits are more easily confused with integers/floating points/other data of a similar pattern.
Every supported operating system needs to provide some way to "stop the world".

<!-- more -->

## Usage

LSan can be used in 3 ways.

* Standalone (`-fsanitize=leak`)
* AddressSanitizer (`-fsanitize=address`)
* HWAddressSanitizer (`-fsanitize=hwaddress`)

The most common way to use LSan is `clang -fsanitize=address` (or `gcc -fsanitize=address`).
For LSan-supported targets (`#define CAN_SANITIZE_LEAKS 1`), the AddressSanitizer (ASan) runtime enables LSan by default.

```
% cat a.c
#include <stdlib.h>
int main() {
  void **p = malloc(42); // leak (categorized as "Direct leak")
  *p = malloc(43);       // leak (categorized as "Indirect leak")
  p = 0;
}
% clang -fsanitize=address a.c -o a
% ./a

=================================================================
==594015==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 42 byte(s) in 1 object(s) allocated from:
    #0 0x55fffef9482e  (/tmp/c/a+0xea82e)
    #1 0x55fffefcf1d1  (/tmp/c/a+0x1251d1)
    #2 0x7f7301626189  (/lib/x86_64-linux-gnu/libc.so.6+0x27189) (BuildId: c4f6727c560b1c33527ff9e0ca0cef13a7db64d2)

Indirect leak of 43 byte(s) in 1 object(s) allocated from:
    #0 0x55fffef9482e  (/tmp/c/a+0xea82e)
    #1 0x55fffefcf1df  (/tmp/c/a+0x1251df)
    #2 0x7f7301626189  (/lib/x86_64-linux-gnu/libc.so.6+0x27189) (BuildId: c4f6727c560b1c33527ff9e0ca0cef13a7db64d2)

SUMMARY: AddressSanitizer: 85 byte(s) leaked in 2 allocation(s).
```

As a runtime-only feature, `-fsanitize=leak` is mainly for link actions.
When linking an executable, with the default `-static-libsan` mode on many targets, Clang Driver passes `--whole-archive $resource_dir/lib/$triple/libclang_rt.lsan.a --no-whole-archive` to the linker.
GCC and some platforms prefer shared runtime/dynamic runtime. See [All about sanitizer interceptors](/blog/2023-01-08-all-about-sanitizer-interceptors#elf-platforms).
```
% clang -fsanitize=leak a.o '-###' |& grep --color lsan
... --whole-archive" "/tmp/Rel/lib/clang/17/lib/x86_64-unknown-linux-gnu/libclang_rt.lsan.a" "--no-whole-archive" ...
% gcc -fsanitize=leak a.o '-###' |& grep --color lsan
... /usr/lib/gcc/x86_64-linux-gnu/12/liblsan_preinit.o --push-state --no-as-needed -llsan ...
```

`-fsanitize=leak` affects compile actions as well, but only for the following C/C++ preprocessor feature.
```c
// Active if -fsanitize=leak
#if __has_feature(leak_sanitizer)
...
#endif

// Active if -fsanitize=leak or -fsanitize=address
#if __has_feature(leak_sanitizer) || __has_feature(address_sanitizer)
#endif
```

## Implementation overview

Standalone LSan intercepts malloc-family and free-family functions. It uses the temporized `SizeClassAllocator{32,64}` with chunk metadata.
The interceptors record the allocation information (requested size, stack trace).

AddressSanitizer and HWAddressSanitizer already intercept malloc-family functions. Their chunk metadata representations have a flag field for LSan.

By default the common options `detect_leaks` and `leak_check_at_exit` are enabled.
The runtime installs a hook with `atexit` which will perform the leak check.
Alternatively, the user can call `__lsan_do_leak_check` to request a leak check before the exit time.

Upon a leak check, the runtime performs a job very similar to a mark-and-sweep garbage collection algorithm.
It suspends all threads ("stop the world") and scans the root set to find reachable allocations.
The root set includes:

* ignored allocations due to `__lsan_ignore_object`, `__lsan::ScopedDisabler`, or `__lsan_disable`
* global regions (`__lsan::ProcessGlobalRegions`). On Linux, these refer to memory mappings due to writable `PT_LOAD` program headers. This ensures that allocations reachable by a global variable are not leaks.
* for each thread, general-purpose registers (default `use_registers=1`), stacks (default `use_stacks=1`), thread-local storage (default `use_tls=1`), and additional pointers in the thread context
* root regions due to `__lsan_register_root_region`
* operating system specific allocations. This is currently macOS-specific: libdispatch and Foundation memory regions, etc.

(When ASan is used and `use_stacks` is enabled, fake stacks are also in the root set for `use-after-return` detection. See `compiler-rt/test/lsan/TestCases/use_after_return.cpp`.)

The runtimes uses a flood fill algorithm to find reachable allocations from the root set.
This done conservatively by finding all aligned bit patterns which look like a pointer.
If the word looks like a pointer into the heap (many 64-bit targets use `[0x600000000000, 0x640000000000)` as the allocator space), the runtime checks whether it refers to an allocated chunk.
In Valgrind's terms, a reference can be a "start-pointer" (a pointer to the start of the chunk) or an "interior-pointer" (a pointer to the middle of the chunk).

Finally, the runtime iterates over all allocated allocations and reports leaks for unmarked allocations.

### Metadata

Each allocation reserves 2 bits to record its state: leaked/reachable/ignored.
For better diagnostics, "leaked" can be direct or indirect (Valgrind memcheck's term).
If a chunk is marked as leaked, all chunks reachable from it are marked as indirectly leaked. (`*p = malloc(43);` in the very beginning example is an indirect leak.)

```cpp
enum ChunkTag {
  kDirectlyLeaked = 0,  // default
  kIndirectlyLeaked = 1,
  kReachable = 2,
  kIgnored = 3
};
```

The idea may be that after the user fixes all direct leaks, indirect leaks will likely go away if directly leaked objects have destructors to deallocate their references.
However, indirect leaks may form a cycle and fixing all direct leaks does not fix these indirect leaks.

```c
#include <stdlib.h>
#include <stdio.h>
int main() {
  void *x = malloc(8), *y = malloc(8);
  *(void **)x = y;   // in a cycle. Indirect leak
  *(void **)y = x;   // in a cycle. Indirect leak
  printf("%p %p\n", x, y);
}
```

Standalone LSan uses this chunk metadata struct:
```cpp
struct ChunkMetadata {
  u8 allocated : 8;  // Must be first.
  ChunkTag tag : 2;
#if SANITIZER_WORDSIZE == 64
  uptr requested_size : 54;
#else
  uptr requested_size : 32;
  uptr padding : 22;
#endif
  u32 stack_trace_id;
};
```

ASan just stores a 2-bit `ChunkTag` in its existing chunk metadata (`__asan::Chunkheader::lsan_tag`).
Similarly, HWASan stores a 2-bit `ChunkTag` in its existing chunk metadata (`__hwasan::Metadata::lsan_tag`).

### Allocator

Sanitizers use very similar allocators. To share code, `compiler-rt/lib/sanitizer_common` defines temporized `SizeClassAllocator{32,64}` as allocator factories.
Each sanitizer instantiates one of `SizeClassAllocator{32,64}` as the primary allocator and picks a secondary allocator.
One template argument describes the metadata size per chunk. E.g. standalone LSan sets the metadata size to `sizeof(ChunkMetadata)`.

One template argument defines size classes (`SizeClassMap`).
A `SizeClassAllocator{32,64}` (primary allocator) instance has a thread-local cache (`SizeClassAllocator{32,64}LocalCache`), which maintains a free list for each of these classes.
Upon an allocation request, if the free list for the requested size class is empty, the runtime will call `Refill` on the cache to grab more chunks from `SizeClassAllocator{32,64}`.
If the free list is not empty, the cache will hand over a chunk from the free list to the user.

`SizeClassAllocator64` has a total allocator space of `kSpaceSize` bytes, splitting into multiple regions of the same size (`kSpaceSize`), each serving a single size class.
Each region is sufficiently large and is assumed to be able to accommodate all allocations of its size class.
During `InitializeAllocator`, `address_range.Init` calls `mmap` with `MAP_PRIVATE | MAP_FIXED | MAP_NORESERVE | MAP_ANON` to mark the whole region as `PROT_NONE`.
When `Refill` is called on the cache, the runtime takes a lock of the `SizeClassAllocator64` region and calls `mmap` to allocate a new memory mapping (and if `kMetadataSize!=0`, another memory mapping for metadata).
For an active allocation, it is very efficient to compute its index in the region and its associated metadata.
Standalone LSan sets the tag to `kDirectlyLeaked` for a new allocation which is not ignored. ASan is similar.

`SizeClassAllocator32` is mainly for 32-bit address spaces but can also be used for 64-bit address spaces.
Like in `SizeClassAllocator64`, the address space splits into multiple regions, but the region size is much smaller (`kRegionSize`; typically 1MiB).
When the cache calls `Refill`, `SizeClassAllocator32` calls `mmap` to allocate a new region serving the requested size class.
The mapping from the region ID to the size class is recorded in `possible_regions_` (which is an array wrapper in the 32-bit address space case).

Iterating over `SizeClassAllocator64` is simple and efficient. Each size class has just one region. A region consists of multiple chunks.
```cpp
  void ForEachChunk(ForEachChunkCallback callback, void *arg) {
    for (uptr class_id = 1; class_id < kNumClasses; class_id++) {
      RegionInfo *region = GetRegionInfo(class_id);
      uptr chunk_size = ClassIdToSize(class_id);
      uptr region_beg = SpaceBeg() + class_id * kRegionSize;
      uptr region_allocated_user_size = AddressSpaceView::Load(region)->allocated_user;
      for (uptr chunk = region_beg; chunk < region_beg + region_allocated_user_size; chunk += chunk_size)
        callback(chunk, arg);
    }
  }
```

LSan iterates over chunks to find ignored chunks (they are in the root set) and leaked chunks (for error reporting).

Iterating over `SizeClassAllocator32` is efficient with a 32-bit address space (at most `2**32/kSpaceSize = 4096` regions) but inefficient with a 64-bit address space.
See an issue [LSan is almost unusable on AArch64](https://github.com/google/sanitizers/issues/703) (fixed by switching to `SizeClassAllocator64`).

### Stop the world

On Linux, the runtime create a tracer thread to suspend all other threads. More specifically:

* Invoke the `clone` syscall to create a new process which shared the address space with the calling process.
* In the new process, list threads by iterating over `/proc/$pid/task/`.
* In the new process, call `SuspendThread` (ptrace `PTRACE_ATTACH`) to suspend a thread.

`StopTheWorld` returns. The runtime performs mark-and-sweep, reports leaks, and then calls `ResumeAllThreads` (ptrace `PTRACE_DETACH`).

Note: the implementation cannot call libc functions. It does not perform code injection. The toot set includes static/dynamic TLS blocks for each thread.

On Fuchsia, the runtime calls `__sanitizer_memory_snapshot` to stop the world.

## Runtime options

LSan provides some runtime options to toggle behaviors.
Specify the environment variable `LSAN_OPTIONS` to toggle runtime behaviors.

`LSAN_OPTIONS=use_registers=0:use_stacks=0:use_tls=0` can remove some default root regions.
`report_objects=1` reports the addresses of individual leaked objects.

`use_stacks=0`: remove stacks from the root set. The default is 1 to prevent false positives for the following early exit code.
```cpp
int main() {
  std::unique_ptr<XXX> a = ...;
  if (...) exit(0);
}
```

However, the default can cause LSan to detect fewer leaks as there may be dangling pointers on the stacks.
```cpp
struct C { int x = 0; };
int main() { C *a = new C, *b = new C, *c = new C, *d = new C; }
```

Some options (below) are implemented as common options (`compiler-rt/lib/sanitizer_common/sanitizer_flags.inc`).

* Standalone: use `LSAN_OPTIONS`
* AddressSanitizer: use either `LSAN_OPTIONS` or `ASAN_OPTIONS`
* HWAddressSanitizer: use either `LSAN_OPTIONS` or `HWASAN_OPTIONS`

For standalone LSan, `exitcode=23` is the default. The runtime calls an exit syscall with code 23 upon leaks.
For ASan-integrated LSan, `exitcode=1` is the default.

`verbosity=1` prints some logs.

`leak_check_at_exit=0` disables registering an `atexit` hook for leak checking.

`detect_leaks=0` disables all leak checking, including user-requested ones due to `__lsan_do_leak_check` or `__lsan_do_recoverable_leak_check`.
This is similar to defining `extern "C" int __lsan_is_turned_off() { return 1; }` in the program.
Using standalone LSan with `detect_leaks=0` has a performance characteristic similar to using pure `SizeClassAllocator{32,64}` and has nearly no extra overhead except the stack trace.
If we can get rid of the stack unwinding overhead, we will have a simple one benchmarking `SizeClassAllocator{32,64}`'s performance.

If `__lsan_default_options` is defined, the return value will be parsed like `LSAN_OPTIONS`.

## False negatives

There are several reasons that may cause false positives.

First, when LSan scans the root set, it finds aligned bit patterns which look like a pointer to the allocator space.
A integer/floating point number may happen to have a similar bit pattern and tricks LSan.

Dangling pointers may trick LSan. The `use_stacks=0` section above gives an example.
Let's see another example:
```cpp
#include <vector>
std::vector<int *> *a;
int main() {
  a = new std::vector<int *>;
  a->push_back(new int[1]);
  a->push_back(new int[2]);
  a->push_back(new int[3]);
  a->pop_back();
  a->pop_back();
}
```

## Issue suppression

The user can call `__lsan_ignore_object` to ignore an allocation. 
`__lsan::ScopedDisabler` is a RAII class for ignoring all allocations in the scope.
`__lsan_disable` can be called to ignore all allocations for the current thread until `__lsan_enable` is called.

Here are some scenarios that we may want to ignore an allocation.

* a prebuilt library has a known leak
* the program performs a tagged pointer trick and makes a pointer no longer look like a pointer to the allocation. E.g. LSan [added](https://reviews.llvm.org/D133126) built-in support for Objective-C runtime "camouflaging" pointers on macOS

The runtime scans each ignored allocation. An allocation reachable by an ignored allocation is not considered as a leak.

`__lsan_register_root_region` registers a region as a root. The runtime scans the intersection of the region and valid memory mappings (`/proc/self/maps` on Linux).
`LSAN_OPTIONS=use_root_regions=0` can disable the registered regions.

To use the above APIs, include the header `#include <sanitizer/lsan_interface.h>` first.

The runtime parses suppression rules from 3 sources:

* default suppressions (`kStdSuppressions`)
* if `__lsan_default_suppressions` is defined, its return value
* user-specified `LSAN_OPTIONS=suppressions=a.supp`

A suppression file contains a rule per line, each rule being of the form `leak:<pattern>`.
For a leak, the runtime checks every frame in the stack trace.
A frame has a module name associated with the call site address (executable or shared object), and when symbolization is available, a source file name and a function name.
If any module name/source file name/function name matches a pattern (substring matching using glob), the leak is suppressed.

Note: symbolization requires debug information and a symbolizer (internal symbolizer (not built by default) or an `llvm-symbolizer` in a `PATH` directory).

Let's see an example.
```sh
cat > a.c <<'eof'
#include <stdlib.h>
#include <stdio.h>
void *foo();
int main() {
  foo();
  printf("%p\n", malloc(42));
}
eof
cat > b.c <<'eof'
#include <stdlib.h>
void *foo() { return malloc(42); }
eof
clang -fsanitize=leak -fpic -g -shared b.c -o b.so
clang -fsanitize=leak -g a.c ./b.so -o a

which llvm-symbolizer  # available
LSAN_OPTIONS=suppressions=<(printf 'leak:a') ./a      # suppresses both leaks (by module name)
LSAN_OPTIONS=suppressions=<(printf 'leak:a.c') ./a    # suppresses both leaks (by source file name)
LSAN_OPTIONS=suppressions=<(printf 'leak:b.c') ./a    # suppresses the leak in b.so (by source file name)
LSAN_OPTIONS=suppressions=<(printf 'leak:main') ./a   # suppresses both leaks (by function name)
LSAN_OPTIONS=suppressions=<(printf 'leak:foo') ./a    # suppresses the leak in b.so (by function name)
```

## Miscellaneous

Standalone LeakSanitizer can be used with SanitizerCoverage: `clang -fsanitize=leak -fsanitize-coverage=func,trace-pc-guard a.c`

The testsuite has been moved. Use `git log -- compiler-rt/lib/lsan/lit_tests/TestCases/pointer_to_self.cc` (`.cc` instead of `.cpp`) to do archaeology for old tests.

I have a pending FreeBSD port <https://github.com/MaskRay/llvm-project/tree/rt-freebsd-lsan>. Wish that I will find some time to finish it.

## Similar tools

Valgrind's Memcheck defaults to `--leak-check=yes` and performs leak checking.
The feature has been available since the initial revision (`git log -- vg_memory.c`) in 2002.

Google open sourced HeapLeakChecker as part of gperftools. It is part of TCMalloc and used with `debugallocation.cc`.
Multi-threading allocations need to grab a global lock and are much slower than a sanitizer allocator (`compiler-rt/test/lsan/TestCases/high_allocator_contention.cpp`).
`heap_check_max_pointer_offset` (default: 2048) specifies the largest offset it scans an allocation for pointers.
The default makes it vulnerable to false positives.

[heaptrack](https://github.com/KDE/heaptrack) is a heap memory profiler which supports leak checking.
