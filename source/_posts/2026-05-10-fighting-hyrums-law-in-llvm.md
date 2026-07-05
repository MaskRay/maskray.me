---
layout: post
title: Fighting Hyrum's Law in LLVM
author: MaskRay
tags: [llvm,clang,lld]
---

> *With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody.* — Hyrum's Law

In a compiler, the most common form of Hyrum's Law is dependence on *unspecified behavior* — hash bucket order, the order of equal elements after `std::sort`, padding offsets.
The same framing covers a few cases that are technically undefined behavior (use of an invalidated iterator) or plain incidental properties (ABI struct layout, ELF section offsets).

When the compiler itself harbors such a dependency, the symptom is usually output that varies build-to-build: an unstable sort that lands differently after the standard library changes, a hash map whose iteration order shifts when the hash function does.
Occasionally the variation is run-to-run within a single build — `DenseMap<void *, X>` keys with an ASLR-derived seed reorder buckets each invocation.
Either way, reproducible builds, bisection, and bug reports all assume same input → same output, and a stealth Hyrum dependency breaks that.

This post surveys some mechanisms that perturb the contract's blind spots so dependencies cannot quietly form.

<!-- more -->

## Hash seed perturbation

The first line of defense is the hash function itself.
`llvm/include/llvm/ADT/Hashing.h`:

```cpp
inline uint64_t get_execution_seed() {
#if LLVM_ENABLE_ABI_BREAKING_CHECKS
  return static_cast<uint64_t>(
      reinterpret_cast<uintptr_t>(&install_fatal_error_handler));
#else
  return 0xff51afd7ed558ccdULL;
#endif
}
```

The seed XORed into every `llvm::hash_value` is the runtime address of `install_fatal_error_handler` — under ASLR, different every process.
The header comment is explicit:

> *the seed is non-deterministic per process (address of a function in LLVMSupport) to prevent having users depend on the particular hash values.*

Every `hash_combine` / `hash_integer_value` call picks up the seed, and every `DenseMap<K, V>` keyed by a `hash_value`-using type then reorders its buckets per run.
MD5, BLAKE3, SHA1, SHA256 stay byte-stable — those are the right tools when you actually want a digest.

My [commit `ce80c80dca45`](https://github.com/llvm/llvm-project/commit/ce80c80dca45) introduced the seed in 2024.

## Container iteration order

Code can grow dependencies on the iteration order.
`LLVM_ENABLE_REVERSE_ITERATION` walks hash containers backwards to flag violations.
`llvm/include/llvm/Support/ReverseIteration.h`:

```cpp
template <class T = void *> constexpr bool shouldReverseIterate() {
#if LLVM_ENABLE_REVERSE_ITERATION
  return detail::IsPointerLike<T>::value;
#else
  return false;
#endif
}
```

`DenseMap` flips its `BucketItTy` to `std::reverse_iterator<pointer>`; `SmallPtrSet` swaps `begin()` and `end()`; `StringMap` bitwise-NOTs the hash before bucket selection — the only thing that perturbs `StringMap`, since its hash bypasses `get_execution_seed`.

Unlike the hash seed, reverse iteration isn't auto-on with assertions; `-DLLVM_REVERSE_ITERATION=ON` opts in explicitly.
In 2026 has already merged fixes triggered by it: [`7f703cabf728`](https://github.com/llvm/llvm-project/commit/7f703cabf728) (MLIR SSA-value completion order), [`0b3afd35c41d`](https://github.com/llvm/llvm-project/commit/0b3afd35c41d) (MLIR SROA alloca order), and [`f5e2c5ddcec7`](https://github.com/llvm/llvm-project/commit/f5e2c5ddcec7) (a clang test).

## Iterator invalidation

Orthogonal to iteration order: what happens to an existing iterator after a mutation.
`llvm/include/llvm/ADT/EpochTracker.h`:

```cpp
class DebugEpochBase {
  uint64_t Epoch = 0;
public:
  void incrementEpoch() { ++Epoch; }
  ~DebugEpochBase() { incrementEpoch(); }  // catches use-after-free

  class HandleBase {
    bool isHandleInSync() const {
      return *EpochAddress == EpochAtCreation;
    }
  };
};
```

`DenseMap` and friends inherit from `DebugEpochBase`.
Mutations bump the epoch; iterators capture it at construction and assert on mismatch.
The destructor bumps too, so stale iterators into destroyed containers assert rather than read freed memory.

Without it, mutate-during-iteration "happens to work" depending on bucket layout — and bucket layout is what the hash seed and reverse iteration above perturb.
The epoch check turns the latent bug into a clean assert regardless of which "lucky" layout the run lands on.
Collapses to a no-op under `NDEBUG`.

## Pre-shuffling unstable sorts

The same defensive pattern shows up twice in the monorepo, in different sub-projects, years apart.

### `llvm::sort` under `EXPENSIVE_CHECKS`

`llvm/include/llvm/ADT/STLExtras.h`:

```cpp
#ifdef EXPENSIVE_CHECKS
namespace detail {
inline unsigned presortShuffleEntropy() {
  static unsigned Result(std::random_device{}());
  return Result;
}

template <class IteratorTy>
inline void presortShuffle(IteratorTy Start, IteratorTy End) {
  std::mt19937 Generator(presortShuffleEntropy());
  llvm::shuffle(Start, End, Generator);
}
} // end namespace detail
#endif

template <typename IteratorTy, typename Compare>
inline void sort(IteratorTy Start, IteratorTy End, Compare Comp) {
#ifdef EXPENSIVE_CHECKS
  detail::presortShuffle<IteratorTy>(Start, End);
#endif
  std::sort(Start, End, Comp);
}
```

`std::sort` and `qsort` are unstable; code observing the order of equal elements is depending on undocumented behavior.
Pre-shuffling makes that observation different every run.
[commit `5a3d47fabcb6`](https://github.com/llvm/llvm-project/commit/5a3d47fabcb6) added the wrapper in 2018, motivated by [PR35135](https://bugs.llvm.org/show_bug.cgi?id=35135).

LLVM also ships its own `llvm::shuffle` rather than calling `std::shuffle`, "so that LLVM behaves the same when using different standard libraries."
A reproducibility tool whose reproducibility depends on the host stdlib is worse than no tool — and the linker section below relies on this.

`llvm::stable_sort` deliberately does not pre-shuffle; it is the explicit opt-in for code that legitimately needs ordering of equal elements.

### libc++ `_LIBCPP_DEBUG_RANDOMIZE_UNSPECIFIED_STABILITY`

libc++ has a near-perfect parallel mechanism, designed for downstream users rather than the project's own internals.
`libcxx/include/__debug_utils/randomize_range.h`:

```cpp
template <class _AlgPolicy, class _Iterator, class _Sentinel>
_LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX14
void __debug_randomize_range(_Iterator __first, _Sentinel __last) {
#ifdef _LIBCPP_DEBUG_RANDOMIZE_UNSPECIFIED_STABILITY
  if (!__libcpp_is_constant_evaluated())
    std::__shuffle<_AlgPolicy>(__first, __last, __libcpp_debug_randomizer());
#else
  (void)__first; (void)__last;
#endif
}
```

Three callsites:

* `std::sort` — pre-shuffles the input.
* `std::partial_sort` — pre-shuffles the input *and* re-shuffles the unsorted tail afterward.
* `std::nth_element` — pre-shuffles, then re-shuffles each side of the partition.

Seed handling rhymes with `get_execution_seed`: ASLR or static `std::random_device` for per-process variation, with `_LIBCPP_RANDOMIZE_UNSPECIFIED_STABILITY_SEED=<n>` as a fixed-seed escape hatch.
Off by default; C++11 and later only.

`libcxx/docs/DesignDocs/UnspecifiedBehaviorRandomization.rst` explains the motivation:

> *Google has measured couple of thousands of tests to be dependent on the stability of sorting and selection algorithms.
> As we also plan on updating (or least, providing under flag more) sorting algorithms, this effort helps doing it gradually and sustainably.*

It cites [PR20837](https://llvm.org/PR20837) — a worst-case `O(n²)` `std::sort` — as the upgrade libc++ specifically wanted to ship.
The shuffle is the gating tool: if downstream tests pass with it enabled, they will pass after the algorithm change too.

Comparing the two is more interesting than either alone:

* `llvm::sort`'s wrapper is internal hygiene: LLVM is its own primary user, so the shuffle lives in `STLExtras.h` behind a build flag with no docs.
* libc++'s wrapper is user-facing — `DesignDocs/` page, public macro, public seed override, explicit "Patches welcome." invitation.
  It has to be: libc++'s users are not libc++, and the contract being defended is the C++ standard itself.
* libc++ generalizes the primitive: `__debug_randomize_range` applies at three callsites, each declaring which sub-range the algorithm leaves unspecified.
  LLVM's wrapper only covers the simpler equal-element case.
* Hashed containers — `std::unordered_*` iteration order — are unspecified in both, but libc++ does not randomize them.
  LLVM-the-library does; on this one surface LLVM is ahead of its own stdlib.

## Linker output: `--shuffle-sections` and `--randomize-section-padding`

Two ELF-only lld knobs perturb layout details that no contract covers.

### `--shuffle-sections=<glob>=<seed>`

`lld/ELF/Writer.cpp`:

```cpp
for (const auto &patAndSeed : ctx.arg.shuffleSections) {
  ...
  const uint32_t seed = patAndSeed.second;
  if (seed == UINT32_MAX) {
    // If --shuffle-sections <section-glob>=-1, reverse the section order.
    // The section order is stable even if the number of sections changes.
    // This is useful to catch issues like static initialization order
    // fiasco reliably.
    std::reverse(matched.begin(), matched.end());
  } else {
    std::mt19937 g(seed ? seed : std::random_device()());
    llvm::shuffle(matched.begin(), matched.end(), g);
  }
}
```

Three regimes in one option:

* `seed = -1` — deterministic reverse, stable even as new sections appear.
  Glob `.init_array*` to `-1`, rebuild, run the test suite: anything that breaks is a real static-init-order bug.
  One flag, no Frankenstein link script.
* `seed > 0` — deterministic random shuffle, reproducible across runs and hosts (because `llvm::shuffle` is host-independent).
  Useful in CI without breaking bisection.
* `seed = 0` — `std::random_device()`-seeded.
  Fresh nondeterminism every link.

History: [`423cb321dfae`](https://github.com/llvm/llvm-project/commit/423cb321dfae) introduced the `=-1` reverse mode; [`16c30c3c23ef`](https://github.com/llvm/llvm-project/commit/16c30c3c23ef) generalized to per-glob seeds, which is what makes the `.init_array*=-1` recipe possible; [`c135a68d426f`](https://github.com/llvm/llvm-project/commit/c135a68d426f) fixed a bug where the feature itself produced an invalid dynamic relocation order — even Hyrum mitigations have correctness traps.

### `--randomize-section-padding=<seed>`

The sister option perturbs section *offsets* by inserting padding between input sections and at segment starts (`lld/ELF/Writer.cpp`):

```cpp
static void randomizeSectionPadding(Ctx &ctx) {
  std::mt19937 g(*ctx.arg.randomizeSectionPadding);
  // Insert padding between input sections and at segment starts.
}
```

Callers grow dependencies on padding-induced offsets the linker never promised — profile-guided pipelines, side-channel research, exploit toolchains pinning to specific addresses.
A seeded perturbation makes those dependencies visible.

Both options are ELF-only; MachO and COFF ports have nothing equivalent.

## ABI break detection

`llvm/include/llvm/Config/abi-breaking.h.cmake`:

```c
#if LLVM_ENABLE_ABI_BREAKING_CHECKS
ABI_BREAKING_EXPORT_ABI extern int EnableABIBreakingChecks;
LLVM_HIDDEN_VISIBILITY
__attribute__((weak)) int *VerifyEnableABIBreakingChecks =
    &EnableABIBreakingChecks;
#else
ABI_BREAKING_EXPORT_ABI extern int DisableABIBreakingChecks;
...
#endif
```

Every TU including the header takes a weak reference to `EnableABIBreakingChecks` or `DisableABIBreakingChecks` depending on its own build flag.
Mixing the two against the same `libLLVM` produces an unresolved symbol at link time.
MSVC gets the same guarantee via `#pragma detect_mismatch`.

Out-of-tree users routinely compile against headers from one tree and link against a different `libLLVM`.
Without this gate, whichever struct layout the link happens to pick silently miscompiles; with it, the link fails.

## What LLVM is *not* doing

The mechanisms above all target surfaces no stable consumer should care about: bucket order, equal-element sort order, init-array order.
Debuggers, profilers, sanitizers, and reproducible-build infrastructure consume those outputs and need them stable.

In some cases, stronger guarantee is only provided with explicit options.
For example, Bitcode and textual IR preserve use-list order only under `-preserve-bc-uselistorder` / `-preserve-ll-uselistorder`.

A near-cousin: clang's `-frandomize-layout-seed` / `__attribute__((randomize_layout))`.
Mechanically the same — seeded `std::shuffle` on struct fields — and it does coincidentally invalidate `offsetof` dependencies.
But the intent is exploit mitigation, cribbed from GrSecurity's Randstruct GCC plugin: per-build kernel hardening, not a developer tool.
