layout: post
title: Precompiled headers
author: MaskRay
tags: [binutils,linker,llvm]
---

C/C++ projects can benefit from using precompiled headers to improve compile time.
GCC [added support for precompiled headers](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=17211ab55314d76370a68036f2d057b1effd687f) in 2003 ([version 3.4](https://gcc.gnu.org/gcc-3.4/changes.html)), and the current documentation can be found at <https://gcc.gnu.org/onlinedocs/gcc/Precompiled-Headers.html>.

Even with the emergence of C++ modules, precompiled headers remain relevant for several reasons:

* Precompiled headers share implementation aspects with modules (e.g., AST serialization in Clang).
* Many C++ projects rely on the traditional compilation model and are not converted to C++ modules.
* Modules may possibly use some preamble-like technology to accelerate IDE-centric operations.
* C doesn't have C++ modules.

This article focuses on Clang precompiled headers (PCH). Let's begin with an example.

<!-- more -->

```sh
cat > a1.cc <<'eof'
#include "b.hh"
int fa1() { return fb(); }
eof
cat > a.cc <<'eof'
#include "b.hh"
int fa1();
int main() { return fa1() + fb(); }
eof
cat > b.hh <<'eof'
#ifndef B_HH
#define B_HH
inline int fb() { return 42; }
#endif
eof

clang -c b.hh -o b.hh.pch            # cc1 action is -emit-pch
clang -c -include-pch b.hh.pch a.cc  # or -include b.hh
clang -c -include-pch b.hh.pch a1.cc
clang++ a.o a1.o -o a
```

We compile `b.hh` using `-c`, just like we would compile a non-header file.
Clang parses the file, performs semantic analysis, and writes the precompiled header (as a serialized AST file) into `b.hh.pch`.

When compiling `a.cc`, we use `-include-pch` as a prefix header.
This means that the translation unit will get two `b.hh` copies: one from `b.hh.pch` and one from the textual `b.hh`. The same applies to `a1.cc`.
To avoid a `redefinition of 'fb'` error, `b.hh` should have a header guard or use `#pragma once`.

Now, let's examine the steps in detail.

## PCH generation

Given a header file as input, Clang determines the input type as either `c-header` (`.h`) or `c++-header` (`.hh`/`.hpp`/`.hxx`) based on the file extension.

For compilation actions, either `clang` or `clang++` can be used.
If we treat `.h` as a C++ header, we need to specify `-xc++-header` (e.g., `clang -c -xc++-header b.h -o b.h.pch`).
(It's worth noting that the behavior of `clang++ -c a.h` is deprecated.
Other than that, the only significant difference between `clang` and `clang++` is the linking process, specifically whether the C++ standard library is linked.)

When the input type is `c-header` or `c++-header`, Clang Driver selects the `-emit-pch` frontend action.
(Note: `c++-user-header`/`c++-system-header` are used for C++ modules and have different functionality.)

Conventionally, the extension used for Clang precompiled headers is `.pch` (similar to MSVC).
However, to match GCC, when the `-o` option is omitted, the default output file for older Clang versions is `input_file + ".gch"` (see `Driver::GetNamedOutputPath`).
For example, when the input file is `d/b.hh`, the output is `d/b.hh.gch`. This is different from `d/b.cc` that will go to `b.o`.

The frontend parses the file, performs semantic analysis, and writes the precompiled header (as a serialized AST file) (see `PCHGenerator`).
For the serialized format, refer to [Precompiled Header and Modules Internals](https://clang.llvm.org/docs/PCHInternals.html).

## Using PCH

`-include-pch b.hh.pch` (`PreprocessorOptions::ImplicitPCHInclude`) loads the precompiled header `b.hh.pch` as a prefix header.

We can also write `-include b.hh`, and Clang will probe `b.hh.pch`/`b.hh.gch` and use the file if present. This is a behavior ported from GCC.

`-include` and `-include-pch` may specify a directory. Clang will search for a suitable precompiled header in the directory (see `ASTReader::isAcceptableASTFile`).
The directory may contain precompiled headers for different compiler options. This is another behavior ported from GCC.

```sh
echo 'extern int X;' > d.hh
echo 'int use() { return X; }' > e.cc
mkdir d.hh.pch
clang++ -c -DX=x d.hh -o d.hh.pch/x.pch
clang++ -c -DX=y d.hh -o d.hh.pch/y.pch
clang++ -c -DX=x -include-pch d.hh.pch e.cc  # use d.hh.pch/x.pch
clang++ -c -DX=x -include d.hh e.cc          # use d.hh.pch/x.pch
```
```
% clang++ -c -DX=z -include d.hh e.cc
error: no suitable precompiled header file found in directory 'd.hh.pch'
1 error generated.
% clang++ -c -include d.hh e.cc  # clang>=15.0
error: no suitable precompiled header file found in directory 'd.hh.pch'
1 error generated.
```

## PCH validation

When we generate and use a precompiled header with different compiler options, the behavior will be a combination of those options.
Consequently, the behavior of `-include b.hh` may differ depending on the presence of `b.hh.pch`/`b.hh.gch`.

To identify this common pitfall, Clang performs PCH validation (see `PCHValidator`) to check for inconsistent options, similar to how MSVC handles it.
The validated options include those that can affect AST generation, such as language options (driver `-std=`, `-fPIC`, `-fPIE`), target options (driver `--target`), file system options, header search options, and preprocessor options.

Modules employ the same validation mechanism, but PCH validation is stricter (`!AllowCompatibleConfigurationMismatch`). This means that `COMPATIBLE_LANGOPT`/`COMPATIBLE_ENUM_LANGOPT`/`COMPATIBLE_VALUE_LANGOPT` options (e.g., whether the built-in macro `__OPTIMIZE__` is defined) must match as well.

If one side of the precompiled header and the user code are compiled with the `-D` option, the other side should either use the same `-D` option or omit it entirely.

```sh
clang -c b.hh -o b.hh.pch -DB=1

# tolerable
clang -c -include-pch b.hh.pch a.cc

# definition of macro 'B' differs between the precompiled header ('1') and the command line ('2')
clang -c -include-pch b.hh.pch a.cc -DB=2

# error: macro 'B' was defined in the precompiled header but undef'd on the command line
clang -c -include-pch b.hh.pch a.cc -UB
```

As an escape hatch, `-Xclang -fno-validate-pch` disables PCH validation.

## Performance optimization

In order to achieve better performance, it is possible to make certain compromises on properties such as language standard conformance.

### `-fpch-instantiate-templates`

[`-fpch-instantiate-templates`](https://reviews.llvm.org/D69585) allows pending template instantiations to be performed in the PCH file.
This means that these instantiations do not need to be repeated in every translation unit that includes the PCH file.
This optimization can significantly improve the speed of certain projects.
However, the option changes the point of instantiation for certain function templates, which is non-conforming.
Nevertheless, the altered behavior is generally harmless in most cases.

```cpp
#ifndef HEADER
#define HEADER
template<typename T> void f();
struct X;
void g() { f<X>(); } // delayed instantiation

template<typename T> void f() { T t; };
#else
struct X {};
#endif
```

```
% clang++ -c -xc++-header a.cc -o a.pch
% clang++ -c -include-pch a.pch a.cc
% clang++ -c -xc++-header a.cc -o a.pch -fpch-instantiate-templates
d.cc:5:35: error: variable has incomplete type 'X'
    5 | template<typename T> void f() { T t; };
      |                                   ^
...
```

`-fpch-instantiate-templates` is the default when using the clang-cl driver mode, as MSVC appears to have a similar behavior.

# Modular code generation

Modular code generation was [initially implemented](https://reviews.llvm.org/D28845). It was later extended to [support precompiled headers](https://reviews.llvm.org/D69778) in Clang 11.
To utilize this feature, you can specify `-Xclang -fmodules-codegen` as a command-line option or use the driver option `-fpch-codegen`.

When generating a serialized AST file for PCH or modules, Clang identifies non-always-inline functions that do not depend on template parameters and have linkages other than `GVA_Internal` or `GVA_AvailableExternally`.
These functions are then serialized (see `ASTWriter::ModularCodegenDecls`).

In an importer that encounters such a definition, the linkage is adjusted to `GVA_AvailableExternally`. This allows for discarding of the definition if it is not required for optimizations.

Let's consider an example using the files `a.cc`, `a1.cc`, and `b.hh` from the initial example provided at the beginning of this article.
```sh
echo 'module b { header "b.hh" }' > module.modulemap

clang -c -Xclang -emit-module -fmodules -xc++ module.modulemap -fmodule-name=b -o b.pcm -Xclang -fmodules-codegen
clang -c b.pcm
clang -c -fmodules -fno-implicit-module-maps -fmodule-file=b.pcm a.cc
clang -c -fmodules -fno-implicit-module-maps -fmodule-file=b.pcm a1.cc
clang++ a.o a1.o b.o -o a
```

Both `a.cc` and `a1.cc` include `b.hh` and obtain an inline definition of `fb`.
In a regular build, the `fb` definition has `GVA_DiscardableODR` linkage and is compiled twice into `a.o` and `a1.o`.
These duplicate definitions are then deduplicated by the linker, following COMDAT semantics.

In a modular code generation build, `fb` is assigned `GVA_StrongODR` linkage in `b.pcm` and is emitted into `b.o`.
The copies of `fb` in `a.cc` and `a1.cc` are adjusted to `GVA_AvailableExternally`. They are used for optimizations by callers but are not emitted otherwise.
In a `-O0` build, the `GVA_AvailableExternally` definitions are simply discarded.
Regardless, both the code generator and the linker have reduced work, resulting in decreased build time.

However, there are two primary differences in behavior.

First, if `b.hh` contains a `GVA_StrongExternal` definition, a regular build will encounter a linker error due to a duplicate symbol. However, in the prebuilt modules build using `-fmodules-codegen`, this error does not occur.

Second, in a regular build, if `fb` is unused, no translation unit will contain its COMDAT definition. On the other hand, in the prebuilt modules build using `-fmodules-codegen`, we compile the prebuilt module `b.pcm` into `b.o` and link `b.o` into the executable, always resulting in a definition of `fb`, even when using `-O1` or above.
To discard `fb`, linker garbage collection can be leveraged by using `-Wl,--gc-sections` (`-ffunction-sections` is unneeded even for ELF targets, since COMDAT functions are emitted in separate sections anyway).

If `b.hh` contains an inline variable with an initializer involving a side effect (e.g., `inline int vb = puts("vb"), 1;`), the modular code generation build will always observe the side effect.
In contrast, a regular build may not observe the side effect if, for example, the containing header is not included in any translation unit.

Nevertheless, these behavior differences are almost always benign, and the speedup gained in build time may outweigh the downsides.

Note: Clang exhibits a similiar behavior when compiling module interface units and module partitions for strong definitions.

Modular code generation has been extended to support PCH in Clang 11. We specify `-fpch-codegen` to pass `-fmodules-codegen` to the frontend.
```sh
clang++ -c -fpch-codegen b.hh -o b.hh.pch
clang++ -c b.hh.pch -o b.o
clang++ -c -include-pch b.hh.pch a.cc
clang++ -c -include-pch b.hh.pch a1.cc
clang++ a.o a1.o b.o -o a
```

When using `-fpch-codegen`, compared to the non-`-fpch-codegen` usage of PCH, it is necessary to compile the PCH file `b.pch` into `b.o` and link `b.o` into the executable.
If `b.o` is not linked, a linker error for an undefined `fb()` will occur.

Here is a CMake [feature request for `-fpch-codegen`](https://gitlab.kitware.com/cmake/cmake/-/issues/21133).

### `-Xclang -fmodules-codegen`

The cc1 option `-fmodules-debuginfo` serves a similar purpose as `-fmodules-codegen`, but specifically for debug information descriptions of `CXXRecordDecl` that is non-dependent.
When using `-g`, the compiled PCH or module file emit type definitions while importers just emit declarations (`DIFlagFwdDecl`).

`-fpch-debuginfo` instructs Clang Driver to pass `-fmodules-codegen` to the frontend.

## MSVC-style precompiled headers

Here is an example of using MSVC-style precompiled headers with clang-cl (a CL-style driver mode), using the files `a.cc`, `a1.cc`, and `b.hh` from the initial example provided at the beginning of this article.

```sh
clang-cl /c /Ycb.hh a.cc
clang-cl /c /Yub.hh a1.cc
```

The `/Ycb.hh` command instructs the Clang Driver to perform two frontend actions.
First, Clang parses the base source file `a.cc` up to and including `#include "b.hh"`, performs semantic analysis, and writes the precompiled header into `b.pch`.
It replaces the header file extension with `.pch`, unlike GCC.
Second, Clang compiles `a.cc` using `-include-pch b.pch`, but it skips preprocessing tokens up to and including `#include "b.hh"` (see `Preprocessor::SkippingUntilPCHThroughHeader`).

The `/Yub.hh` command is similar to the second frontend action of `/Ycb.hh`.
It compiles `a1.cc` using `-include-pch b.pch`, but it also skips preprocessing tokens up to and including `#include "b.hh"`.

Internally, `/Ycb.hh` and `/Yub.hh` instruct the driver to pass `-pch-through-header=b.hh` to the frontend.
This helps Clang detect common pitfalls by examining whether the source file contains the `#include "b.hh"` directive.

It is also possible to use `/Yc` and `/Yu` without specifying a filename.
In this case, the precompiled header region is determined by `#pragma hdrstop` or the end of the source file.
For more details, refer to [/Yc (Create Precompiled Header File)](https://learn.microsoft.com/en-us/cpp/build/reference/yc-create-precompiled-header-file).

In MSVC, [an inline function with the dllexport attribute](https://learn.microsoft.com/en-us/cpp/cpp/defining-inline-cpp-functions-with-dllexport-and-dllimport) is exported, whether or not it is referenced.
LLVM IR models such a definition with the `weak_odr` linkage.
When using `clang-cl /Yc`, the cc1 option `-building-pch-with-obj` is passed to the frontend.
This option instructs the frontend to serialize inline functions with the dllexport attribute, similar to `-fmodules-codegen`.
In an importer that encounters such a definition, the linkage is adjusted to `GVA_AvailableExternally`.

See also `/Zc:dllexportInlines-`.

TODO: PCH signature and linker

## Precompiled preamble

During IDE-centric operations, such as code completion and reporting diagnostics, an approach called precompiled preamble (introduced to Clang in 2010) is used to minimize the need for re-parsing the entire file after making changes.
The precompiled preamble is the initial region of the source file that includes only preprocessor tokens and comments.
It is precompiled and then reused to speed up subsequent reparsing.

A precompiled preamble can be serialized to disk, similar to precompiled headers.

## `-Xclang -fno-pch-timestamp`

By default, the generated PCH file includes the modified time of the headers it depends on.
When using the PCH file, PCH validation checks whether the included headers have changed based on their sizes and modified time.

```sh
clang -c b.hh -o b.hh.pch
clang -c -include-pch b.hh.pch a.cc
touch b.hh
clang -c -include-pch b.hh.pch a.cc  # error: file '/tmp/d/b.hh' has been modified since the precompiled header 'b.hh.pch' was built: mtime changed (was 1689570104, now 1689576446)
```

This timestamp safeguard, however, renders builds non-reproducible and can cause inconvenience for distributed build systems.

The cc1 option `-fno-pch-timestamp` can be used to disable timestamp for PCH generation and bypass the timestamp validation.

For distributed build systems, precompiled headers have another major issue.
They are usually much larger than the source they are built from. The network traffic can offset some advantages.
