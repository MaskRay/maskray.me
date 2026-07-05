---
layout: post
title: Recent LLVM hash table improvements
author: MaskRay
tags: [llvm,data-structures,performance]
---

LLVM has several hash tables. They used quadratic probing with in-band sentinel keys (empty, tombstone); recent work has been replacing that with linear probing with tombstone key removed.

- `DenseMap` (replacement for `std::unordered_map`): `DenseMapInfo::getEmptyKey()` / `getTombstoneKey()`.
  + `DenseSet`: implemented using `DenseMap`
  + `compiler-rt/lib/sanitizer_common/sanitizer_dense_map.h` ports the implementation for sanitizers.
- `SmallPtrSet` (replacement for `std::unordered_set<T *>`): hard-coded `-1` (empty) and `-2` (tombstone).
- `StringMap` (replacement for `std::unordered_map<std::string, V>`)
  + `StringSet`: implemented using `StringMap`

For the open-addressed `DenseMap` and `SmallPtrSet`, pointers, references, and iterators are invalidated by insert.
`StringMap` is different: each entry lives in a heap-allocated `StringMapEntry<V>` node, so entry pointers survive grow.
`std::unordered_map`, being node-based, keeps surviving-element pointers valid across both insert and erase and only invalidates the erased element's own iterator.
LLVM code rarely needs that stronger contract — callers do not hold long-lived references into the container across mutation — and that gap is what gives pass to relocating erase and bit-array occupancy.

Recently,

- Tombstones have been removed from DenseMap and SmallPtrSet. `erase()` also invalidates pointers.
- DenseMap has also retired its empty-key sentinel, leading to significant performance improvements.
  DenseMap with integer keys (`int`/`unsigned`/`size_t`) had `-1`/`-2` reserved — a footgun, now fixed.
- StringMap got Algorithm R deletion too. Its entries are separately heap-allocated, so erase keeps entry pointers valid but invalidates iterators; erase-while-iterating moved to `remove_if`.

<!-- more -->

## SmallPtrSet

SmallPtrSet has a small mode, used when the number of elements is below a threshold, and a large mode.
The tombstone state was removed by [#96762](https://github.com/llvm/llvm-project/pull/96762) in 2024.
The patch changed `erase()` operation to invalidate iterators, requiring treewide fixes.

The large mode used a quadratic probing with two sentinel keys (empty, tombstone).

Knuth TAOCP Vol. 3 §6.4 Algorithm R describes an algorithm that avoids lazy deletion.
[#197637](https://github.com/llvm/llvm-project/pull/197637) has implemented it.

I've investigated Robin Hood Hashing and Abseil Swiss Table family implementations - both would lead to inferior performance.
Robin Hood Hashing primarily improves find-miss on high-load factors but imposes a small cost on find-hit and inserts.

## DenseMap

Two major changes have been merged:

- Replace the tombstone state with Algorithm R deletion. ([#200595](https://github.com/llvm/llvm-project/pull/200595))
- Replace the empty state with a bitarray. ([#201281](https://github.com/llvm/llvm-project/pull/201281))

### What the workload looks like

An instrumented clang counts every `(KeyT, ValueT)` operation while compiling `llvm/lib/Analysis/ScalarEvolution.cpp`. **597** distinct `DenseMap`/`DenseSet` types, **~186M** user ops:

| op | count | probes mean |
| --- | ---: | ---: |
| find-hit | 65.2M | 1.55 |
| find-miss | 65.7M | 1.28 |
| insert | 47.8M | 1.71 |
| erase | 7.0M | — |
| grow | 1.5M | — |

Lookups are ~70% of the load; insert ~26%, erase ~4%. Probe means sit between 1.3 and 1.7 — well inside what linear probing handles. `operator[]` is classified at the bucket it lands on: a present key is counted as find-hit, an empty slot as insert.

A few instantiations carry most of the traffic:

| type | find-hit | find-miss (mean) | insert | erase | grow | peak |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `DenseMap<const void *, Pass *>` | 680K | 24.8M (1.03) | 202K | 202K | 9.3K | 1 KiB |
| `DenseMap<Value *, ValueHandleBase *>` | 2.8M (2.02) | 0 | 2.2M | 2.2M | 11 | 1 MiB |
| `DenseMap<const Value *, StringMapEntry<Value *> *>` | 3.0M | 1.4M (2.26) | 540K | 439K | 13 | 4 MiB |
| `DenseMap<DeclarationName, StoredDeclsList>` | 772K | 1.2M (1.94) | 143K | 0 | 9.0K | 64 KiB |
| `DenseMap<LazyCallGraph::Node *, LazyCallGraph::SCC *>` | 1.3M | 98K (2.95) | 175K | 173K | 15 | 258 KiB |
| `DenseMap<AnalysisKey *, bool>` | 2.8M | 2.0M (1.52) | 2.0M | 0 | 124K | 1 KiB |

`DenseMap<const void *, Pass *>` runs 25.5M lookups at 1.03 mean — volume is not clustering. `DenseMap<Value *, ValueHandleBase *>` is the single largest erase consumer at 2.2M; this is the workload where the Algorithm R relocating-erase callback earns its keep. Its find-miss column is zero because every access in `llvm/lib/IR/Value.cpp` goes through `operator[]` or `erase` — there is no `find`/`lookup` call site, so an empty-slot probe is bucketed as insert rather than find-miss. The bucket's value-slot address is captured by `ValueHandleBase *&Entry = Handles[V]` and stored as a linked-list back-pointer (`PrevPtr`), which is exactly what the new `OnMoved` callback refreshes when Algorithm R shifts neighbors. `StringMapEntry` peaks at 4 MiB, the regime where the used-bit array's byte overhead matters most. Structural and graph-pointer keys (`DeclarationName`, `LazyCallGraph::Node *`) still cost a couple of extra probes per miss.

Code size matters. SIMD tables may be a poor fit.

### Linear probing + Algorithm R deletion ([#200595](https://github.com/llvm/llvm-project/pull/200595))

Linear probing needs a strong pointer hash.
The old `DenseMapInfo<T*>::getHashValue` (`(p>>4)^(p>>9)`) never mixes the high bits of the address.
Pointers from one allocator grown across multiple slabs map to the same narrow bucket range, bad under linear probing.
Quadratic probing had masked this issue.
[#197390](https://github.com/llvm/llvm-project/pull/197390) switched to a stronger mixer, unblocking both SmallPtrSet and DenseMap.
[#199369](https://github.com/llvm/llvm-project/pull/199369) tightened DenseMap's `erase()` to invalidate iterators under `LLVM_ENABLE_ABI_BREAKING_CHECKS`, surfacing stale-iterator call sites before Algorithm R could crash on them.

#200595 replaces quadratic probing plus tombstones with linear probing plus Algorithm R.
Two bucket states, no lazy markers.
Algorithm R requires each entry to sit on the contiguous probe chain from its home bucket; that's linear probing, not quadratic — the probe scheme had to change.
Erase walks forward from the hole and pulls back any entry whose home bucket lies behind the hole, so entry pointers may be invalidated:

```c++
// Erase buckets[i]; then close the gap.
unsigned j = i;
while (true) {
  j = (j + 1) & mask;
  if (!used(j)) break;
  unsigned home = hash(buckets[j].key) & mask;
  // The hole at i lies on j's probe chain from home → shift j into i.
  if (((i - home) & mask) < ((j - home) & mask)) {
    buckets[i] = std::move(buckets[j]);
    i = j;
  }
}
unsetUsed(i);
```

The new `erase(Key, OnMoved)` and `erase(iterator, OnMoved)` overloads fire a callback once per shifted bucket; `ValueHandleBase::RemoveFromUseList` uses this to refresh `PrevPtr`.

The first attempt had a war story.
The original landed as [#199615](https://github.com/llvm/llvm-project/pull/199615), then got reverted by [#200421](https://github.com/llvm/llvm-project/pull/200421) after a SCEV crash: `PoisoningVH` cached a bare bucket pointer across an op that, post-Algorithm R, relocates.
The bug was latent under tombstones, which don't relocate, and only surfaced once the algorithm flipped.
Fixed in [#200540](https://github.com/llvm/llvm-project/pull/200540), relanded as #200595.

On stage1-O3 the change is **-1.34%** instructions and all ten CTMark benchmarks improve between -0.85% and -1.61% ([compare](https://llvm-compile-time-tracker.com/compare.php?from=90779840d51aa3fff6fa030ade7150bc647ac5ff&to=1f10f1ca8af3dff956b353d7ff3ea169c82ed909&stat=instructions:u)).
Clang wall is **-1.54%**, stage1 binary **-1.40%** — fewer bucket states means less generated code.
The commit message remarks that *"the in-band sentinel value approach … is the best, or at least very difficult to beat."*
The used-bit array below earns the right to violate this.

### Packed used-bit array ([#201281](https://github.com/llvm/llvm-project/pull/201281))

The empty-slot test has switched from `getEmptyKey()` to a bit test.
There is a 1-bit-per-bucket `uint32` array sharing one allocation with the buckets.

This is a trade-off

* Find-miss, iteration, and large-bucket inserts win: the bit array terminates the probe walk and skips the empty buckets without ever loading the bucket key.
* Find-hit and small-bucket inserts lose: a hit loads the matched bucket anyway, and a small-bucket insert pays one extra word write.

The aggregate numbers are interesting ([compare](https://llvm-compile-time-tracker.com/compare.php?from=d7a23b7ab0afad10d26f0f1a5edfce88de452f81&to=1e3dc606df3aefc924fb58238dd705b8bfde1581&stat=instructions:u)).
Stage1-O3 instructions are **-0.99%**, but stage2-O3 is **+0.13%** and stage2-O0-g is **+0.34%** — instructions actually go *up*.
Clang wall time is **-1.37%** while instructions are only **-0.49%**, and `size-file` is **+0.87%** because the bit array costs bytes.
`instructions:u` is the wrong cost model when the savings are cache loads: the counter sees the new bit-test sequence; it does not see the bucket load you didn't issue.

After this patch, `DenseMap<int/unsigned/size_t, X>` accepts every value of the key domain.

### Tree-wide simplification

- `getTombstoneKey()` went out as [ADT #200959](https://github.com/llvm/llvm-project/pull/200959), [IR+Analysis #200958](https://github.com/llvm/llvm-project/pull/200958), [CodeGen+Transforms #200956](https://github.com/llvm/llvm-project/pull/200956), [llvm-rest #200957](https://github.com/llvm/llvm-project/pull/200957), [Target #200955](https://github.com/llvm/llvm-project/pull/200955), and [clang #200634](https://github.com/llvm/llvm-project/pull/200634).
- `getEmptyKey()` removal (post-#201281) followed in [BOLT #201986](https://github.com/llvm/llvm-project/pull/201986), [clang #201987](https://github.com/llvm/llvm-project/pull/201987), [flang #201988](https://github.com/llvm/llvm-project/pull/201988), [lld #201989](https://github.com/llvm/llvm-project/pull/201989), [lldb #201990](https://github.com/llvm/llvm-project/pull/201990), [mlir #201991](https://github.com/llvm/llvm-project/pull/201991), [Polly #201992](https://github.com/llvm/llvm-project/pull/201992), [Target #201993](https://github.com/llvm/llvm-project/pull/201993), [CodeGen+Transforms #201994](https://github.com/llvm/llvm-project/pull/201994), [llvm #201996](https://github.com/llvm/llvm-project/pull/201996), [IR+Analysis #201997](https://github.com/llvm/llvm-project/pull/201997), and [ADT #201998](https://github.com/llvm/llvm-project/pull/201998).

Several of these commits showed small but consistent wins on the tracker.
The IR+Analysis cleanups in particular — [#200958](https://llvm-compile-time-tracker.com/compare.php?from=414b8b986705d24e415e8ef17bc8ddbbd6839712&to=e0f5ce18c04a5c55e5eb4e35e5ab1f1980e15c31&stat=instructions:u) for `getTombstoneKey()` and [#201997](https://llvm-compile-time-tracker.com/compare.php?from=c384de1c688bab5d5d1fbf1bb3d31895c98e34cd&to=3232b4d0e07802a1852874910815489dc3e4774a&stat=instructions:u) for `getEmptyKey()` — each measured around -0.05% to -0.10% stage1-O3 instructions and -0.23% clang wall time.
But the headline win is in the trait itself, not the timings: integer keys are now safe, the MLIR #54908 crash class is gone, and the AssertingVH downcast UB becomes moot.
The design discussion lives in [#200183](https://github.com/llvm/llvm-project/issues/200183).

## StringMap

`StringMap` keys are arbitrary byte strings, so each entry is a heap-allocated `StringMapEntry<V>` node; a bucket is just a pointer, paired with a 32-bit hash in a parallel array that rejects mismatches before the byte-wise key compare.

[#202103](https://github.com/llvm/llvm-project/pull/202103) drops the tombstone and switches to linear probing with Algorithm R; xxh3 is strong, so there was no weak-mixer hazard to fix first.
The blast radius is milder than `DenseMap`: erase relocates only the bucket pointers, not the nodes, so entry pointers and references survive — only iterators are invalidated.

That invalidation broke the erase-while-iterating idiom tombstones had supported by accident.
[#202237](https://github.com/llvm/llvm-project/pull/202237) and [#202520](https://github.com/llvm/llvm-project/pull/202520) add a `DebugEpochBase` so a stale iterator asserts under `LLVM_ENABLE_ABI_BREAKING_CHECKS`, and [#202272](https://github.com/llvm/llvm-project/pull/202272) adds `remove_if` (one O(N) pass, like `SmallPtrSet`) for the migrated call sites.

No used-bit array: `StringMap` lacks the in-bucket key load that justified it on `DenseMap` — occupancy is a pointer-null test and the hash array already serves as the reject filter.
A measured port helped find-miss but left find-hit flat, regressed iteration, and added a third array with no aggregate win.

## Rejected alternatives

Prototyped, measured, rejected on `SmallPtrSet` and `DenseMap`.

- **Verstable** (metadata + home-rooted chains). Wins negative-probe and iteration; loses `clang` self-build — metadata + chain logic inlines into every call site for **+4–10% binary**. Combined allocation, NOINLINE cold paths, fragment removal all tried; ~+3% instructions / ~+5% size at best.
- **Robin Hood**. The find-miss early-out depends on knowing each resident's displacement; without stored metadata it requires re-hashing every resident on the probe walk, which costs more than the saved probes. Storing the displacement recovers the early-out but adds a metadata cache line, and the swap-carry on insert is intrinsic — **+10–20% insert vs Algorithm R** regardless of layout.
- **`boost::unordered_flat_map`**. Bad at pointer keys and code size.

## DenseMap variant optimized for pointer key?

I considered whether DenseMap for pointer keys should keep an in-band sentinel variant ([#200183](https://github.com/llvm/llvm-project/issues/200183)) — it would avoid the used-bit array's max-rss overhead.
Measurements on AMD Zen 4 and Apple M4 ruled it out: the used-bit version is faster despite the extra memory.

## Takeaways

- Linear probing + a good mixer beats quadratic + tombstones on a pointer-heavy workload. [#197390](https://github.com/llvm/llvm-project/pull/197390) fixed the weak pointer hash.
- Swiss Table family implementations are poor at small keys. If deletion performance does not matter, you don't need its heavy implementation.
- `SmallPtrSet` was the dry run that de-risked `DenseMap`: same algorithm, smaller blast radius, same conclusion on rejected alternatives.
- Tombstones aren't free at 4% erase — they slow down insertion.

## Aggregate compile-time impact

All numbers below compare one squashed commit against its parent, the pre-series baseline `5d5220c5` (the commit before `[SmallPtrSet] Drop tombstones in large mode`, #197637).
The squash bundles the whole series so the tracker sees the cumulative effect as a single from→to:

- SmallPtrSet/DenseMap: #197637, #198982, #199369, #200540, #200595, #201281, #201742
- IR/Analysis `getTombstoneKey`/`getEmptyKey` cleanups: #200958, #201997
- StringMap: #202103, #202237, #202272, #202520

Aggregate measurements from [llvm-compile-time-tracker.com](https://llvm-compile-time-tracker.com/):

Significant (≥3σ vs. measured noise): 🟢 improvement · 🔴 regression. Unmarked = within noise.

| Configuration | instructions:u | max-rss |
|---|---:|---:|
| stage1-O3 | 🟢 **-0.38%** | +0.06% |
| stage1-ReleaseThinLTO | 🟢 **-0.40%** | -0.19% |
| stage1-ReleaseLTO-g | 🟢 **-0.43%** | -0.01% |
| stage1-O0-g | 🟢 **-0.19%** | -0.16% |
| stage1-aarch64-O3 | 🟢 **-0.35%** | +0.01% |
| stage1-aarch64-O0-g | 🟢 **-0.18%** | -0.20% |
| stage2-O3 | 🟢 **-0.42%** | -0.13% |
| stage2-O0-g | 🟢 **-0.18%** | -0.03% |

Clang build:

| Metric | Old | New | Δ |
|---|---:|---:|---:|
| instructions:u | 20882372M | 20784665M | 🟢 **-0.47%** |
| wall-time | 350.61s | 352.30s | +0.48% |
| size-file | 137834KiB | 137726KiB | 🟢 **-0.08%** |
| size-file (stage1) | 154243KiB | 154177KiB | 🟢 **-0.04%** |

Build wall-time is a single-shot measurement and noisier than the tracker's per-config medians; instructions:u and size-file are the cleaner signals here.

To localize the win, compile the preprocessed sqlite3 amalgamation under callgrind and sum self-instructions (`Ir`) per source file; a clang built with `-g1` lets the line tables map inlined header code back to its header.
Profile the `cc1` invocation directly so the driver's fork/exec doesn't hide the work:

```sh
# clang built with -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS=-g1 -DLLVM_ENABLE_ASSERTIONS=off
cc1=$(clang -### -O1 -w -c sqlite3.i -o /dev/null 2>&1 | grep -- -cc1)
eval valgrind --tool=callgrind --callgrind-out-file=cg.out --dump-line=yes $cc1
# sum the Ir column grouped by `fl=` source file
# (positions: instr line ⇒ Ir is the 3rd field; skip the cost line after each calls=)
```

Baseline `5d5220c5` vs the squashed series (`whole compile` matches callgrind's own `summary:` line, 22.59B → 22.25B):

| File | Before | After | Δ |
|---|---:|---:|---:|
| `DenseMap.h` | 1537M | 1462M | -4.8% |
| `DenseMapInfo.h` | 320M | 238M | -25.7% |
| `SmallPtrSet.{h,cpp}` | 611M | 554M | -9.3% |
| `StringMap.{h,cpp}` | 19.9M | 17.5M | -12.1% |
| four containers | 2488M | 2272M | -8.7% |
| whole compile | 22590M | 22247M | -1.5% |
