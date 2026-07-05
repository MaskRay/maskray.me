---
layout: post
title: Optimizing LLVM's bump allocator
author: MaskRay
tags: [llvm,codegen,performance]
---

`BumpPtrAllocator` is LLVM's bump allocator (arena allocator): each allocation bumps a pointer within a slab, and everything is freed at once when the allocator dies.
It backs Clang's `ASTContext`, lld's `make<T>` object pools, TableGen records, and many other arenas.

Here is the fast path before three recent changes:

```cpp
__attribute__((returns_nonnull)) void *Allocate(size_t Size, Align Alignment) {
  BytesAllocated += Size;                               // (3) accounting RMW
  uintptr_t AlignedPtr = alignAddr(CurPtr, Alignment);  // (1) always realign
  size_t SizeToAllocate = Size;
#if LLVM_ADDRESS_SANITIZER_BUILD
  SizeToAllocate += RedZoneSize;
#endif
  uintptr_t AllocEndPtr = AlignedPtr + SizeToAllocate;
  if (LLVM_LIKELY(AllocEndPtr <= uintptr_t(End)
                  && CurPtr != nullptr)) {              // (2) bound + null check
    CurPtr = reinterpret_cast<char *>(AllocEndPtr);
    ...
    return reinterpret_cast<char *>(AlignedPtr);
  }
  return AllocateSlow(Size, SizeToAllocate, Alignment);
}
```

<!-- more -->

Three changes streamline the three marked lines.

## A minimum alignment skips the realign ([#205240](https://github.com/llvm/llvm-project/pull/205240))

`alignAddr(CurPtr, Alignment)` is wasteful: a freshly-bumped pointer is usually aligned enough already.
[#205240](https://github.com/llvm/llvm-project/pull/205240) rounds each size up to `MinAlign` (default 8), so the fast path realigns only for over-aligned requests.
I've learned the trick from [Bump Allocation: Up or Down?](https://coredumped.dev/2024/03/25/bump-allocation-up-or-down/):

```cpp
// Optimized to a constant
SizeToAllocate = alignToPowerOf2(SizeToAllocate, MinAlign);

uintptr_t AlignedPtr = uintptr_t(CurPtr);
// For the common `alignof(T) <= 8` case the branch drops out entirely.
if (Alignment.value() > MinAlign)
  AlignedPtr = alignAddr(CurPtr, Alignment);
```

I made a mistake in the first attempt: `nullptr` plus a non-zero offset triggered a UBSan diagnostic.
Fixed by keeping the math in the `uintptr_t` domain.

`SpecificBumpPtrAllocator<T>` is the typed variant: it owns its objects and runs `~T()` on each in `DestroyAll`, which walks the slabs at a fixed `sizeof(T)` stride.

`T` may be **incomplete** where the allocator is declared — e.g. a class member `SpecificBumpPtrAllocator<MCSectionELF>` sitting next to only a forward declaration of `MCSectionELF`.
`alignof(T)` would force `T` complete there; a literal does not.

```cpp
using BumpPtrAllocatorTy =
    BumpPtrAllocatorImpl<MallocAllocator, 4096, 4096, 128, /*MinAlign=*/1>;

T *Allocate(size_t num = 1) {
  if constexpr (alignof(T) <= alignof(std::max_align_t))
    return static_cast<T *>(Allocator.Allocate(num * sizeof(T), Align()));
  return Allocator.Allocate<T>(num);
}
```

Slabs are `max_align_t`-aligned, and this allocator only ever hands out `num * sizeof(T)` bytes — a multiple of `alignof(T)` — so the bump pointer is *already* `alignof(T)`-aligned; requesting `Align()` (i.e. 1) lets the fast path skip the realign branch.
The body is only instantiated when `Allocate` is actually called, by which point `T` is complete, so the `alignof(T)` here is fine even though the class declaration tolerated an incomplete `T`.
Over-aligned types (`alignof(T) > alignof(std::max_align_t)`) take the general path, which realigns.

## A sentinel End drops the null check ([#205485](https://github.com/llvm/llvm-project/pull/205485))

`__attribute__((returns_nonnull))` specifies the return value is non-null.
In a fresh allocator whose `CurPtr` and `End` are both null, `Allocate(0)` used to return null.
In 2022, <https://reviews.llvm.org/D125040> added the `&& CurPtr != nullptr` check to the fast path condition, which was not ideal.

I tried
```cpp
// Fast path check. The condition also fails for a fresh allocator (End ==
// nullptr) to avoid a separate null check.
if (LLVM_LIKELY(AlignedPtr + SizeToAllocate - 1 < uintptr_t(End))) { ... }
```

but then adopted aengelke's suggestion.
Storing the end as a sentinel one past the real end (`EndSentinel = realEnd + 1`, and `0` when there is no slab) folds both conditions into one unsigned compare:

```cpp
if (LLVM_LIKELY(AllocEndPtr < EndSentinel)) { ... }
```

An empty allocator has `EndSentinel == 0`, so `AllocEndPtr < 0` is always false and the null case falls through to the slow path with no separate branch.

## Dropping the per-allocation accounting ([#205711](https://github.com/llvm/llvm-project/pull/205711))

`BytesAllocated += Size` was a read-modify-write to a member on every allocation, backing a `getBytesAllocated()` that reported *requested* bytes — distinct from `getTotalMemory()`'s slab capacity.
It had only stats/diagnostic consumers: lldb's ConstString memory report, a clangd debug log, TableGen's `dumpAllocationStats`, and one clang regression test.
Dropping the member and migrating those consumers (mostly to `getTotalMemory()`) removes the hot-path store.

**A detail: the red zone and ABI.**
The ASan red-zone size is also a member.
Gating it on `#if LLVM_ADDRESS_SANITIZER_BUILD` to drop it in release builds would be an ABI footgun: that macro is *per translation unit*, so an ASan-instrumented TU and a non-ASan `libLLVM` would silently disagree on the struct layout.
The member is instead gated on `LLVM_ENABLE_ABI_BREAKING_CHECKS`, which is fixed per library build and link-time-enforced (via the `EnableABIBreakingChecks` symbol); the red-zone arithmetic is then gated on both macros.

Combined, the fast path becomes:

```cpp
void *Allocate(size_t Size, Align Alignment) {
  size_t SizeToAllocate = Size;
#if LLVM_ADDRESS_SANITIZER_BUILD && LLVM_ENABLE_ABI_BREAKING_CHECKS
  SizeToAllocate += RedZoneSize;
#endif
  SizeToAllocate = alignToPowerOf2(SizeToAllocate, MinAlign);
  uintptr_t AlignedPtr = uintptr_t(CurPtr);
  if (Alignment.value() > MinAlign)
    AlignedPtr = alignAddr(CurPtr, Alignment);
  uintptr_t AllocEndPtr = AlignedPtr + SizeToAllocate;
  if (LLVM_LIKELY(AllocEndPtr < EndSentinel)) {
    CurPtr = reinterpret_cast<char *>(AllocEndPtr);
    ...
    return reinterpret_cast<char *>(AlignedPtr);
  }
  return AllocateSlow(Size, SizeToAllocate, Alignment);
}
```

## Generated assembly

Allocating a typical arena object — a 24-byte, 8-aligned node via `Allocate<T>()` — compiles to a six-instruction fast path (`clang -O2`, release):

```asm
mov  rax, [rdi]        # CurPtr (also the return value)
lea  rcx, [rax + 0x18] # new = CurPtr + 24
cmp  rcx, [rdi + 0x8]  # vs EndSentinel
jae  .slow
mov  [rdi], rcx        # CurPtr = new
ret
```

That matches the canonical bump fast path.
A *downward*-bumping allocator would not need the `rax`/`rcx` distinction — one fewer live value, but the instruction count stays the same.
LLVM bumps upward by design: `identifyObject`, allocation order, and `SpecificBumpPtrAllocator::DestroyAll`'s forward `sizeof(T)` stride all assume it.
The remaining gap is space, not instructions.

## Aggregate compile-time impact

These changes shrink `Allocate` below the inliner's cost threshold, so its callers (e.g. `new (Context) T`) inline at sites that previously called out of line.
Executed instructions fall — but as a *redistribution*: object files where the chain now inlines grow, while the rest shrink slightly from the dropped store.

The performance win is larger at stage2 (built by stage1 Clang) than at stage1 (built by system GCC).

Reverting all three on top of `main` isolates their combined effect ([compare](https://llvm-compile-time-tracker.com/compare.php?from=dbd070fbd793c8a9129044abd669466e87d2ea8e&to=3a7d64a882421052101899d7d9c23685db5fd355&stat=instructions:u)):

Significant (≥3σ vs. measured noise): 🟢 improvement. Unmarked = within noise.

| Configuration | instructions:u | max-rss |
|---|---:|---:|
| stage1-O3 | −0.04% | +0.04% |
| stage1-ReleaseThinLTO | −0.04% | −0.01% |
| stage1-ReleaseLTO-g | −0.04% | +0.06% |
| stage1-O0-g | 🟢 −0.09% | +0.25% |
| stage1-aarch64-O3 | −0.04% | +0.04% |
| stage1-aarch64-O0-g | 🟢 −0.12% | −0.01% |
| stage2-O3 | 🟢 −0.14% | −0.15% |
| stage2-O0-g | 🟢 −0.36% | −0.06% |

## Takeaways

- A bump allocator's fast path is a few instructions of real work wrapped in a realign and accounting; each can be hoisted out of the common case.
- Encoding "empty" as a `0` sentinel folds a null check into the bound compare.
- The measurable instruction-count win is the inlining a cheaper `Allocate` unlocks, not the removed micro-op — and it appears as a size *redistribution*, not a uniform shrink.
- A layout-affecting member may key on `LLVM_ENABLE_ABI_BREAKING_CHECKS` (link-enforced) but never on the per-TU `LLVM_ADDRESS_SANITIZER_BUILD`.
