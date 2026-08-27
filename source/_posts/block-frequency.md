---
layout: post
title: Block frequency
date: 2026-08-23
tags: [algorithm,graph,llvm]
---

[Estimating branch probabilities](/blog/estimating-branch-probabilities) says how one branch splits.
`BlockFrequencyInfo` turns those local numbers into per-block frequencies, which nearly every profitability decision in LLVM ends up reading.

The core is a linear-time propagation over loop-packaged regions.
Where no such structure exists — irreducible control flow — the accuracy goes with it.

<!-- more -->

## What it produces

`getBlockFreq(BB)` returns a `BlockFrequency`, a bare `uint64_t` with no unit.
`finalizeMetrics` rescales each function so that its *hottest* block lands on 2⁵⁴.

```cpp
  const unsigned Slack = 10;
  Scaled64 ScalingFactor = Scaled64(1, MaxBits - Slack) / Max;  // Max: the hottest block
```

As `opt -passes='print<block-freq>'` on a module shows:

```
block-frequency-info: hot
 - entry: float = 1.0,  int = 562949953421312     <- 2^49
 - loop:  float = 32.0, int = 18014398509481984   <- 2^54
 - exit:  float = 1.0,  int = 562949953421312

block-frequency-info: flat
 - entry: float = 1.0,  int = 18014398509481984   <- 2^54
 - a:     float = 0.5,  int = 9007199254740992
 - b:     float = 0.5,  int = 9007199254740992
```

The two printed columns do not share a reference: `float` is the frequency with the *entry* at 1.0, `int` is the same frequency with the *hottest block* at 2⁵⁴.

`MachineBlockFrequencyInfo::getBlockFreqRelativeToEntryBlock` divides by `getEntryFreq()` to recover the entry-relative ratio, which is the whole of what register allocation weights a spill by.
Turning a frequency into a count needs a profile as well:

```
count(BB) = entryCount × freq(BB) / freq(entry)
```

`getProfileCountFromFreq` returns nothing when the function carries no entry count — which is how `isHotBlock` and the rest degrade to "unknown" rather than to "cold".

2⁵⁴ rather than `UINT64_MAX` because consumers add frequencies up and multiply them by instruction costs, and `Slack = 10` leaves room before those saturate.
At the other end every block is floored at 1: the coldest-to-hottest range rarely fits in 64 bits, and the comment says outright that the precision is spent on the hot end.

Until `finalizeMetrics` runs, a frequency is a `Scaled64` — `ScaledNumber<uint64_t>`, a software float with a 64-bit significand and an exponent.
Loop scales compose multiplicatively, so the range runs far past anything a fraction of one could hold; the integers above are the last step, not the working representation.

## Mass, packaging, scale

BFI propagates mass one loop level at a time.
The scheme is the propagation half of Wu and Larus, *Static Branch Frequency and Program Profile Analysis* (MICRO-27, 1994), though nothing in the tree cites it.

Blocks are numbered in reverse post-order, and a `BlockNode`'s index **is** its RPO index — which is why later code can test edge direction with a plain `<`.

```
compute(F):
  for L in loop forest of F, innermost first:
    sweep(L)
    L.scale = 1 / (1 - L.BackedgeMass)
    package L: one pseudo-node reusing its header's RPO number,
               whose successors are the exits sweep(L) recorded
  sweep(F)                                 # every loop is a single node by now
  freq = mass                              # each block's mass within its innermost region
  for L in loop forest of F, outermost first:
    scale = L.scale * freq[L.package]      # final already; read before the writes below
    for b in members(L):
      freq[b] = mass[b] * scale

# members(R) are R's *immediate* members, in RPO order.  A nested loop contributes
# exactly one node, its package, never its blocks.  For the function that means every
# block outside any loop, plus one node per top-level loop.

sweep(R):                                  # R is one loop, or the whole function
  mass[entry of R] = 1
  for b in members(R):
    for (b -> s, w) in succWeights(b):     # a package reports its recorded exits
      m = mass[b] * w / total weight out of b
      if s is R's header:   R.BackedgeMass += m
      elif s is outside R:  R.Exits += (s, m)
      else:                 mass[s] += m
```

`sweep` runs on a DAG and computes `mass[b]`, the probability that a walk from the header passes through `b`.
A backedge deposits into `BackedgeMass` instead of being followed and every sub-loop is already a single node, so RPO is a topological order there and one pass suffices: `O(V+E)`.
GCC's `propagate_freq` instead re-walks each block once per loop depth, `O(V·D)`.

Mass is therefore a fraction of one, and the `1` above is a `BlockMass`: a `uint64_t` in which `BlockMass::getFull()` — `UINT64_MAX` — is that one.
Fixed point, so a split across successors recombines exactly; `distributeMass` dithers the remainder, an even split of full mass giving `8000000000000000` and `7fffffffffffffff`, which add back to `ffffffffffffffff`.
Distribution stays exact; the scales, and the frequencies built from them, are `Scaled64`.

The scale is the geometric series: return 90% of the mass to the header and the loop runs 10 times.

What makes this sound is that a natural loop's header is its *only* way in.
All mass enters there, so each member's mass comes out a fixed multiple of the header's, determined by the branch probabilities alone.
The loop's internal shape and its scale are settled without knowing anything outside it, and `unwrapLoops` supplies the header's real frequency afterwards — one multiplication, applied to a shape that never needed revising.

An irreducible loop has no such invariant.
Mass enters at several blocks at once, and the members' relative frequencies depend on how it splits among those entries — a ratio fixed by the enclosing region, not by the SCC.

## Irreducible loops

A natural loop is a strongly connected component with one way in, its header.
An SCC with two or more entries is *irreducible*.
It is packaged like a loop — the lowest-RPO member stands for the package and donates its RPO number — and `solveIrreducibleMass` settles the members instead.

The routine is up against one missing number: inside the region the frequencies solve

```
f = e + fP
```

with `P` the intra-SCC probabilities and `e` the mass arriving from outside at each entry.
Packaging is bottom-up: the SCC has to collapse to a single node before the enclosing region is swept, because that sweep is only valid on a DAG — but `e` is what that sweep produces.
At solve time it does not exist yet.
A natural loop does not care, its `e` being a scalar on the one header; an irreducible SCC's is a distribution over its entries, and its shape is part of the answer.

So the solve drops `e`.
It starts `F` uniform over the members and iterates `F ← F·P`, converging towards **π**, the dominant left eigenvector of `P`: the shape the SCC settles into after circulating a long time, whichever way the entry mass split.
Members carrying `!irr_loop_header_weight` — PGO metadata recording a measured frequency for an irreducible header — are held fixed instead, and the rest settle around them.

What that gives up shows in the true solution.
Expand it as a series, where term `k` is the mass that came in through the entries and has since gone round the SCC `k` times.
Split `e` along the eigenvectors of `P` — `e = c₁π + c₂e₂ + ⋯`, with `λᵢ` the eigenvalue belonging to `eᵢ` — and that series collapses:

```
f = e(I − P)⁻¹ = e + e·P + e·P² + ⋯
               = c₁·π/(1−λ₁) + c₂·e₂/(1−λ₂) + ⋯
```

λ₁ is the largest eigenvalue, so π gets the largest weight; an SCC that circulates many times before exiting has λ₁ near 1 and is almost entirely that term, which is what makes π a reasonable thing to aim at.
What aiming at it misses is the head of the series: the first term is `e` itself, the mass that leaves without going round at all, and it dominates whenever most mass exits after a lap or two.
So the solve is exact when `e` already points along π and decays as it tilts away — a condition on direction, not on balance, which is why entries of equal frequency are neither necessary nor sufficient.

The error does not stay inside the package either, because `F` also decides how much mass leaves.
Each exit edge contributes `F[i]` times its probability, those contributions sum to the package's exit mass, and the loop scale is its reciprocal.
Weight a block that exits readily where the truth weights one that circulates, and the exit mass comes out too large, so the scale comes out too small and every block in the package is uniformly that much too cold.
Those same contributions are the exits the package hands its parent, so the distortion travels outward too — `yyparse_1` below is that failure at 276×, landing on the enclosing loop.

The iteration is bounded rather than run to convergence:

```cpp
  // A backstop, not a convergence criterion: a periodic SCC never settles.
  const unsigned MaxIterations = 16;
```

A periodic SCC oscillates forever — 10 of 38 in one corpus were still moving at 100000 iterations.
Sweeping the bound leaves the error distribution unchanged from 10 iterations to 1000, and not monotonically — 20 scores better than 30 — so the iterate wanders rather than steadily improving and the tail is noise.
Cost per SCC of `m` internal edges goes `O(m)` → `O(k·m)` with `k ≤ 16`, bounded by the SCC rather than by the function.

The solve also inherits circulation that never becomes a loop scale.
Since [#213488](https://github.com/llvm/llvm-project/pull/213488) the forest BFI wants is a strict subset of the cycle forest: `initializeLoops` does not represent a reducible cycle whose header is an *entry* of an enclosing irreducible cycle, so that equal entries stay equal:

```
  entry -> a | b        a -> c | b        c -> a        b -> a | exit
```

`{a,c}` is a natural loop headed by `a`, and `a` is an entry of `{a,b,c}`, so `-debug-only=block-freq` prints no `- loop =` line for it and its ~32× trip count never arrives as a scale; all of it has to come out of the solve.

All three nestings occur, but only two survive into the packaging.
An irreducible SCC inside a natural loop is the ordinary case and both are represented, the SCC packaged first and given its own scale.
A natural loop inside an irreducible SCC is also packaged first, so it enters the `IrreducibleGraph` as a single node — unless `hasLoop` dropped it, as above.
An irreducible cycle inside an irreducible cycle nests freely in the forest, three deep here:

```
  entry -> a | b        a -> c        b -> d        c -> d | a        d -> c | b
```

but `analyzeIrreducible` takes *maximal* SCCs and a nested irreducible cycle lies inside the same one, so all of it comes out as a single flat package — one power iteration over every member, with none of the inner structure surviving as a scale.

Members are still classified, by `analyzeIrreducible` running Tarjan over an `IrreducibleGraph` — the CFG restricted to the enclosing loop, quotiented by packaged sub-loops, cut at the enclosing header:

```cpp
  // entry: an edge from another SCC reaches it
  if (SccId[U] != SccId[V]) IsEntry.set(V);

  // extra header: an intra-SCC retreating edge targets it
  if (!IsEntry[U] && SccId[V] == SccId[U] && !(U.Node < V->Node))
    Extra.set(V);
```

**Extra headers** are internal cycles that would otherwise wrap back past an already-swept block; the `!IsEntry[U]` guard keeps a retreating edge *from* an entry from creating one, entries having no meaningful relative order.
Since the solve treats all members alike, the split now survives only to decide `isIrrLoopHeader()`, where PGO instrumentation places its counters.

Almost none of these regions are written by anyone.
Tail duplication replicates a computed-goto interpreter dispatch, each copy becomes another entry into the same cycle, and the SCC ends up as wide as the dispatch table: 22 of 221703 functions at MIR against 3 of 64391 at IR.

## Measuring it

For a region with an exit, frequencies are the unique solution of `f = e + fP`, where `P` holds the branch probabilities and `e` is the *entry injection* — the mass arriving from outside the region; for a closed region only ratios are defined, and the answer is the stationary distribution of `P`.
Both are small exact rational solves: scrape `print<branch-prob>` for the probabilities, eliminate over `fractions.Fraction`, compare against `print<block-freq>`.

Two traps in the comparison.
Filter blocks below ~`1e-6` of the hottest — BPI's weight-1 floor manufactures `1e-19` blocks that otherwise dominate every relative error.
And do not renormalize `P`'s rows to stochastic in order to compare ratios when the region has a real exit; that changes the object being solved, and will report an improvement as a regression.

Upstream today, against that oracle, on four closed-form functions from `llvm/test/Analysis/BlockFrequencyInfo/irreducible.ll`:

```
                          exact           BFI
  equalrows   lh:o1:o2    5:3:2           5:3:2
  selfloops   a:b         1.600           1.600
  unequalrows lh:o1       2.667           2.667
  nonentry    a/c/b       1008/976.5/32   519.62/992.0/32.0
```

Three are exact.
`nonentry`'s `a` is 1.94× low, and the residual is the missing `e`.
The tree records the fix under the class's known flaws: partially compute mass in the parent loop and stop at the SCC, which gives the correct ratio of entry masses; compute mass in the SCC, then continue in the parent.

Damping (`F ← (F + F·P)/2`) would fix the oscillation and converges in a median of 32 steps, but it converges *towards* the quasi-stationary vector, which is the wrong target while the injection is missing: it turns three exact cases into 1.24–1.41× to buy one improvement.

## The header split

Until August 2026 there was no solve.
BFI invented a header, as the file's comment said:

> Block frequency calculations act as if a block is inserted that intercepts all the edges to the headers.
> All backedges and entries point to this block.
> Its successors are the headers, which split the frequency evenly.

Entries and extras were sorted *together* into a header prefix of `LoopData::Nodes`, so `isHeader` was a binary search and `getHeaderIndex` a `lower_bound`.
The primary entry was `Nodes[0]`, the lowest-RPO header — and because of that joint sort it might well be an extra header rather than a real entry.

Mass reached the headers in four steps.
Full mass was **seeded** across them by `!irr_loop_header_weight`, or by `MinHeaderWeight` for headers lacking it (the minimum, the comment noted, beats the average), or evenly when no metadata existed at all.
Every node was **swept**, headers included.
Any edge to any header **accumulated** into a per-header `BackedgeMass` vector, indexed by `getHeaderIndex`.
Finally `adjustLoopHeaderMass` **corrected**, but only if no header had metadata: it discarded the seeded split and redistributed full mass in proportion to the backedge mass each header actually received.

That correction was the entire approximation — **one step of power iteration**, seeded from uniform, never repeated, and skipped outright when a profile was present.
`unequalrows` at 4.000 against an exact 2.667 is precisely what one such step from an even split gives.

[#215170](https://github.com/llvm/llvm-project/pull/215170) packaged the SCC with a single representative and solved it instead.
`NumHeaders`, the sorted prefix, `getHeaderIndex`, `distributeIrrLoopHeaderMass`, `adjustLoopHeaderMass` and `MinHeaderWeight` all went; `isHeader` collapsed to `Node == Nodes[0]` and the per-header `BackedgeMass` vector to a scalar, at net −80 lines.

```
                          exact    before     after
  selfloops   a:b         1.600     1.529     1.600
  unequalrows lh:o1       2.667     4.000     2.667
  nonentry    worst           -    31.50×     1.94×
  nonentry_header worst       -     5.63×     5.24×
```

Over a corpus of irreducible functions:

```
          higher is better             |  relative error (lower is better)
          exact  within 1%  within 10% |   p50    p90     max
before       93        111         118 |  0.99   5.27   521.6
after        93        112         129 |  0.77   2.40    46.2
```

On a tail-duplicated computed-goto dispatch — the shape irreducible CFGs actually take in the wild — worst-block error went 1.536× → 1.003×.

## Relaxing the whole function

LLVM carries a second solver for the same system, behind `-use-iterative-bfi-inference` and gated on profile data plus an irreducible loop.
It relaxes `f = e + fP` over the whole function, where the entry mass *is* known — so unlike the SCC solve it has a fixed point to converge to, not merely a direction.

Where both apply, inference wins.
On `yyparse_1` (`Transforms/SampleProfile/profile-correlation-irreducible-loops.ll`), the function whose comment says inference exists to fix it, relative to `b1`:

```
            exact    solve only    + inference
  b2       586.19       2.1256            586
  b3       585.78       1.1267            586
  b4       585.19       1.1256            585
```

Most of that gap is the missing injection arriving as a wrong scale, one level out from where it was made.
The SCC is packaged as `b7**` inside a natural loop headed by `b2`, and the exits recorded for that package — read off `F` — decide how much of the loop's mass escapes instead of returning to `b2`.
`compute-loop-scale` then puts the *loop's* exit mass at 0.4705 of full against a true 1/586.19 = 0.0017, so 276× too much leaves per iteration, and the trip count is its reciprocal: `b2` prints 2.1256 where the truth is 586.19.
An error made inside the package is charged to the region enclosing it.
The shape largely survives — `b3`:`b4` comes out 1.0010 against an exact 1.0010 — and what is off is the scale everything is multiplied by.

The two stack rather than compete, because inference starts from BFI's own output: it is a refinement seeded by `getFloatingBlockFreq`, so a better seed is worth iterations.
Over the corpus above, enabling inference takes the functions within 10% of exact from 118 to 137 before the SCC solve, and from 129 to 144 after.

On accuracy alone, though, inference dominates at any budget.
`-iterative-bfi-max-iterations-per-block=K` buys K sweeps; on 800 generated ~100-block functions with small embedded irreducible cycles, counting those whose worst block lands within 1% of exact:

```
                  solve off   solve on
  no inference           16         25
  K = 1                  30         52
  K = 4                 191        224
  K = 16                545        552
  K = 1000              756        761
```

Down a column, one sweep already beats the solve — inference solves the system where the solve recovers a direction.
Across the columns, the solve is the better seed at every capped budget and a wash by convergence.

Granularity is what keeps them from being redundant.
Inference's smallest purchase is one relaxation pass over the whole reachable function; the solve's is one power iteration over the cycle.
On `sqlite3WhereBegin` at `-O3` — 985 blocks and 1651 edges around one irreducible cycle of 22 blocks and 42 edges — the solve's *entire* budget is 16 × 42 = 672 multiply-adds against 1651 for a single sweep.
Below one sweep there is nothing to buy but the solve — and the default budget does not converge either: that function runs 985001 updates, exactly the 1000-per-block cap, stopping at a discrepancy of 8.8e-4 against a 1e-12 target.

Shape separates them too: `initTransitionProbabilities` routes sinks back to the entry, so the chain it solves is closed, and where a function has *no* exit blocks — the computed-goto dispatch again — inference is a no-op.
That holds even after forcing its `hasProfileData` gate open, so it is the missing exits, not the gate.

Two defects sit in that same routine.
It drops parallel edges rather than summing their probabilities, and discards zero-probability jumps, so the chain it converges to is not the one the CFG describes — enough to move a generated corpus's median error from 4.6× to 16.2× once the generator stresses both.
And `findReachableBlocks` excludes blocks that cannot reach a sink, which is *right* — a reachable closed SCC has unbounded expected visits, and `f = e + fP` has no finite solution there — but those blocks are then assigned `Scaled64::getZero()`, so an ordinary reducible `while (1)`, the hottest block in its function, reports 0 because an irreducible loop exists elsewhere.
Declining an ill-posed question is correct; answering it with zero is not.

Priorities follow from the `yyparse_1` row: getting it exact needs the flow equations solved, not a better eigenvector for one SCC — an argument for the entry-injection work, visiting SCCs in topological order and handing each the mass its entries actually receive, over anything else described here.

## Detecting irreducibility up front

BFI used to discover which regions need this by failing.
Distribution ran until `addToDist` hit a retreating local edge to a non-header, then aborted, packaged, and reran — which is why `IrreducibleGraph::addNode` resets each node's mass as a side effect of building the graph.
CycleInfo already knows, so [#214941](https://github.com/llvm/llvm-project/pull/214941) marks the regions while `initializeLoops` walks the cycle forest and packages them before distributing.

## What not to do

Sourcing the SCC decomposition from CycleInfo rather than running Tarjan over `IrreducibleGraph` looks obvious and is not worth it.

The graph is not the CFG — it is quotiented by packaged sub-loops and cut at the enclosing header, without which the entire loop is one SCC and nothing decomposes.
More decisively, the order is load-bearing: packages are created in reverse-topological order and siblings interact, since an edge into a sibling resolves to its raw block if it is not yet packaged but to its header if it is.
7.8% of irreducible regions contain two or more SCCs, so this is common, not a corner case, and restoring the order needs a topological sort of the SCC DAG — the thing just deleted.
`Extra` headers also still need the intra-SCC RPO scan, which CycleInfo cannot answer.

What CycleInfo *can* answer is "does this region contain an irreducible cycle" — which is exactly the detection above.
