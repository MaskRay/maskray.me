layout: post
title: Modified condition/decision coverage (MC/DC) and compiler implementations
author: MaskRay
tags: [linker]
---

Key metrics for code coverage include:

* function coverage: determines whether each function been executed.
* line coverage (aka statement coverage): determines whether every line has been executed.
* branch coverage: ensures that both the true and false branches of each conditional statement or the condition of each loop statement been evaluated.

Condition coverage offers a more fine-grained evaluation of branch coverage.
It requires that each individual boolean subexpression (condition) within a compound expression be evaluated to both true and false.
For example, in the boolean expression `if (a>0 && f(b) && c==0)`, each of `a>0`, `f(b)`, and `c==0`, condition coverage would require tests that:

<!-- more -->

* Evaluate `a>0` to true and false
* Evaluate `f(b)` to true and false
* Evaluate `c==0` to true and false

A condition combination refers to a specific set of boolean values assigned to individual conditions within a boolean expression.
Multiple condition coverage ensures that all possible condition combinations are tested. In the example above, with three conditions, there would be 2³ = 8 possible condition combinations.

## Modified condition/decision coverage

While multiple condition coverage may not be practical, [Modified condition/decision coverage (MC/DC)](https://en.wikipedia.org/wiki/Modified_condition/decision_coverage) offers a more cost-effective approach.
Introduced in 1992 by [DO-178B](https://en.wikipedia.org/wiki/DO-178B) (later superseded by DO-178C), MC/DC became mandatory for Level A software in the aviation industry.
Its popularity has since extended to safety-critical applications in automotive ([ISO 26262](https://en.wikipedia.org/wiki/ISO_26262)) and other domains.
Notably, SQLite boasts 100% MC/DC coverage link to SQLite testing page: https://sqlite.org/testing.html#mcdc.

Consider a boolean expression like `(A && B) || (C && D)`. This has four conditions (A, B, C, and D), each potentially a subexpression like `x>0`.
Tests evaluate condition combinations (ABCD) and their corresponding outcomes.

Multiple flavors of MC/DC exist, with Unique-Cause MC/DC representing the strongest variant.
When demonstrating the independence of A in the boolean expression `(A && B) || (C && D)`, Unique-Cause MC/DC requires two tests with different outcomes and:

* A is false in one test and true in the other.
* B, C and D values remain identical.

The two tests form an _independence pair_ for A. A _coverage set_ comprises tests offering such independence pairs for each condition.
However, achieving this set may be impossible in the presence of _strongly coupled conditions_.

Coupling examples:

* The two conditions in `x==0 && x!=0` are _strongly coupled_: changing one automatically changes the other.
* `x==0 || x==1 || x==3` exhibits _weakly coupled conditions_: changing x from 0 to 2 alters only the first condition, while changing it to 1 affects the first two.

### Masking MC/DC

_Masking_ involves setting one operand of a boolean operator to a value that renders the other operand's influence on the outcome irrelevant.
Examples:

* Masking the LHS of `&&` with `A && false` (outcome is always false, unaffected by A).
* Masking the LHS of `||` with `A || true` (outcome is always true, unaffected by A).

Due to short-circuit semantics, the RHS of `&&` is not evaluated when the LHS is false. 

_Masking MC/DC_ demonstrates condition independence by showing the condition in question affects the outcome and keeping other conditions masked.
For example, to provde the independence of A in the boolean expression `(A && B) || (C && D)`, C and D can change values as long as `C && D` remains false.
In this way, each condition allows more independence pairs than Unique-Cause MC/DC.

In 2001, masking MC/DC has been considered an acceptable method for meeting objective 5 of Table A-7 in DO-178B.

Unique-Cause + Masking MC/DC is weaker than Unique-Cause MC/DC but stronger than Masking MC/DC, allowing masking only for strongly coupled conditions.

### Minimum coverage set size

If an expression has N unique conditions, both Unique-Cause MC/DC and Unique-Cause Masking MC/DC require a minimum of N+1 tests.
It is not clear whether this is an exact bound.
[When N<=4](https://math.stackexchange.com/questions/1645578/is-it-always-possible-to-get-mc-dc-coverage-on-an-n-input-boolean-function-wit), it is always possible to get Unique-Cause MC/DC with N+1 tests.
Masking MC/DC requires a minimum of `ceil(2*sqrt(N))`. See _An Investigation of Three Forms of the Modified Condition Decision Coverage (MCDC) Criterion_ for detail.

### Binary decision diagram

Binary decision diagram (BDD) is a data structure that is used to represent a boolean function.
Boolean expressions with `&&` and `||` compile to reduced ordered BDDs.

There is another coverage metric called object branch coverage, which determines whether each branch is taken at least once and is also not taken at least once.
Object branch coverage does not guarantee MC/DC, but does when the reduced ordered BDD is a tree.

`(B && C) || A` is a non-tree example that achieving object branch coverage requires 3 tests, which are insufficient to guarantee MC/DC.
If the expression is rewritten to `A || (B && C)`, then the reduced ordered BDD will become a tree, making object branch coverage guarantee MC/DC.

## GCC

Since GCC 3.4, GCC has employed `.gcno` and `.gcda` files to store control-flow graph information and arc execution counts, respectively.
This format has undergone enhancements but remains structurally consistent.
`.gcno` files contain function records describeing basic blocks, arcs between them, and lines within each basic block.
Column information is only available for functions.
`.gcda` files store arc execution counts.

gcov identifies basic blocks on a particular line (usually one) and locates successor basic blocks to infer branches.
When `-b` is specified, gcov prints branch probabilities, though the output may be unclear since `.gcno` does not encode what true and false branches are.

```sh
cat > a.c <<e
int test(int a, int b) {
  if (a > 0 && b > 0)
    return 1;
  return 0;
}

int main() {
  test(0, 1);
  test(1, 0);
  test(1, 1);
}
e
gcc --coverage a.c -o a && ./a
gcov -b a.
```

The output
```
        -:    0:Source:a.c
        -:    0:Graph:a.gcno
        -:    0:Data:a.gcda
        -:    0:Runs:1
function test called 3 returned 100% blocks executed 100%
        3:    1:int test(int a, int b) {
        3:    2:  if (a > 0 && b > 0)
branch  0 taken 67% (fallthrough)
branch  1 taken 33%
branch  2 taken 50% (fallthrough)
branch  3 taken 50%
        1:    3:    return 1;
        2:    4:  return 0;
        -:    5:}
        -:    6:
function main called 1 returned 100% blocks executed 100%
        1:    7:int main() {
        1:    8:  test(0, 1);
call    0 returned 100%
        1:    9:  test(1, 0);
call    0 returned 100%
        1:   10:  test(1, 1);
call    0 returned 100%
        -:   11:}
```

However, there is no direct MC/DC support.
I believe that people just approximate MC/DC with branch coverage.
For side-effect-free expressions like `(B && C) || A`, there might be avenues for compiler transformation into a tree-style BDD, such as `A || (B && C)`.
However, I don't know the presence of such tools.

_Efficient Test Coverage Measurement for MC/DC_ describes a code instrumentation technique to determine masking MC/DC.
For a boolean expression with N conditions, each condition is assigned 2 bits:

* One bit shows that the condition independently affects the outcome while evaluating to false.
* The other bit shows that the condition independently affects the outcome while evaluating to false.

The instrumentation adds a few bitwise instructions that records the branches taken in conditions and applies a filter for masking effects.
When both bits assigned to a condition are 1, we have found an independence pair for this condition.

Jørgen Kvalsvik posted the first MC/DC patch to gcov in March 2022 and [PATCH v9](https://gcc.gnu.org/pipermail/gcc-patches/2023-December/641601.html) in December 2023.
With this patch, we compile source files using `gcc --coverage -fcondition-coverage` and pass `--conditions` to gcov.
The output should look like:
```
        3:   17:void fn (int a, int b, int c, int d) {
        3:   18:    if ((a && (b || c)) && d)
conditions covered 3/8
condition  0 not covered (true false)
condition  1 not covered (true)
condition  2 not covered (true)
condition  3 not covered (true)
        1:   19:        x = 1;
        -:   20:    else
        2:   21:        x = 2;
        3:   22:}
```

## Clang

Clang offers a sophisticated approach to code coverage called [Source-based Code Coverage](https://llvm.org/docs/CoverageMappingFormat.html).
Unlike gcov's line-oriented method, Clang utilizes coverage mapping, a format capable of encoding:

* Nested control structures
* Line/column information
* Macro expansion tracking (`ExpansionRegion`)

In January 2021, the framework has been [enhanced with branch coverage](https://reviews.llvm.org/D84467). This addition:

* Creates a new region for `&&` and `||` operators.
* Tracks execution counts for individual conditions.

The primary data structure changes are the additions of the second counter (`CountedRegion::FalseExecutionCount` and `CounterMappingRegion::FalseCount`) and a new `CounterMappingRegion::RegionKind` named `BranchRegion`.

```c
x = x > 0 && y > 0;  // 2 extra counters

if (x > 0 && y > 0)  // 2 extra counters, while 1 suffices
  x = 1;

if (true && y > 0)   // 2 extra counters, while 1 suffices
  x = 1;
```

When the boolean expression is used in an `if` statement, the `then` counter can be reused by the right operand of the logical operand, but this optimization has not been implemented (mentioned by D84467).

The presentation "_Branch Coverage: Squeezing more out of LLVM Source-based Code Coverage, 2020_" elaborates on the design.

```sh
clang -fprofile-instr-generate -fcoverage-mapping a.c -o a
./a
llvm-profdata merge -sparse default.profraw -o a.profdata
llvm-cov show a -instr-profile=a.profdata -show-branches=count
```

```
    1|      3|int test(int a, int b) {
    2|      3|  if (a > 0 && b > 0)
  ------------------
  |  Branch (2:7): [True: 2, False: 1]
  |  Branch (2:16): [True: 1, False: 1]
  ------------------
    3|      1|    return 1;
    4|      2|  return 0;
    5|      3|}
    6|       |
    7|      1|int main() {
    8|      1|  test(0, 1);
    9|      1|  test(1, 0);
   10|      1|  test(1, 1);
   11|      1|}
```

A Rust feature request was since then filed: https://github.com/rust-lang/rust/issues/79649

In January 2024, Clang's Source-based Code Coverage [got masking MC/DC](https://reviews.llvm.org/D138849) capability.

* `-fcoverage-mcdc` tells Clang to instrument `&&` and `||` expressions to record the condition combinations and outcomes, and store the reduced ordered BDDs into the coverage mapping section. The bitmap is stored in the `__llvm_prf_bits` section in a [raw profile](https://llvm.org/docs/InstrProfileFormat.html).
* `llvm-profdata merge` merges bitmaps from multiple raw profiles and stores the merged bitmap into an indexed profile.
* When passing `--show-mcdc` to `llvm-cov show`, `llvm-cov` reads a profdata file, retrieves the bitmap, computes independence pairs, and print the information.

Clang adopts a distinct approach to Masking MC/DC compared to the paper "_Efficient Test Coverage Measurement for MC/DC_".
Instead of complex masking value computation, it uses a "boring algorithm":

* Encodes boolean expressions with N conditions as integers within `[0, 2**N)`.
* When the expression result is determined, sets a bit in a bitmap indexed by the integer.
* Limits condition count to 6 for space optimization.

For example,
```c
if (a && b || c)
  return 1;
```

Let's say in one execution path `a=c=1` and `b=0`. the condition combination (0b101) leads to an index of 5.
The instrumentation locates the relevant word in the bitmap and set the bit 5.

The approach is described in detail in "_MC/DC: Enabling easy-to-use safety-critical code coverage analysis with LLVM_" in 2022 LLVM Developers' Meeting.

Pros:

* Easier to understand
* Each condition instrumentation adds just one single bitwise OR instruction, instead of possibly three (one bitwise AND plus two bitwise OR).

Cons:

* More bits to encode N conditions (`2**N` vs. `2*N`)
* More metadata to encode the reduced ordered BDDs, required by the reader to compute independence pairs
* Determining independent pairs involves a brute-force algorithm in llvm-cov, which has a high time complexity but probably acceptable due to the limited condition count.

## References

* An Investigation of Three Forms of the Modified Condition Decision Coverage (MCDC) Criterion, John Joseph Chilenski, 2011
* Formalization and Comparison of MCDC and Object Branch Coverage Criteria, 2012
* Efficient Test Coverage Measurement for MC/DC, 2013
* Branch Coverage: Squeezing more out of LLVM Source-based Code Coverage, 2020
* MC/DC: Enabling easy-to-use safety-critical code coverage analysis with LLVM, 2022
