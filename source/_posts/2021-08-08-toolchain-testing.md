layout: post
title: Toolchain testing
author: MaskRay
tags: [binutils,llvm]
---

Updated in 2025-06.

Some random notes about toolchain testing.

## Classification

Testing levels.

<!-- more -->

### Automated testing

Often called unit testing and integration testing.
As <https://matklad.github.io//2022/07/04/unit-and-integration-tests.html> mentioned, the distinction is confusing.
We can use purity and extent to classify the tests.

In llvm-project, `unittest/` directories (e.g. `llvm/unittest/` and `clang/unittest/`) have some examples of lightweight tests, which are traditionally categorized as unit tests.
They test an individual function, class, or module. Here are some examples:

* A homebrew container has interfaces similar to a STD container.
* The target triple parser can handle a triplet.
* An IR mutator performs the expected operation.
* The source file formatter can handle some JavaScript syntax.
* A code completer can suggest some functions, variables, and types.
* A semantic analysis function can constant evaluate an expression.

Some tests may have a high setup cost for the environment.

The traditional "integration testing" refers to testing multiple modules as a group.

With good modularity, many modules have intuitive and expected behaviors.
Many integration tests may look similar to unit tests.

In llvm-project, many lit tests belong to this category. They may test:

* The Clang driver can detect a GCC installation.
* A Clang driver option passes an option to cc1.
* A cc1 option can cause Clang's IR generator to emit something.
* The optimization and code generation pipelines contain expected passes.
* An instrumentation pass correctly instruments a particular function.
* An optimization pass performs expected transformation.
* The code generator can translate an LLVM IR instruction.
* The code generator can optimize a machine instruction in a certain way.
* The integrated assembler can parse an instruction and encode it correctly.
* A linker option performs the expected operation.
* llvm-readelf can dump a section in an object file.
* A compiled program has the expected behavior at run-time.
* A debugger command performs the expected operation.

### System testing

Test the entire system.
I consider that runtime tests belong to this category because they exercise many modules of the system.

* An environment variable can change a sanitizer behavior.
* A C++ source file can be compiled with debug information coverage above a certain threshold.
* A pthread mutex implementation behaves correctly in a case of three threads.
* A C source file with a certain floating point function has the expected output.
* A linker can link Clang.
* A Clang built by GCC can build itself.
* A 2-stage Clang is identical to a 3-stage Clang. Build reproducibility.
* A debugger can handle some C++ syntax emitted by a compiler.
* A build system supports a particular build configuration.
* A compiler runs on a particular OS.
* The new Clang can build a collection of software.


Another way of classification.

### Regression testing

A bug is fixed or a behavior is changed. Can we develop a test to distinguish the new state from the previous state?

### Performance testing

Measure the build speed, intermediate file (usually object files) sizes, latency, throughput, time/memory/disk usage, CPU/network utilization, etc.

Many metrics can be refined arbitrarily.

Oh, how is network relevant in toolchain testing? Think of a debug client talking to a debug server.

### Compatibility testing

* Can object files produced by the different versions of compiler linked together?
* Can different translation units have different versions of C++ standard library? (In libc++, this requires `_LIBCPP_HIDE_FROM_ABI_PER_TU`.)
* Can the new version of libc/dynamic loader work with programs built with an older version of libc/dynamic loader?

### Random testing

### Fault injection

For example, we can utilize strace's syscall fault injection feature.

## Common problems

### I don't know whether a test is needed

When in doubt, the answer is usually yes.
Bug fixes and behavior changes are usually interesting.

In some tools, it seems valuable to test the erroneous cases.
But sometimes erroneous cases are difficult to construct, and many people tend to omit tests for them.

### I don't know where to add a test

Ideally, the documentation should make it clear how to run the entire testsuite.
A good continuous integration system can improve testsuite discoverability.
One may check its configuration to figure out how to build and test the project.

You can comment out code that your patch has modified, or make an intentional mistake, then run the whole testsuite to locate relevant tests.
You can use `git log -- file` to learn the history of these tests.

Many projects have a significant barrier to entry because they may require a very specific environment which is difficult to set up.
Not-so-good problems:

* you need to install XXX container technology.
* if you use Debian, it is difficult to install XXX.
* you need to install A, B, and C in a virtual machine/container image.
* if you test on Debian, you likely get more failures than on Fedora.

In llvm-project, if you change `llvm/`, run `ninja check-llvm`; if you change `clang/`, run `ninja check-clang`.
Some cross-toplevel-directory dependencies may be less clear.
If you remove a function from `clang/include/clang/` or change a function signature, you may need `ninja check-clang-tools`.
If you change optimization/debug information generation/code generation, you may need `check-clang check-flang` for a small number of code generation tests.

### I don't know an existing test can be enhanced

Adding a new file is much easier than finding the right file for extension.
A new contributor may be impatient at understanding the file hierarchy or reading existing tests.
A reviewer can suggest the right file for extension, if reusing a file is the right call.

When can reusing an existing test be better?

* The file also exercises the behavior but just misses some behavior checks.
* The changed behavior is related to an option, if placed side by side with another set of invocations, can improve readability/discoverability/etc.
* The new test shares some preparation steps with an existing test.

Sometimes, refactoring existing tests can be done before adding the new test. Adding a generalized test can enable deletion of existing tests.

### The test checks too little

A reader may have difficulty understanding the intention of the test.

The test may not reliably test the behavior change or bug fix. It may become stale and irrelevant as soon as the modified functions/modules are slightly changed.

Scroll down for antipatterns in some GNU toolchain projects that may give inspiration.

A regression test which simply runs the compiler and expects it not to crash has low utilization.
It is often better to additionally test some properties of the produced object file.

For example, a patch fixing an assertion failure just added `# RUN: ld.lld %t.so -o /dev/null`.
```diff
 uint32_t AMDGPU::calcEFlags() const {
-  assert(!objectFiles.empty());
+  if (objectFiles.empty())
+    return 0;
```

I suggest that we test the `e_flags` field, to better utilize the test:
```
# RUN: ld.lld %t.so -o %t
# RUN: llvm-readobj --file-header %t | FileCheck %s

...
```

### The test checks too much

It can harm readability and orthogonality.
It's painful to update many tests with the same characteristics.

The test may incur frequent updates for quite unrelated changes. The person doing that unrelated change needs to waste a few seconds/minutes on updating every affected test and sometimes understanding them.

In llvm-project, there are many LLVM CodeGen and Clang tests which have such maintenance problems.
This problem can sometimes be mitigated. E.g. `llvm/utils/update_llc_test_checks.py` usually generates a huge amount of output.
The same detail may be repeatedly checked by many test files. Fortunately the automation makes test update straightforward.
The convenience is often weighed over the noise.

In `lld/test/ELF`, we have many tests that check exact addresses, but we don't have an auto-update tool.
The tests make address affecting changes (changing the [section layout](https://reviews.llvm.org/D58892), adding a new program header or dynamic tag) very tricky.
On the other hand, they test some details which are hard to test by other means.
Technically we can use more of captures and expressions to decrease sensitivity, but it requires more efforts on writing a test.
In the old days this was often neglected.

### The test checks at the wrong layer

This anti-pattern may lead to two problems: "spooky actions at a distance" and flaky failures for non-issues.

For the first point, developers working on a layer may be familiar with tests at the layer and somewhat familiar with nearby layers.
If a patch causes a test many layers above to fail, it can take time to pinpoint the issue.
When working on a patch, the author may have to run more tests than needed to catch issues, slowing down the development.

For the second point, such tests are oftentimes sensitive and can pass or fail for unrelated reasons.
They may incur frequent updates for unrelated changes.
When porting the toolchain to a new platform, different guarantees may lead to failures. It may not be obvious to know whether it is an important issue or not.

Clang is responsible for producing an IR.
The middle end optimization is responsible for optimizing the IR.
The code generator is responsible for translating the IR into a machine form, optimizing the machine form, and emitting the assembly file/object file.
The linker is responsible for linking object files and runtime libraries.

For example, a `.c` to `.s` test is often undesired: it may be affected by changes to many intermediate layers.
It's recommended to split this into two steps: (1) `.c` => `.ll` (IR generation), (2) `.ll` => `.ll` (middle-end optimizations), (3) `.ll` => `.s` (assembly generation).

## Command line option compatibility

In Clang, LLVM codegen operations and cc1 options are accessible via `-mllvm` and `-Xclang`.
There is no guarantee about the compatibility.

On the other hand, driver options have compatibility requirements.

## Examples

### binutils-gdb

The test harness is based on DejaGNU. Ian Lance Taylor [described DegaGNU's disadvantages in 2011](https://www.airs.com/blog/archives/499).

In `binutils/testsuite/lib/binutils-common.exp`, `run_dump_test` is the core that runs `as` and `ld`, and checks the output of a dump program matches the expected patterns in the .d file.

Let's take a whirlwind tour and figure out the numerous problems.

First, test discoverability.
AFAIK, `make check-ld`, `make check-gas`, `make check-binutils` are not documented. `make check` runs many tests which a contributor may not care about.

Many test files do not have descriptive names. For example, there are `ld/testsuite/ld-elf/group{1,2,3a,3b,4,5,6,7,8a,8b,9a,9b,10,12,12}.d`.
We can tell from the filenames they are related to section groups, but we cannot tell what an individual file is.

If you open an arbitrary `group*.d` file, it is difficult to know its intention.
Either there is no comment, or the comment just explains why it excludes execution on some targets.

```sh
% cat ld/testsuite/ld-elf/group2.d
#source: ../../../binutils/testsuite/binutils-all/group.s
#ld: -r
#readelf: -Sg --wide
# xstormy uses a non-standard script, putting .data before .text.
#xfail: xstormy*-*-*

#...
  \[[ 0-9]+\] \.group[ \t]+GROUP[ \t]+.*
#...
  \[[ 0-9]+\] \.text.*[ \t]+PROGBITS[ \t0-9a-f]+AXG.*
#...
  \[[ 0-9]+\] \.data.*[ \t]+PROGBITS[ \t0-9a-f]+WAG.*
#...
COMDAT group section \[[ 0-9]+\] `\.group' \[foo_group\] contains . sections:
   \[Index\]    Name
   \[[ 0-9]+\]   .text.*
#...
   \[[ 0-9]+\]   .data.*
#pass
```

Unfortunately `git log` gives very little clue, because many commits touching these files don't document the intention in their commit messages.

```text
commit 6a0d0afdc7ca5c7e0605ede799e994c98d596644
Author: hidden
Date:   Thu Oct 20 10:06:41 2005

    binutils/testsuite/

    2005-10-20  hidden

        PR ld/251
        * binutils-all/group.s: New file.

        * binutils-all/objcopy.exp (objcopy_test_readelf): New
        procedure.
        Use it to test ELF group.

    ld/testsuite/

    2005-10-20  hidden

        PR ld/251
        * ld-elf/group.2d: New file.
```

Now you probably find more problems. There is no way to run multiple ld tests in one file. (Well, running the GCC testsuite can be parallelized.)
If you want to alter the command line options a bit, you have to create a new test file and add some lines in an `.exp` file.

The test uses `\[[ 0-9]+\]` to match section indexes.
To make better use of the test, you may want to test that the section group indexes match their indexes in the section header table.
Unfortunately `run_dump_test` provides no regex capture functionality.

You probably notice that some readelf tests are somehow placed in the `ld/testsuite` directory.

`check-ld` is slow. In an edit-compile-test cycle, a contributor wants to know how to run a reduced set of tests which are usually sufficient to catch common problems.
So how do I run just one test file?
It seems that you can't, but you can run a single `*.exp` file using this obscure command: `make check-ld RUNTESTFLAGS=ld-elf/shared.exp`.
You can use `make check-ld RUNTESTFLAGS=shared.exp=glob` to skip some tests. Unfortunately many tests are run regardless of the filter pattern.

I filed a feature request: [binutils/testsuite/lib/binutils-common.exp: Support free-form shell commands and check patterns](https://sourceware.org/PR28602).

### GCC

GCC's test harness is also based on DejaGNU. However, it employs [many directives](https://gcc.gnu.org/onlinedocs/gccint/Test-Directives.html), which offer more functionality than binutils' direstives.
Unfortunately, the directives are quite under-powered and the composability is limited.

A notable limitation is that GCC test files cannot invoke the compiler twice within a single test.
To work around this, developers often duplicate an existing test file and modify it for additional test cases, which can be cumbersome and error-prone.

In the `gcc/testsuite/gcc.target/` directory, some test files include directives like `/* { dg-final { scan-assembler "vhadd.u16"  }  } */`.
They typically check just one instruction and lack context. For example:

```text
/* Verify we that use %rsp to access stack.  */
/* { dg-final { scan-assembler-not "%esp" } } */
/* { dg-final { scan-assembler "%rsp" } } */
```

The final directive checks the presence of `%rsp`, but key details are missing.
What instruction uses `%rsp`? What is the other operand? Is `%rsp` emitted in the correct location?
Without this context, the directive's usefulness is limited.

### Clang driver

`clang/test/Driver/linux-cross.cpp` has some nice tests.

```test
// UNSUPPORTED: system-windows

/// Test native x86-64 in the tree.
// RUN: %clang -### %s --target=x86_64-linux-gnu --sysroot=%S/Inputs/debian_multiarch_tree \
// RUN:   -ccc-install-dir %S/Inputs/basic_linux_tree/usr/bin -resource-dir=%S/Inputs/resource_dir \
// RUN:   --stdlib=platform --rtlib=platform 2>&1 | FileCheck %s --check-prefix=DEBIAN_X86_64
// DEBIAN_X86_64:      "-resource-dir" "[[RESOURCE:[^"]+]]"
// DEBIAN_X86_64-SAME: "-internal-isystem"
// DEBIAN_X86_64-SAME: {{^}} "[[SYSROOT:[^"]+]]/usr/lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10"
// DEBIAN_X86_64-SAME: {{^}} "-internal-isystem" "[[SYSROOT:[^"]+]]/usr/lib/gcc/x86_64-linux-gnu/10/../../../../include/x86_64-linux-gnu/c++/10"
// DEBIAN_X86_64-SAME: {{^}} "-internal-isystem" "[[SYSROOT]]/usr/lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/backward"
// DEBIAN_X86_64-SAME: {{^}} "-internal-isystem" "[[RESOURCE]]/include"
// DEBIAN_X86_64-SAME: {{^}} "-internal-isystem" "[[SYSROOT]]/usr/local/include"
// DEBIAN_X86_64-SAME: {{^}} "-internal-isystem" "[[SYSROOT]]/usr/lib/gcc/x86_64-linux-gnu/10/../../../../x86_64-linux-gnu/include"
/// We set explicit -ccc-install-dir ensure that Clang does not pick up extra
/// library directories which may be present in the runtimes build.
// DEBIAN_X86_64:      "-L
// DEBIAN_X86_64-SAME: {{^}}[[SYSROOT]]/usr/lib/gcc/x86_64-linux-gnu/10"
/// Debian patches MULTILIB_OSDIRNAMES (../lib64 -> ../lib), so gcc uses 'lib' instead of 'lib64'.
/// This difference does not matter in practice.
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/usr/lib/gcc/x86_64-linux-gnu/10/../../../../lib64"
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/lib/x86_64-linux-gnu"
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/lib/../lib64"
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/usr/lib/x86_64-linux-gnu"
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/usr/lib/../lib64"
/// /usr/x86_64-linux-gnu does not exist, so there is no /usr/lib/gcc/x86_64-linux-gnu/10/../../../../x86_64-linux-gnu/lib.
/// -ccc-install-dir is not within sysroot. No bin/../lib.
/// $sysroot/lib and $sysroot/usr/lib. Fallback when GCC installation is unavailable.
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/lib"
// DEBIAN_X86_64-SAME: {{^}} "-L[[SYSROOT]]/usr/lib"
```

It marks Windows as unsupportted: while it is generally nice to make tests runnable on as many platforms as possible, sometimes generality can hinder test readability/value.
On Windows, Clang driver uses both `/` and `\` as path separators, and where `\` is needed is quite unpredictable.
If we want to support Windows, we have to replace many `/` below with `{{/|\\\\}}`, which will be a huge pain.

The Clang driver invocation prints many `-internal-isystem`. The order and completeness matters.
If we use something like `"-internal-isystem" "{{.*}}/usr/local/include"`, we cannot know how many `-internal-isystem` options this pattern matches.
`...-SAME: {{^}} "-internal-isystem" "...` allows us to wrap lines and assert that the options appear as in the written order.

We use `DEBIAN_X86_64:      "-internal-isystem"` to match the first occurrence.
Not placing the value on the same line is intentional. If things go wrong, FileCheck will give us a better diagnostic.

### split-file

In Aug 2020, I added `split-file` to llvm-project which allows you to place multiple extra files in one file.

Use case A (organizing input of different formats (e.g. linker
script+assembly) in one file).

```text
# RUN: split-file %s %t
# RUN: llvm-mc %t/asm -o %t.o
# RUN: ld.lld -T %t/lds %t.o -o %t
This is sometimes better than the %S/Inputs/ approach because the user
can see the auxiliary files immediately and don't have to open another file.

#--- asm
...
#--- lds
...
```

Use case B (for utilities which don't have built-in input splitting feature):

```text
// RUN: split-file %s %t
// RUN: llc < %t/1.ll | FileCheck %s --check-prefix=CASE1
// RUN: llc < %t/2.ll | FileCheck %s --check-prefix=CASE2
Combing tests prudently can improve readability.
For example, when testing parsing errors if the recovery mechanism isn't possible,
grouping the tests in one file can more readily see test coverage/strategy.

//--- 1.ll
...
//--- 2.ll
...
```

### lit macro-like substitutions

In 2022 lit [introduced](https://reviews.llvm.org/D132513) `DEFINE` and `REDEFINE` directives.
This feature allows running a test multiple times with different configurations.
See <https://llvm.org/docs/TestingGuide.html#test-specific-substitutions> for detail.

### Use the `CHECK-NEXT` directive

If `-NEXT` can be used, it's almost always better than not using `-CHECK`:
```
# CHECK:      AAA
# CHECK-NEXT: BBB
# CHECK-NEXT: CCC
```

... is likely better than
```
# CHECK:      AAA
# CHECK:      BBB
# CHECK:      CCC
```

If someone makes a change so that the next line no longer contains `BBB`, `# CHECK-NEXT: BBB` will cause FileCheck to report a precise diagnostic.
Let's say after the change, `BBB` is still present but after `CCC`, then FileCheck will report a diagnostic about `BBB` for the first example while report an unmatched `CCC` for the second example.
The first example is more desired to pinpoint the issue.

By using `-CHECK`, there is a small possibility that the arguments in "the test checks too much" (updates for quite unrelated changes) will apply, but in my experience this is quite rare.

### Pay attention to the `CHECK-NOT` directive

Let's say we use the [`-NOT` directive](https://llvm.org/docs/CommandGuide/FileCheck.html#the-check-not-directive) to check that a string does not occur between two lines.
```
# CHECK:      AAA
# CHECK-NOT:  warning: unknown platform
# CHECK:      BBB
```

It is easy to make a typo so that the `-NOT` directive does not test anything.
Adjustment to the diagnostic message can render this directive not to what it intended as well.

There are multiple possible replacements. One is to omit the detailed diagnostic message.
```
# CHECK:      AAA
# CHECK-NOT:  warning:
# CHECK:      BBB
```

However, in many times, using `-NEXT` is better, which additionally asserts that there can be no extra line in between.
```
# CHECK:      AAA
# CHECK-NEXT: BBB
```

Sometimes we want to check that there is no `warning:` before `AAA` or after `BBB`.
In such a scenario we can use `FileCheck --implicit-check-not=warning:`.

## Random notes

Compare symbol size changes:

```zsh
nvim -d =(llvm-nm -U --size-sort /tmp/c/0) =(llvm-nm -U --size-sort /tmp/c/1)
```
<!--
nvim -d =(fllvm-nm -U --size-sort /t/lld0) =(fllvm-nm -U --size-sort /t/lld1)
-->

Compare disassembly listings of specified functions in two objects:

```zsh
nvim -d =(llvm-objdump -d --symbolize-operands -M intel --no-leading-addr --no-show-raw-insn /tmp/c/0 --disassemble-symbols=$(</tmp/0)) =(llvm-objdump -d --symbolize-operands -M intel --no-leading-addr --no-show-raw-insn /tmp/c/1 --disassemble-symbols=$(</tmp/0))
```
