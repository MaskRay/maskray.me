---
layout: post
title: -march=, -mcpu=, and -mtune=
author: MaskRay
tags: [clang,gcc]
---

Updated in 2025-05.

In GCC and Clang, there are three major options specifying the architecture and microarchitecture the generated code can run on.
The general semantics are described below, but each target machine may assign different semantics.

* `-march=X`: (execution domain) Generate code that can use instructions available in the architecture X
* `-mtune=X`: (optimization domain) Optimize for the microarchitecture X, but does not change the ABI or make assumptions about available instructions
* `-mcpu=X`: Specify both `-march=` and `-mtune=` but can be overridden by the two options. The supported values are generally the same as `-mtune=`. The architecture name is inferred from `X`

<!-- more -->

Below are some notes about GCC's implementation.

## AArch32

See <https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html>.

AArch32 follows the general description.

`-march=name[+extension...]` may specify architecture extensions.

In September 2024, [GCC introduced `-march=unset`](https://gcc.gnu.org/git/?p=gcc.git&a=commit;h=7d6c6a0d15c136a68d066c60da0f48265a2b1886) to override a previously set `-march=`, enabling the use of an incompatible `-mcpu=` value with `-mcpu=something -march=unset`.

## AArch64

See <https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html>.

AArch64 follows the general description.

See [Compiler flags across architectures: -march, -mtune, and -mcpu](https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/compiler-flags-across-architectures-march-mtune-and-mcpu).

`gcc/config/aarch64/aarch64.cc:aarch64_validate_march` verifies that `-march=` specifies a value defined in `all_architectures`.
`gcc/config/aarch64/aarch64.cc:aarch64_validate_mtune` verifies that `-mtune=` specifies a value defined in `all_cores`.

The selected architecture is stored in `gcc/config/aarch64/aarch64.opt:selected_arch` (a `TargetVariable`).
The selected tune microarchitecture is stored in `gcc/config/aarch64/aarch64.opt:selected_tune` (a `TargetVariable`).

## MIPS

See <https://gcc.gnu.org/onlinedocs/gcc/MIPS-Options.html>.

`-march=` supports generic ISA names and processor names.

`-mcpu=` is not implemented in GCC.

## PowerPC

See <https://gcc.gnu.org/onlinedocs/gcc/RS_002f6000-and-PowerPC-Options.html>.

`-march=` is not implemented in GCC.
```
% powerpc64le-linux-gnu-gcc -fsyntax-only -march=power10 a.c
powerpc64le-linux-gnu-gcc: error: unrecognized command-line option ‘-march=power10’; did you mean ‘-mcpu=power10’?
```

## RISC-V

RISC-V follows the general description.

`-mtune=` must specify an element in `gcc/config/riscv/riscv-cores.def`.
The default `-mcpu=` value is `RISCV_TUNE_STRING_DEFAULT`.

`-mcpu=` does not seem to infer `-march=` in GCC.

## Sparc

`-mcpu=` must specify an element in `gcc/config/sparc/sparc.md`.

`-march=` is not allowed.

## x86

See <https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html>.

`-mcpu=` has been [deprecated since 2003-02](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=9d913bbf3fc996874649168d7d144a642012ac9b). It is an alias for `-mtune=`.

`-march=` specifies a _cpu-type_. _cpu-type_ is a microarchitecture name (e.g. `skylake`, `znver3`), a microarchitecture level (`x86-64`, `x86-64-v2`, `x86-64-v3`, `x86-64-v4`), or `native`.
`-march=native` enables all instruction subsets supported by the local machine and recognized by the GCC version.

If `-mtune=` is not specified, use the `-march=` value (if a microarchitecture name) or `generic`.
```text
% gcc -Q --help=target -march=skylake a.c |& grep --color 'march\|mtune\|mcpu'
  -march=                               skylake
  -mcpu=
  -mtune-ctrl=
  -mtune=                               skylake
  Known valid arguments for -march= option:
  Known valid arguments for -mtune= option:
```

`-march=` and `-mtune=` must specify an element in `gcc/common/config/i386/i386-common.cc:processor_alias_table`.
`-mtune=` must specify an element without the `PTA_NO_TUNE` flag.

`-mtune=` decides features in `gcc/config/i386/i386-options.cc:ix86_tune_features`:
```text
% gcc -c -mdump-tune-features a.c
List of x86 specific tuning parameter names:
schedule : on
partial_reg_dependency : on
sse_partial_reg_dependency : on
sse_split_regs : off
...
```
`-mtune=` decides instruction costs `gcc/config/i386/i386-options.cc:processor_cost_table`.

`-mtune=` defines some built-in macros `__tune_*` (see `gcc/config/i386/i386-c.cc:ix86_target_macros_internal`).

### `x86-64, x86-64-v2, x86-64-v3, x86-64-v4`

The x86-64 psABI defines some microarchitecture levels. They are a very small set of choices suitable for a Linux distribution default with a neutral name (i.e. not tied to an AMD or Intel microarchitecture name).
See [New x86-64 micro-architecture levels](https://sourceware.org/pipermail/libc-alpha/2020-July/116135.html) for the original proposal on libc-alpha.

One major use case is [glibc's hwcap support for x86-64](https://sourceware.org/git/?p=glibc.git;a=commit;h=f267e1c9dd7fb8852cc32d6eafd96bbcfd5cbb2b).

I implemented `x86-64-v2`, `x86-64-v3`, and `x86-64-v4` for Clang.

## Recommendation

For the simplest case where only one option is desired, use `-march=` for x86 and `-mcpu=` for other targets.

When optimizing for the local machine, just use `-march=native` for x86 and `-mcpu=native` for other targets.

When the architecture and microarchitecture are both specified, i.e. when both the execution domain and the optimization domain need to be specified, specify `-march=` and `-mtune=`, and avoid `-mcpu=`.
On PowerPC, specify both `-mcpu=` and `-mtune=`.

## Clang

Clang driver parses `-march=`, `-mcpu=`, `-mtune=`, and pass `-target-cpu`, `-tune-cpu`, `-target-feature` to cc1.

`-mtune=` mostly affects `MCSubtargetInfo::CPUSchedModel` and target features.

For each target, `-mcpu=native` is implemented by `llvm::sys::getHostCPUName`.

### `--target=`

Clang supports `--target=` to set the target triple (for cross compilation).
The specified target should be built into Clang (see `LLVM_TARGETS_TO_BUILD` and `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD`).
See [Compiler driver and cross compilation](/blog/2021-03-28-compiler-driver-and-cross-compilation#clang).

`--target=aarch64-unknown-linux-gnu -mcpu=X` can be used to cross compile for AArch64 and optimize for the microarchitecture X.

## Function multi-versioning

GCC 4.4 [added](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=5779e7133d84c5873249bb643d9852f314022f0b) the function attribute [`target`](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#:~:text=target%20%28) for x86.
`__attribute__((target(arch=k8)))` is supported to override `-march=` for a function.
GCC 4.8 [extended](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=f80e0faf19690e5c92ca8b3eb5e920855e39c758) the syntax with `"default"` which is then named function multi-versioning.

GCC 6 [added](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=3b1661a9b93fe8000faa6ab4b721a96ffb48d525) the function attribute [`target_clones`](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#:~:text=target_clones%20%28) to support a new style of function multi-versioning for x86.
`target_clones` supports multiple `arch=X`.

## GNU assembler

`-march=`, `-mcpu=`, and `-mtune=` are optionally supported by various ports.

When `-march=` is defined, the assembler will report an error when assembling an instruction not supported by the specified architecture.

`-mtune=` is implemented for mips, ia64 (Itanium), visium, and x86.
The Itanium port uses `-mtune=` just to [decide the desired NOP sequence](https://sourceware.org/PR803). The x86 port `-mtune=` copies this design in 2006.

Clang does not support `-Wa,-mtune=`. I dropped a glibc use in [i386: Remove -Wa,-mtune=i686](https://sourceware.org/git/?p=glibc.git;a=commit;h=c5bec9d491c5d066dd238ccafcdec78cd4592e8e).
