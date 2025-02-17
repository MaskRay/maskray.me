layout: post
title: Clang -Wunused-command-line-argument
author: MaskRay
tags: [llvm]
---

clangDriver is the library implementing the compiler driver for Clang.
It utilitizes LLVMOption to process command line options.
As options are processed when required, as opposed to [use a large switch](/blog/2020-10-03-llvm-command-line-processing), Clang gets the ability to detect unused options straightforwardly.

<!-- more -->

When an option is checked with APIs such as `hasArg` and `getLastArg`, its "claimed" bit is set.
```cpp
       template<typename ...OptSpecifiers>
       Arg *getLastArg(OptSpecifiers ...Ids) const {
         Arg *Res = nullptr;
         for (Arg *A : filtered(Ids...)) {
           Res = A;
           Res->claim();
         }
         return Res;
       }
```

After all options are processed, Clang reports a `-Wunused-command-line-argument` diagnostic for each unclaimed option.
There are multiple possible messages, but `argument unused during compilation` is the most common one.
```
% clang -c a.c -la
clang: warning: -la: 'linker' input unused [-Wunused-command-line-argument]
% clang -c -Werror a.c -la
clang: error: -la: 'linker' input unused [-Werror,-Wunused-command-line-argument]
% clang -c -Werror=unused-command-line-argument a.c -la
clang: error: -la: 'linker' input unused [-Werror,-Wunused-command-line-argument]
```

There are many heuristics to enhance the desirability of `-Wunused-command-line-argument`, which can be rather subjective.
For instance, options that are relevant only during compilation do not result in `-Wunused-command-line-argument` diagnostics when linking is performed.
This is necessary to support linking actions that utilitize `CFLAGS` or `CXXFLAGS`.
```
% clang -faddrsig -fpic -march=generic a.o
```

## Where options are claimed

In clangDriver, there are many job actions, e.g. preprocess, precompile, compile, backend, assemble, link.
For most actions (preprocess/precompile/compile/backend/etc), `clang::driver::tools::Clang` is selected.
`Clang::ConstructJob` is where most compilation only options are handled.
Target-specific options are handled by `Clang`'s member function `RenderTargetOptions`.

For assembly files that do not need preprocessing (e.g. `.s` (`clang::driver::types::TY_PP_Asm`)), the driver selects `clang::driver::tools::ClangAs` to use the integrated assembler.
`ClangAs::ConstructJob` claims very few options.

For a link job, oftentimes `clang::driver::tools::gcc::Linker` is selected.
Its `ConstructJob` forwards `Link_Group` and `LinkOption` options to the linker, and processes `-Wa,` options.

## Assembly files

Assembling a `.s` file uses `ClangAs`. As mentions, it claims very few options and we can get many `-Wunused-command-line-argument` diagnostics.
```
% gcc -S -fpic -mfpmath=sse a.c
% gcc -c -fpic -mfpmath=sse a.s
% clang -c -fpic -mfpmath=sse a.s
clang: warning: argument unused during compilation: '-mfpmath=sse' [-Wunused-command-line-argument]
```

Preprocessing and assembling a `.S` file selects `Clang`.
We will get behaviors similar to compiling a `.c` file: compilation only options do not lead to diagnostics.
```
% clang -c -fpic -mfpmath=sse a.S
```

## Default options

There is a tension between `-Wunused-command-line-argument` and default options.
Let's consider a scenario where we specify `--rtlib=compiler-rt --unwindlib=libunwind` in `CFLAGS` and `CXXFLAGS` to utilize compiler-rt and LLVM libunwind.
ClangDriver claims `--rtlib` and `--unwindlib` in the following code snippet:
```cpp
if (!Args.hasArg(options::OPT_nostdlib, options::OPT_r)) {
  if (!Args.hasArg(options::OPT_nodefaultlibs)) {
    // Handle --rtlib and --unwindlib.
  }
  if (!Args.hasArg(options::OPT_nostartfiles)) {
    // Handle --rtlib. Append clang_rt.crtend.o or GCC style crtend{,S,T}.o
  }
}
```

However, if a build target employs `-nostdlib` or `-nodefaultlibs`, options such as `--rtlib`, `--unwindlib`, and many other linker options (e.g. `-static-libstdc++` and `-pthread`) will not be claimed, resulting in unused argument diagnostics:
```
% clang --rtlib=compiler-rt --unwindlib=libunwind -stdlib=libstdc++ -static-libstdc++ -pthread -nostdlib a.o
clang: warning: argument unused during compilation: '--rtlib=compiler-rt' [-Wunused-command-line-argument]
clang: warning: argument unused during compilation: '--unwindlib=libunwind' [-Wunused-command-line-argument]
clang: warning: argument unused during compilation: '-static-libstdc++' [-Wunused-command-line-argument]
clang: warning: argument unused during compilation: '-pthread' [-Wunused-command-line-argument]
```

While some options like `-stdlib=` do not trigger a diagnostic, this seems more like a happenstance rather than a deliberate design choice.

To suppress the diagnostics, we can utilitize [`--start-no-unused-arguments` and `--end-no-unused-arguments`](https://reviews.llvm.org/D116503) (Clang 14) like the following:
```
% clang --start-no-unused-arguments --rtlib=compiler-rt --unwindlib=libunwind -stdlib=libstdc++ -static-libstdc++ -pthread --end-no-unused-arguments -nostdlib a.o
```

There is also a heavy hammer `-Qunused-arguments` to suppress `-Wunused-command-line-argument` diagnostics regardless of options' positions:
```
% clang -Qunused-arguments --rtlib=compiler-rt --unwindlib=libunwind -stdlib=libstdc++ -static-libstdc++ -pthread -nostdlib a.o
```

`-Qunused-arguments` is similar to `-Wno-unused-command-line-argument`.

Conveniently, options specified in a Clang configuration file are automatically claimed.

```sh
cat >a.cfg <<e
-stdlib=libstdc++
--rtlib=libgcc
--unwindlib=libgcc
e
clang --config=./a.cfg a.c            # no diagnostic
mkdir bin lib
ln -s /usr/bin/clang bin/
cp a.cfg bin/clang.cfg
ln -s /usr/lib/clang lib/
bin/clang -no-canonical-prefixes a.c  # no diagnostic
```

In the last command, I specify `-no-canonical-prefixes` so that clang will find `dirname(clang)/clang.cfg`, otherwise Clang would try to find `dirname(realname(clang))/clang.cfg`.

## Target-specific options

GCC has [machine-dependent options](https://gcc.gnu.org/onlinedocs/gcc/Submodel-Options.html) that usually begin with `-m`.
These options are often referred to as "machine-specific" or sometimes as "target-specific".
I am not certain whether there are any distinctions in these terms within the context of GCC.
In Clang, we prefer the term "target-specific".

For instance, certain targets implement `-mtls-dialect=`.
This is achieved through files like `gcc/config/aarch64/aarch64.opt`. If we use such options on unsupported targets, we will encounter an error:

```
% aarch64-linux-gnu-gcc -c -mtls-dialect=desc a.c
% powerpc64le-linux-gnu-gcc -c -mtls-dialect=desc a.c
powerpc64le-linux-gnu-gcc: error: unrecognized command-line option ‘-mtls-dialect=desc’
```

However, Clang employs a unified table named `clang/include/clang/Driver/Options.td` for all options, thereby eliminating the need for maintaining target-specific lists.
Historically, an unsupported option [merely issued a warning](https://github.com/llvm/llvm-project/issues/64632) (unless `-Werror` is used):
```
% clang-16 -c --target=aarch64 -mavx a.c
clang-16: warning: argument unused during compilation: '-mavx' [-Wunused-command-line-argument]
```

For Clang 17, I have [implemented](https://reviews.llvm.org/D151590) the `TargetSpecific` flag in `clang/include/clang/Driver/Options.td` and annotated numerous options accordingly.
If an option is annotated as `TargetSpecific`, the `-Wunused-command-line-argument` diagnostic will be elevated to an error
```
% clang -c --target=aarch64 -mavx a.c
clang: error: unsupported option '-mavx' for target 'aarch64'
```

For `-x assembler` input, clangDriver [recognizes assembler tools and suppresses warning-to-error promotion](https://reviews.llvm.org/D159173).
```
% clang -c -mfpmath=sse a.s
clang: warning: argument unused during compilation: '-mfpmath=sse' [-Wunused-command-line-argument]
```

I have mentioned that

> Options that are relevant only during compilation do not result in `-Wunused-command-line-argument` diagnostics when linking is performed.

They also don't lead to "unsupported option" errors when linking is performed.
Certain driver options only affect linking are marked as `LinkerInput` in Clang.

```
% clang --target=aarch64 -mavx a.c -fdriver-only      # should report an error but not
clang: error: unsupported option '-mavx' for target 'aarch64'
% clang --target=aarch64 -mavx a.c b.o -fdriver-only  # should report an error but not
% clang --target=aarch64 -mavx a.c -lm -fdriver-only  # should report an error but not
```
