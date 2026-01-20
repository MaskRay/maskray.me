---
layout: post
title: Compiler driver and cross compilation
author: MaskRay
tags: [gcc,llvm]
---

Updated 2023-03.

## Compiler driver

The `gcc` program is a compiler driver. It invokes other programs to do the work of compiling (cc1, cc1plus), assembling (GNU as), and linking (collect2).
The behavior is controlled by spec strings, which are provided by a plain-text [spec file](https://gcc.gnu.org/onlinedocs/gcc/Spec-Files.html).

<!-- more -->

You can run `gcc -dumpspecs` to dump the built-in spec file. It is complex but the main idea is construction of cc1/assembler/linker command lines.
Note: the interaction with the assembler/the linker should be clear from the output.

The `g++` program is another compiler driver. It uses `-x c++` by default and additionally links against the C++ library. The two programs are otherwise equivalent.

You can specify `-specs=` to override built-in directives. Here is an spec file derived from musl-gcc:

```text
*srcdir:
/tmp/musl

*prefix:
/tmp/musl/Debug

%rename cpp_options old_cpp_options

*cpp_options:
-nostdinc -isystem %(srcdir)/arch/x86_64 -isystem %(srcdir)/arch/generic -isystem %(srcdir)/include -isystem %(prefix)/obj/include -isystem include%s %(old_cpp_options)

*cc1:
%(cc1_cpu) -isystem %(srcdir)/arch/x86_64 -isystem %(srcdir)/arch/generic -isystem %(srcdir)/include -isystem %(prefix)/obj/include -nostdinc -isystem include%s

*link_libgcc:
-L%(prefix)/lib -L .%s

*libgcc:
libgcc.a%s %:if-exists(libgcc_eh.a%s)

*startfile:
%{!shared: %(prefix)/lib/Scrt1.o} %(prefix)/lib/crti.o crtbeginS.o%s

*endfile:
crtendS.o%s %(prefix)/lib/crtn.o

*link:
-dynamic-linker %(prefix)/lib/libc.so -nostdlib %{shared:-shared} %{static:-static} %{rdynamic:-export-dynamic}
```

This makes it easy to try out musl on a glibc-based system.

While a spec file can control some behaviors of `gcc`, many behaviors (target preferences) are guarded by macros are configured at build time.
It is quite common for toolchain developers to experiment with different configure options.

The Clang driver is similar to the `gcc` program in concepts but does more things.
You can specify `clang --target=aarch64-linux-gnu` to get aarch64-linux-gnu defaults.
Here are some examples:

* `clang++ --target=aarch64-linux-gnu -fno-pic -no-pie -fuse-ld=lld -Wl,--dynamic-linker=/usr/aarch64-linux-gnu/lib/ld-linux-aarch64.so.1 -Wl,-rpath=/usr/aarch64-linux-gnu/lib a.cc -o a`
* `clang++ --target=s390x-linux-gnu -fno-pic -no-pie -fuse-ld=lld -Wl,--dynamic-linker=/usr/s390x-linux-gnu/lib/ld64.so.1 -Wl,-rpath=/usr/s390x-linux-gnu/lib a.cc -o a`

The specified other options are translated by the driver into cc1 options.
In many cases you can observe the differences between two targets by comparing their cc1 output.
This design makes testing easy. If a feature passes on an x86_64-linux-gnu machine, it is highly likely it will also pass on another x86_64-linux-gnu machine, and should pass on other architectures or other OSes if the varying parts are controlled.
It is recommended to test features with cc1 options and place the target-specific behavior tests under `clang/test/Driver/`.

```cpp
static CodeGenOptions::FramePointerKind
getFramePointerKind(const ArgList &Args, const llvm::Triple &Triple) {
  Arg *A = Args.getLastArg(options::OPT_fomit_frame_pointer,
                           options::OPT_fno_omit_frame_pointer);
  bool OmitFP = A && A->getOption().matches(options::OPT_fomit_frame_pointer);
  bool NoOmitFP =
      A && A->getOption().matches(options::OPT_fno_omit_frame_pointer);
  bool OmitLeafFP = Args.hasFlag(options::OPT_momit_leaf_frame_pointer,
                                 options::OPT_mno_omit_leaf_frame_pointer,
                                 Triple.isAArch64() || Triple.isPS4CPU());
  if (NoOmitFP || mustUseNonLeafFramePointerForTarget(Triple) ||
      (!OmitFP && useFramePointerForTargetByDefault(Args, Triple))) {
    if (OmitLeafFP)
      return CodeGenOptions::FramePointerKind::NonLeaf;
    return CodeGenOptions::FramePointerKind::All;
  }
  return CodeGenOptions::FramePointerKind::None;
}

  CodeGenOptions::FramePointerKind FPKeepKind = getFramePointerKind(Args, RawTriple);
  const char *FPKeepKindStr = nullptr; 6 refs
  switch (FPKeepKind) {
  case CodeGenOptions::FramePointerKind::None:
    FPKeepKindStr = "-mframe-pointer=none";
    break;
  case CodeGenOptions::FramePointerKind::NonLeaf:
    FPKeepKindStr = "-mframe-pointer=non-leaf";
    break;
  case CodeGenOptions::FramePointerKind::All:
    FPKeepKindStr = "-mframe-pointer=all";
    break;
  }
  CmdArgs.push_back(FPKeepKindStr);
```

### Input kind and output kind

The driver recognizes the file name suffix to determine the compilation pipeline.

* `*.c`: C source code which must be preprocessed
* `*.h`: C header file to precompile
* `*.i`: C source code which should not be preprocessed
* `*.cc *.cpp`: C++ source code which must be preprocessed
* `*.hh *.hpp`: C++ header file to precompile
* `*.ii`: C++ source code which should not be preprocessed
* ...
* other: object file to be fed straight into linking

`gcc a.c` performs preprocessing/analysis/compiling/assembly generation/assembling/linking.
`gcc a.i` skips preprocessing.
`g++ a.cc b.cc` performs every phase before linking for each input file and does a link on all object files.

Some options can cause the driver/compiler to dispatch/do less work.
The most common ones are:

* `-E`: preprocess
* `-fsyntax-only`/clang cc1 `-emit-ast`: semantic analysis
* `-S`: compile, emit assembly
* `-c`: compile, emit object file
* default: link

Clang has an integrated assembler which is enabled by default for most cases.
When it is enabled, `clang -c` and `clang -S` just choose the different streamers (assembly vs object file).
`clang -S -fno-integrated-as` may behave differently because certain features may be integrated assembler only, or only supported by very new GNU as.
I added `-fbinutils-version=` to give users a choice not to worry about old GNU as/ld.

GCC does not have an integrated assembler. `-c` causes GCC to additionally feed the assembly to GNU as.

### Debugging

`-v` and `-###` can print the command lines. `-###` skips execution.

```
% /tmp/RelA/bin/clang a.c '-###'
clang version 13.0.0
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /tmp/RelA/bin
 "/tmp/RelA/bin/clang-13" "-cc1" "-triple" "x86_64-unknown-linux-gnu" "-emit-obj" "-mrelax-all" "--mrelax-relocations" "-disable-free" "-main-file-name" "a.c" "-mrelocation-model" "static" "-mframe-pointer=all" "-fmath-errno" "-fno-rounding-math" "-mconstructor-aliases" "-munwind-tables" "-target-cpu" "x86-64" "-tune-cpu" "generic" "-debugger-tuning=gdb" "-fcoverage-compilation-dir=/tmp/c" "-resource-dir" "/tmp/RelA/lib/clang/13.0.0" "-internal-isystem" "/usr/local/include" "-internal-isystem" "/usr/lib/gcc/x86_64-linux-gnu/10/../../../../x86_64-linux-gnu/include" "-internal-isystem" "/tmp/RelA/lib/clang/13.0.0/include" "-internal-externc-isystem" "/usr/include/x86_64-linux-gnu" "-internal-externc-isystem" "/include" "-internal-externc-isystem" "/usr/include" "-fdebug-compilation-dir=/tmp/c" "-ferror-limit" "19" "-fgnuc-version=4.2.1" "-fcolor-diagnostics" "-faddrsig" "-D__GCC_HAVE_DWARF2_CFI_ASM=1" "-o" "/tmp/a-04ff16.o" "-x" "c" "a.c"
 "/usr/local/bin/ld" "--eh-frame-hdr" "-m" "elf_x86_64" "-dynamic-linker" "/lib64/ld-linux-x86-64.so.2" "-o" "a.out" "/lib/x86_64-linux-gnu/crt1.o" "/lib/x86_64-linux-gnu/crti.o" "/usr/lib/gcc/x86_64-linux-gnu/10/crtbegin.o" "-L/usr/lib/gcc/x86_64-linux-gnu/10" "-L/usr/lib/gcc/x86_64-linux-gnu/10/../../../../lib64" "-L/lib/x86_64-linux-gnu" "-L/lib/../lib64" "-L/usr/lib/x86_64-linux-gnu" "-L/usr/lib/../lib64" "-L/usr/lib/gcc/x86_64-linux-gnu/10/../../.." "-L/tmp/RelA/bin/../lib" "-L/lib" "-L/usr/lib" "/tmp/a-04ff16.o" "-lgcc" "--as-needed" "-lgcc_s" "--no-as-needed" "-lc" "-lgcc" "--as-needed" "-lgcc_s" "--no-as-needed" "/usr/lib/gcc/x86_64-linux-gnu/10/crtend.o" "/lib/x86_64-linux-gnu/crtn.o"
```

`-H` can dump the include hierarchy.

```text
% gcc -H a.cc
. /usr/include/c++/10/iostream
.. /usr/include/x86_64-linux-gnu/c++/10/bits/c++config.h
... /usr/include/x86_64-linux-gnu/c++/10/bits/os_defines.h
.... /usr/include/features.h
..... /usr/include/x86_64-linux-gnu/sys/cdefs.h
...... /usr/include/x86_64-linux-gnu/bits/wordsize.h
...... /usr/include/x86_64-linux-gnu/bits/long-double.h
..... /usr/include/x86_64-linux-gnu/gnu/stubs.h
...... /usr/include/x86_64-linux-gnu/gnu/stubs-64.h
... /usr/include/x86_64-linux-gnu/c++/10/bits/cpu_defines.h
.. /usr/include/c++/10/ostream
```

## Compile modes

* `-fno-pic/-fno-PIC/-fno-pie/-fno-PIE` are identical. The use the traditional no-PIC mode which can only be linked as a position dependent executable (`ld -no-pie`).
* `-fpic` enables PIC mode which can be linked as either an executable (`-pie` or `-no-pie`) or a shared object (`-shared`).
* `-fpie` was introduced to GCC in 2003. The object file must be linked to an executable (`-pie` or `-no-pie`).

`-fpie` enables some optimizations which are unavailable for `-fpic`.
In GCC, `-fpie` is actually very similar to `-fpic -fno-semantic-interposition` plus TLS optimization.

The lower/upper case distinction `-fpic`/`-fPIC` is IMO a bad design.
Very few legacy architectures need such distinction: ppc32, sparc.

## Link modes

GNU ld supports 4 modes: `-r`, `-no-pie`, `-pie`, and `-shared`. GCC supports 4 driver options of the same names.

To add to the number of available link modes, GCC supports `-static` which uses ld's `-no-pie` mode but performs a static link.
Additionally, GCC supports `-static-pie`, which uses ld's `-pie` mode but performs a static link. The output is a static PIE.

## Language modes

### C

* GCC 5 defaults to `-std=gnu11` (`__cplusplus == 201112`).
* GCC 8 defaults to `-std=gnu17` (`__cplusplus == 201710L`).
* Clang 3.6 defaults to `-std=gnu11` (`__cplusplus == 201112L`).
* Clang 11 defaults to `-std=gnu17` (`__cplusplus == 201710L`).

### C++

* GCC 6 defaults to `-std=gnu++14` (`__cplusplus == 201402L`).
* GCC 11 defaults to `-std=gnu++17` (`__cplusplus == 201703L`).
* Clang 6 defaults to `-std=gnu++14`.

## Search paths

In GCC, cc1/cc1plus has default search paths for `#include` (C++ standard library headers and system headers), start files and libraries, and subprograms.

Internally, there are three search path lists, computed from the three global variables.

* `exec_prefixes`
* `startfile_prefixes`
* `include_prefixes`

You can specify `-B $prefix` (or `--prefix=$prefix`) to add an entry to all of the three lists.
If `$prefix` is a directory, `$prefix/include` will be prepended to `#include <...> search starts here:`. `$prefix/$triple/$version/$file`, (if `--enable-multi-arch`) `$prefix/$multiarch/$file`, `$prefix$file` will be used to searched for libraries (like a `-L`) and subprograms.
If `$prefix` is not a directory, it is still useful, e.g. `-B /tmp/x86_64-linux-gnu-` means GCC may pick `/tmp/x86_64-linux-gnu-as` for as and `/tmp/x86_64-linux-gnu-ld` for ld.

In Clang,

* if `$prefix` is a directory, search `$prefix/$file` for executables, libraries, and data files. Before 12.0.0, Clang also searched `$prefix/$triple-$file`.
* If `$prefix` is a file, search `$prefix$file` for executables, libraries, and data files.

`-nostdinc` drops default include paths.
`-nostdlib` drops default library paths.

For programs like `ld`, Clang calls `GetProgramPath` to get the path.
```cpp
string GetProgramPath(name) {
  for (b : OPT_B) {
    cand = is_directory(b) ? join(b, name) : b + name;
    if (is_executable(cand))
      return cand;
  }
  for (TargetSpecificExecutable : TargetSpecificExecutables) { // e.g. aarch64-linux-gnu-$name, $name
    for (auto dir : TC.getProgramPaths()) {  // -ccc-install-dir, /usr/lib/gcc-cross/aarch64-linux-gnu/13/../../../../aarch64-linux-gnu/bin
      cand = join(b, name);
      if (is_executable(cand))
        return cand;
    }
    for (auto dir : split(getenv("PATH"))) {
      cand = join(b, name);
      if (is_executable(cand))
        return cand;
    }
  }
  return name;
}
```

`COMPILER_PATH=/tmp/c clang -B/tmp --print-search-dirs` dumps `ResourceDir`, `getProgramPaths()`, and `getFilePaths()`.

## Lookup order of include paths

All of `-I dir`, `-iquote dir`, `-isystem dir`, `-idirafter dir` can add the directory dir to the list of directories to be searched for `#include` directives.
If `dir` begins with `=` or `$SYSROOT`, then the prefix is replaced by the sysroot prefix (see `--sysroot` and `-isysroot`).

This search order is documented at <https://gcc.gnu.org/onlinedocs/cpp/Invocation.html>.
Use `-fsyntax-only -v` to get the include paths.

```text
% g++ -fsyntax-only a.cc -v
...
#include "..." search starts here:
A
B
#include <...> search starts here:
C
D
```

For `#include <file>`, the search order is: C, D.
For `#include "file"`, the search order is: the directory of the current file, A, B, C, D.

In Clang, the algorithm of include paths is:

* Concat `-isystem` and built-in `-isystem` to get system_dirs. Delete `-iquote -I -idirafter` elements that appear in `system_dirs`.
* The order is: `unique(-iquote) + unique(-I + -isystem + built-in -isystem + -idirafter)`.
* For `#include <...>`, the `unique(-iquote)` part is ignored.

This is quite similar to but not identical to GCC's. I think with GCC, in some cases a directory may repeat in both the `#include "..."` and `#include <...>` lists.

## Default search paths

### Upstream GCC

At configure time, `--enable-multi-arch` is the default for native builds if glibc supports it.

Let's look at a multiarch build.

```text
# Configured with --disable-bootstrap --enable-languages=c,c++ --with-multilib-list=m64,m32
% /tmp/opt/gcc-debug/bin/gcc --print-multiarch
x86_64-linux-gnu
% /tmp/opt/gcc-debug/bin/gcc --print-multi-os-directory
../lib64
% /tmp/opt/gcc-debug/bin/gcc --print-multi-lib
.;
32;@m32
% /tmp/opt/gcc-debug/bin/gcc -fsyntax-only a.cc -v
...
#include "..." search starts here:
#include <...> search starts here:
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1/x86_64-pc-linux-gnu
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1/backward
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/include
 /usr/local/include/x86_64-linux-gnu         # affected by sysroot, multiarch, usually nonexistent
 /usr/local/include                          # affected by sysroot
 /tmp/opt/gcc-debug/include
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/include-fixed
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../x86_64-pc-linux-gnu/include
 /usr/include/x86_64-linux-gnu               # affected by sysroot, multiarch
 /usr/include                                # affected by sysroot
...
% /tmp/opt/gcc-debug/bin/g++ a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../lib64
 -L/lib/x86_64-linux-gnu                     # affected by sysroot, multiarch
 -L/lib/../lib64                             # affected by sysroot, multi-os
 -L/usr/lib/x86_64-linux-gnu                 # affected by sysroot, multiarch
 -L/usr/lib/../lib64                         # affected by sysroot, multi-os
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../..
 # -L$sysroot/lib if sysroot is not "" or "/"
 # -L$sysroot/usr/lib if sysroot is not "" or "/"
...
```

Some paths are relative to the GCC installation:

* The first three search paths (`include/c++`) are for libstdc++. Debian patched native gcc has altered the search paths.
* `/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/include` refers to GCC's private headers.

The others are relative to sysroot. I have annotated the lines in the output.

multiarch and multi-os can affect include and library paths. See <https://wiki.debian.org/Multiarch/Tuples> for a list of triples.
multi-os leads to the `../lib64` path components.

Due to multiarch, `$sysroot/usr/local/include` and `$sysroot/usr/local/include` are preceded by their `$multiarch` counterparts.
This is the main point: different architectures have separate include directories while they can share some common directories.
However, the library directories cannot really be shared and the common directories just cause issues.
The problem in practice is that Debian has local multiarch patches which do things differently - the differences seem entirely unnecessary to me.
Read on.

Let's see the output of a vanilla `--disable-multi-arch` native compiler.
The sysroot directory should be clear from the output.

```text
# Configured with --disable-multi-arch.
% many=/tmp/glibc-many
% $many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-g++ --print-multiarch

% $many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-g++ --print-multi-os-directory
../lib64
% $many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-g++ --print-multi-lib
.;
32;@m32
x32;@mx32
% $many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-g++ -fsyntax-only a.cc -v
...
#include "..." search starts here:
#include <...> search starts here:
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/include/c++/10.2.1
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/include/c++/10.2.1/x86_64-glibc-linux-gnu
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/include/c++/10.2.1/backward
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/include
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/usr/local/include
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/include-fixed
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/include
 /tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/usr/include
...
% $many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-g++ a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/lib/../lib64
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/lib/../lib64
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/usr/lib/../lib64
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/lib/gcc/x86_64-glibc-linux-gnu/10.2.1/../../../../x86_64-glibc-linux-gnu/lib
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/lib
 -L/tmp/glibc-many/install/compilers/x86_64-linux-gnu/sysroot/usr/lib
```

Let's see the output of a vanilla `--disable-multi-arch` cross compiler.

```text
% many=/tmp/glibc-many
% $many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ --print-multiarch

% $many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ --print-multi-os-directory
../lib64
% $many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ --print-multi-lib
.;
% $many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ -fsyntax-only a.cc -v
...
#include "..." search starts here:
#include <...> search starts here:
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/include/c++/10.2.1
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/include/c++/10.2.1/aarch64-glibc-linux-gnu
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/include/c++/10.2.1/backward
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/include
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/usr/local/include
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/include-fixed
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/include
 /tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/usr/include
...
% $many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/lib/../lib64
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/lib/../lib64
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/usr/lib/../lib64
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/lib/gcc/aarch64-glibc-linux-gnu/10.2.1/../../../../aarch64-glibc-linux-gnu/lib
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/lib
 -L/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot/usr/lib
...
```

### Upstream GCC multilib

multilib allows an x86-64 targeted compiler to use `-m32`/`-mx32` and an i386 targeted compiler to use `-m64`.

```text
% /tmp/opt/gcc-debug/bin/g++ -m32 -fsyntax-only a.cc -v
...
#include "..." search starts here:
#include <...> search starts here:
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1/x86_64-pc-linux-gnu/32
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../include/c++/11.0.1/backward
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/include
 /usr/local/include/i386-linux-gnu           # affected by sysroot, multiarch, usually nonexistent
 /usr/local/include                          # affected by sysroot
 /tmp/opt/gcc-debug/include
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/include-fixed
 /tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../x86_64-pc-linux-gnu/include
 /usr/include/i386-linux-gnu                 # affected by sysroot, multiarch
 /usr/include                                # affected by sysroot
...
% /tmp/opt/gcc-debug/bin/g++ -m32 a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/32
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../../../lib32
 -L/lib/i386-linux-gnu
 -L/lib/../lib32
 -L/usr/lib/i386-linux-gnu
 -L/usr/lib/../lib32
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1           # may be harmful
 -L/tmp/opt/gcc-debug/lib/gcc/x86_64-pc-linux-gnu/11.0.1/../../..  # may be harmful
 -L/lib/i386-linux-gnu
 -L/usr/lib/i386-linux-gnu
...
```

### Debian

Debian likes to have different opinions. (ⓛ ω ⓛ *)
Debian uses multiarch but its native compiler has different search paths.

Because `MULTILIB_OSDIRNAMES` is patched, (with `gcc-multilib-multiarch.diff` or `gcc-multiarch.diff`), the upstream `../lib64` becomes `../lib`.
This is actually nice.

```text
% g++ --print-multiarch
x86_64-linux-gnu
% g++ --print-multi-os-directory
../lib
% g++ --print-multi-lib
.;
32;@m32
x32;@mx32
% g++ -fsyntax-only a.cc -v
...
ignoring duplicate directory "/usr/include/x86_64-linux-gnu/c++/10"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/10/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/10/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/include/c++/10
 /usr/include/x86_64-linux-gnu/c++/10        # weird path
 /usr/include/c++/10/backward
 /usr/lib/gcc/x86_64-linux-gnu/10/include
 /usr/local/include/x86_64-linux-gnu         # affected by sysroot, multiarch, usually nonexistent
 /usr/local/include                          # affected by sysroot
 /usr/include/x86_64-linux-gnu               # affected by sysroot, multiarch
 /usr/include                                # affected by sysroot
...
% g++ a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/usr/lib/gcc/x86_64-linux-gnu/10
 -L/usr/lib/gcc/x86_64-linux-gnu/10/../../../x86_64-linux-gnu
 -L/usr/lib/gcc/x86_64-linux-gnu/10/../../../../lib
 -L/lib/x86_64-linux-gnu                     # affected by sysroot, multiarch
 -L/lib/../lib                               # affected by sysroot, multi-os
 -L/usr/lib/x86_64-linux-gnu                 # affected by sysroot, multiarch
 -L/usr/lib/../lib                           # affected by sysroot, multi-os
 -L/usr/lib/gcc/x86_64-linux-gnu/10/../../..
...
```

Cross compiler. The libstdc++ search paths are not altered.

```text
% aarch64-linux-gnu-g++ --print-multiarch
aarch64-linux-gnu
% aarch64-linux-gnu-g++ --print-multi-os-directory
../lib
% aarch64-linux-gnu-g++ --print-multi-lib
.;
% aarch64linux-gnu-g++ -fsyntax-only a.cc -v
...
ignoring nonexistent directory "/usr/lib/gcc-cross/aarch64-linux-gnu/10/include-fixed"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10
 /usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10/aarch64-linux-gnu
 /usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10/backward
 /usr/lib/gcc-cross/aarch64-linux-gnu/10/include
 /usr/local/include/aarch64-linux-gnu        # affected by sysroot, usually nonexistent
 /usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include   # multiarch
 /usr/include/aarch64-linux-gnu              # affected by sysroot, usually nonexistent
 /usr/include                                # affected by sysroot
...
% aarch64-linux-gnu-g++ a.cc '-###' |& sed -E 's/ "?-[iIL]/\n&/g'
...
 -L/usr/lib/gcc-cross/aarch64-linux-gnu/10
 -L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib/../lib
 -L/lib/aarch64-linux-gnu                    # affected by sysroot, multiarch
 -L/lib/../lib                               # affected by sysroot, multi-os
 -L/usr/lib/aarch64-linux-gnu                # affected by sysroot, multiarch
 -L/usr/lib/../lib                           # affected by sysroot, multi-os
 -L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib
...
```

### Arch Linux

`aarch64-linux-gnu-gcc --print-sysroot` prints `/usr/aarch64-linux-gnu`.
Compilers for different architectures have disjoint include paths.
This can cause some redundancy.

### Remarks

I hope from various dumpings you have some idea what multiarch/multilib/multi-os are.

multilib is for integrating `-m32/-mx32` functionality into an x86-64 targeted compiler, and situations similar to that.
multilib appends `/32` `/x32` suffixes which are convenient if the installation does not want to introduce separate architecture specific directories.
multilib assumes `$sysroot/usr/local/include` and `$sysroot/usr/include` can be used by 32-bit and 64-bit variants, so no additional include path is added.
However, if a library does want different header files, there is no way but to resort to multiarch.
This issue makes it less useful.

The bad parts:

* Upstream gcc uses `../lib64` for `MULTILIB_OSDIRNAMES` (`gcc --print-multi-os-directory`)
* Debian native gcc has a weird libstdc++ include path for its multiarch implementation. Fortunately this is not done for its cross compilers.
* multilib is useless if the user wants different architecture-specific include/library paths under sysroot. multiarch has to be used together.
* The multiarch triple does not necessarily match the original triple. You may get mixed `lib/gcc/x86_64-pc-linux-gnu` and `/usr/lib/x86_64-linux-gnu`.

I think both multilib and multi-os are very broken. multiarch is useful but Debian does it wrongly for its native GCC.

### Clang

Unlike GCC, in Clang, the include paths are computed by the driver.

You can specify `--target=` to ask for cross compiling. Clang will happily detect system GCC installations and add appropriate include and library paths.
Note: Clang has its own resource directory. It should not use GCC's private headers.
(There is a legacy spelling `-target `. Please don't use it.)

Note that Clang before 13.0.0 incorrectly assumes that cross gcc follows the Debian native gcc behavior.

```text
% /tmp/Stable/bin/clang++ --target=aarch64-linux-gnu '-###' a.cc |& sed -E 's/ "?-[iIL]/\n&/g'
...
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/c++/10"
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/aarch64-linux-gnu/c++/10"  # nonexistent
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/aarch64-linux-gnu/c++/10"  # nonexistent, strangely duplicated
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/c++/10/backward"
 "-internal-isystem" "/usr/local/include"
 "-internal-isystem" "/tmp/Stable/lib/clang/13.0.0/include"
 "-internal-externc-isystem" "/include"
 "-internal-externc-isystem" "/usr/include"
...
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../aarch64-linux-gnu"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../lib64"
 "-L/lib/aarch64-linux-gnu"
 "-L/lib/../lib64"
 "-L/usr/lib/aarch64-linux-gnu"
 "-L/usr/lib/../lib64"
 "-L/usr/lib/aarch64-linux-gnu/../../lib64"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../.."
 "-L/tmp/Stable/bin/../lib"
 "-L/lib"
 "-L/usr/lib"
```

Note that `"-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/aarch64-linux-gnu/c++/10"` refers to a nonexistent directory, so compiling a file with C++ headers will lead to such an error:
```text
/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../include/c++/10/iostream:38:10: fatal error: 'bits/c++config.h' file not found
#include <bits/c++config.h>
         ^~~~~~~~~~~~~~~~~~
```

I have fixed the problem in 13.0.0 and cleaned up unneeded search paths.
My guideline is to make Clang able to pick up both vanilla and Debian GCC's libstdc++/start files.

```text
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10"
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10/aarch64-linux-gnu"
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include/c++/10/backward"
 "-internal-isystem" "/tmp/RelA/lib/clang/13.0.0/include"
 "-internal-isystem" "/usr/local/include"
 "-internal-isystem" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/include"
 "-internal-externc-isystem" "/include"
 "-internal-externc-isystem" "/usr/include"
...
 "/usr/bin/aarch64-linux-gnu-ld" "-EL" "--eh-frame-hdr" "-m" "aarch64linux" "-dynamic-linker" "/lib/ld-linux-aarch64.so.1" "-o" "a.out" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib/crt1.o" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib/crti.o" "/usr/lib/gcc-cross/aarch64-linux-gnu/10/crtbegin.o"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10"
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../lib64"  # present if the 'lib' path starts with sysroot
 "-L/lib/aarch64-linux-gnu"                  # affected by sysroot, multiarch
 "-L/lib/../lib64"                           # affected by sysroot, multi-os
 "-L/usr/lib/aarch64-linux-gnu"              # affected by sysroot, multiarch
 "-L/usr/lib/../lib64"                       # affected by sysroot, multi-os
 "-L/usr/lib/gcc-cross/aarch64-linux-gnu/10/../../../../aarch64-linux-gnu/lib"
 "-L/tmp/RelA/bin/../lib"                    # So that -lc++ -lc++abi can pick up libc++/libc++abi built together with clang
 "-L/lib"                                    # affected by sysroot, always added (unlike gcc)
 "-L/usr/lib"                                # affected by sysroot, always added (unlike gcc)
```

In Clang, `--sysroot=` additionally changes where Clang detects GCC installations (`$sysroot` and `$sysroot/usr`).
So the include/library paths for libstdc++/crtbegin/crtend will change as well.
You may specify `--gcc-toolchain=` to override the prefix used to detect GCC installations.

```cpp
if (OPT_gcc_install_dir_EQ)
  return OPT_gcc_install_dir_EQ;

if (OPT_gcc_triple)
  candidate_gcc_triples = {OPT_gcc_triple};
else
  candidate_gcc_triples = collectCandidateTriples();
if (OPT_gcc_toolchain)
  prefixes = {OPT_gcc_toolchain};
else
  prefixes = {OPT_sysroot/usr, OPT_sysroot};
for (prefix : prefixes)
  if "$prefix/lib/gcc" exists // also tries $prefix/lib/gcc-cross
    for (triple : candidate_gcc_triples)
      if "$prefix/lib/gcc/$triple" exists
        return "$prefix/lib/gcc/$triple/$version"; // pick the largest version
```

Before Clang 13, `-B $prefix` causes `$prefix` to be detected as well. I dropped the behavior in <https://reviews.llvm.org/D97993>.

The `--sysroot=` behavior is very convenient.
Say, you have a Debian rootfs `/tmp/debian` (with `{,usr/}lib/gcc-cross/powerpc64le-linux-gnu`), or a prebuilt GCC with glibc system headers, you can specify `clang++ --sysroot=/tmp/debian --target=powerpc64le-linux-gnu`.
It should just work.

### Clang with libc++ and libunwind

```sh
cmake -GNinja -Hllvm -B/tmp/out/custom1 -DCMAKE_BUILD_TYPE=Release -DCMAKE_CROSSCOMPILING=on -DCMAKE_INSTALL_PREFIX=/tmp/opt/aarch64 -DLLVM_TARGETS_TO_BUILD=AArch64 -DLLVM_DEFAULT_TARGET_TRIPLE=aarch64-unknown-linux-gnu -DLLVM_TARGET_ARCH=AArch64 -DLLVM_ENABLE_PROJECTS='clang;lld' -DLLVM_ENABLE_RUNTIMES='compiler-rt;libcxx;libcxxabi;libunwind'
ninja -C /tmp/out/custom1 lld cxx cxxabi unwind builtins
```

Note: it is important to specify `aarch64-unknown-linux-gnu` instead of `aarch64-linux-gnu` (<https://reviews.llvm.org/D110663> will make `aarch64-linux-gnu` work).
You may specify `-DCLANG_DEFAULT_CXX_STDLIB=libc++ -DCLANG_DEFAULT_UNWINDLIB=libunwind -DCLANG_DEFAULT_RTLIB=compiler-rt` to default to libc++, libunwind, and compiler-rt.

```sh
/tmp/out/custom1/bin/clang++ -c -stdlib=libc++ a.cc
/tmp/out/custom1/bin/clang++ -fuse-ld=lld --dyld-prefix=/usr/aarch64-linux-gnu --unwindlib=libunwind --rtlib=compiler-rt -nostdlib++ -pthread -static-libgcc a.o -Wl,--push-state,-Bstatic,-lc++,-lc++abi,--pop-state,-rpath=/usr/aarch64-linux-gnu/lib -ldl -o a
```

I have installed `g++-aarch64-linux-gnu` on my Debian machine. `/usr/aarch64-linux-gnu/lib` has glibc shared objects and `libstdc++.so.6`.

## Link phase

If you link a program with a compiler driver (clang/gcc) in a standard way (not `-nostdlib`), the following components are usually on the linker command line.

* crt1.o (glibc/musl): `-no-pie`/`-pie`/`-static-pie`
  + crt1.o: `-no-pie`
  + Scrt1.o: `-pie`, `-shared`
  + rcrt1.o: `-static-pie`
  + gcrt1.o:
* crti.o (glibc/musl)
* crtbegin.o
  + crtbegin.o: `-no-pie`
  + crtbeginS.o: `-pie`, `-shared`
  + crtbeginT.o: `-static-pie`
* user input
* -lstdc++/-lc++/-lm
* libc/built-in library/libunwind (Some combination of -lc -lgcc_s -lgcc -lgcc_eh)
* crtn.o (glibc/musl)
* crtend.o
  + crtend.o: `-no-pie`
  + crtendS.o: `-pie`, `-shared`
  + crtendT.o: `-static-pie`

`-nostartfiles` drops `*crt*.o` files. `-nodefaultlibs` drops default `-l*`.
`-nostdlib` combines `-nostartfiles` and `-nodefaultlibs`.

### crt1.o

This file is only used by executables.

In glibc, the file is `-r` linked from `csu/start.c csu/abi-note.c csu/init.c csu/static-reloc.c`.
It used to call `__libc_start_main` with arguments `main`, `__libc_csu_init`, `__libc_csu_fini` (defined by `libc_nonshared.a(elf-init.oS)`).
From BZ #23323 onwards, on most architectures, `start.S:_start` calls `__libc_main_start` with two zero arguments instead, and `__libc_csu_init` and `__libc_csu_fini` are moved into `csu/libc-start.c`.

In musl, this file calls `__libc_start_main` with `main`, `_init`, and `_fini`.

### crti.o/crtn.o/crtbegin.o/crtend.o

See [.init, .ctors, and .init\_array](/blog/2021-11-07-init-ctors-init-array).

GCC provide `crtbegin.o` and `crtend.o`.
llvm-project/compiler-rt provides `clang_rt.crtbegin.o` and `clang_rt.crtend.o`.

### Built-in and unwinding library

gcc always uses libgcc.
In Clang, on Linux targets, by default libgcc provides builtin functions (`--rtlib={platform,libgcc}`) and unwinding functions for Itanium C++ ABI exception handling (`--unwindlib={platform,libgcc}`).
There are three cases:

* `-static`, `-static-pie` or `-static-libgcc`: `-lgcc -lgcc_eh -lc -lgcc -lgcc_eh`
* C++ or `-shared-libgcc`: `-lgcc_s -lgcc -lc -lgcc_s -lgcc`
* other (C specific): `-lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed`

`libgcc.a` contains builtin functions and misc functions.
`libgcc_eh.a` contains unwinding functions for Itanium C++ ABI exception handling.
`libgcc_s.so.1` contains a part of `libgcc.a` and a complete set of unwinding functions.
`libgcc_s.so` is a linker script which includes `libgcc_s.so.1` and (sometimes) `libgcc.a`.

Static libgcc links against `libgcc.a` (builtin functions and misc functions) and `libgcc_eh.a` (unwinding).
Shared libgcc links against `libgcc_s.so` (linker script) and `libgcc.a`.
The C specific way is an optimization: because C does not have exceptions (however, `-fexceptions` can enable cleanup functions), the unwinding part of `libgcc_s.so` is often not needed. Annotating the library with `--as-needed` can avoid a `DT_NEEDED` entry in most cases.

So why do we have libgcc things on both sides of `-lc`?
Well, there is an unfortunate violation. Some built-in functions call `abort()` and have a dependency on libc.

If `--rtlib=compiler-rt` is specified, Clang will link against `libclang_rt.builtins-*.a`.

If `--unwindlib=libunwind` is specified, Clang will pass `-l:libunwind.a` or `-l:libunwind.so` to the linker. This option requires `--rtlib=compiler-rt`.

## Appendix

### glibc startup sequence

Below the control flows are flattened.

#### Dynamically linked executable

In rtld (ld.so):

* `sysdeps/x86_64/dl-machine.h:_start`
* `elf/rtld.c:_dl_start`
* `sysdeps/x86_64/dl-machine.h:_dl_start_user`
* `elf/dl-init.c:_dl_init` shared objects' (if `ELF_INITFINI` is defined) `DT_INIT` and `DT_INIT_ARRAY` are executed. Basically the reverse dependency order.
* Jump to the main executable `e_entry`

In the main executable (code linked `libc_nonshared.a`):

* `sysdeps/x86_64/start.S:_start`
* `csu/libc-start.c:__libc_start_main`, the `SHARED` branch
* (if `ELF_INITFINI` is defined) Run `DT_INIT`
* Run `DT_INIT_ARRAY`
* Run `main`
* Run `exit`
* `stdlib/exit.c:__run_exit_handlers`

```cpp
// a.cc -> a
#include <stdio.h>
__attribute__((constructor)) void init() { puts("init"); }
extern "C" void ctors() { puts("ctors"); }
asm(".pushsection .ctors,\"aw\"; .quad ctors; .popsection");
int main() {}

// b.cc -> b.so
#include <stdio.h>
__attribute__((constructor)) void init_b() { puts("init b"); }
extern "C" void ctors_b() { puts("ctors b"); }
asm(".pushsection .ctors,\"aw\"; .quad ctors_b; .popsection");

// c.cc -> c.so
#include <stdio.h>
__attribute__((constructor)) void init_c() { puts("init c"); }
extern "C" void ctors_c() { puts("ctors c"); }
asm(".pushsection .ctors,\"aw\"; .quad ctors_c; .popsection");
```

On a system where `ELF_INITFINI` is defined and crtbegin.o's `_init` fragment calls `.ctors` constructors:

```text
% ./a
ctors c
init c
ctors b
init b
ctors
init
```

#### Statically linked executable

In the main executable:

* `sysdeps/x86_64/start.S:_start`
* `csu/libc-start.c:__libc_start_main`, the `!SHARED` branch
* `_dl_relocate_static_pie`
* `ARCH_SETUP_IREL`
* `ARCH_SETUP_TLS`
* `csu/libc-start.c:call_init`
  + Run `[__preinit_array_start, __preinit_array_end)`
  + (if `ELF_INITFINI` is defined) Run `_init`
  + Run `[__init_array_start, __init_array_end)`
* Run `main`
* Run `exit`

### musl startup sequence

For a dynamically linked executable, the rtld process:

* `arch/x86_64/crt_arch.h:_dlstart`
* `ldso/dlstart.c:_dlstart_c`
* `ldso/dynlink.c:__dls2` relocate rtld
* `ldso/dynlink.c:__dls2b` setup early thread pointer
* `ldso/dynlink.c:__dls3`
* Jump to the main executable `e_entry`

In the main executable:

* `arch/x86_64/crt_arch.h:_start`
* `crt/crt1.c:_start_c`
* `src/env/__libc_start_main.c:__libc_start_main`
* `__init_libc` initialize auxv/TLS/stack protector/etc
* `libc_start_main_stage2`
* `__libc_start_init`
* `exit(main(argc, argv, envp));`

`__libc_start_init` has different behaviors for dynamically and statically linked executables.
For a dynamically linked executable: it runs `DT_INIT` (unless `NO_LEGACY_INITFINI`) then `DT_INIT_ARRAY`.
Note: libc.so has a dummy `_init`.


```
for_each_path
  while (1) {
    for (pl = paths->plist; pl; pl = pl->next) {
      if (!skip_multi_dir)
        Try pl->prefix + "$machine/$version/" (e.g. "x86_64-pc-linux-gnu/10/")
      if (!skip_multi_dir && pl->require_machine_suffix == 2) { // as, ld, etc
        Try pl->prefix + "$machine/" (e.g. "x86_64-pc-linux-gnu/")
      }
      if (!skip_multi_dir && pl->require_machine_suffix == 0 && multiarch_dir) {
        Try pl->prefix + "$multiarch/" (e.g. "x86_64-linux-gnu/")
      }
      if (!pl->require_machine_suffix && !(pl->os_multilib ? skip_multi_os_dir : false)) {
        Try pl->prefix + (pl->os_multilib ? multi_os_dir : multi_dir)
      }
    }
    if (multi_dir == NULL && multi_os_dir == NULL)
      break;
    ...
  }
```

### Make variables

POSIX make specifies `CC, CFLAGS, LDFLAGS`. `CXXFLAGS` is a common one for C++.
bmake and GNU make have `CPPFLAGS` for preprocessing options.

In bmake, the default `LINK.cc` is `LINK.cc = ${CXX} ${CXXFLAGS} ${CPPFLAGS} ${LDFLAGS}`.
In GNU make, the default `LINK.cc` is `$(CXX) $(CXXFLAGS) $(CPPFLAGS) $(LDFLAGS) $(TARGET_ARCH)`.

## Misc

`-no-canonical-prefixes` instructs Clang to call `realpath` on the executable name and use the dereferenced absolute path for the `-cc1` command.
This path, either canonicalized by `realpath` or not, is used to derive the resource directory.
See <https://gcc.gnu.org/legacy-ml/gcc/2011-01/msg00429.html> for some background.

For most tests "-cc1" is sufficient to identify the command line, no need to specifically test the "clang" command, and `-no-canonical-prefixes` can be removed.

To test that certain options do not lead to errors or warnings, use:
```
RUN: %clang -fdriver-only -Werror... 2>&1 | count 0
```

## Environment variables

### `COMPILER_PATH`

As if `-B<prefix>` is specified.

### `LIBRARY_PATH`

If specified, a colon-separated directories passed to `-L` in a link action.

### `CC_LOG_DIAGNOSTICS`

```sh
% CC_LOG_DIAGNOSTICS=1 clang -c a.c
a.c:2:11: warning: non-void function does not return a value [-Wreturn-type]
int foo(){}
          ^
<dict>
  <key>main-file</key>
  <string>a.c</string>
  <key>diagnostics</key>
  <array>
...
```

### `CC_PRINT_HEADERS`, `CC_PRINT_HEADERS_FILTERING`

```sh
% CC_PRINT_HEADERS=1 clang a.c
/usr/include/stdio.h
/usr/include/x86_64-linux-gnu/bits/libc-header-start.h
...
% CC_PRINT_HEADERS=1 CC_PRINT_HEADERS_FILE=a.txt clang a.c
% cat a.txt
/usr/include/stdio.h
/usr/include/x86_64-linux-gnu/bits/libc-header-start.h
...
```

The output can be of a JSON format, but this format requires `CC_PRINT_HEADERS_FILTERING=only-direct-system`.
```sh
% CC_PRINT_HEADERS_FILTERING=only-direct-system CC_PRINT_HEADERS_FORMAT=json clang -c a.c
{"source":"/tmp/c/a.c","includes":["/usr/include/stdio.h"]}
```

### `CC_PRINT_PROC_STAT`

`-fprint-proc-stat` or `-fprint-proc-stat=<file>`
```sh
% CC_PRINT_PROC_STAT=1 clang -c a.c
clang: output=a.o, total=11.869 ms, user=3.956 ms, mem=37000 Kb
% rm -f a.stat && repeat 2 CC_PRINT_PROC_STAT=1 CC_PRINT_PROC_STAT_FILE=a.stat clang -c a.c
% cat a.stat
"clang","a.o",12372,4124,36900
"clang","a.o",11283,7522,36876
```

Each row contains command name, output file name, total time in milliseconds, user time in milliseconds, and maximum resident set size in KiB.

### `CCC_OVERRIDE_OPTIONS`

Apply a comma-separated list to the argument list.

'^': Add FOO as a new argument at the beginning of the command line.

'+': Add FOO as a new argument at the end of the command line.

'xOPTION': Removes all instances of the literal argument OPTION.

...

### `CLANG_NO_DEFAULT_CONFIG=1`

Disable loading of default Clang configuration files.

### `SOURCE_DATE_EPOCH`

If [`SOURCE_DATE_EPOCH`](https://reproducible-builds.org/docs/source-date-epoch/) is set, it specifies a UNIX timestamp to be used in replacement of the current date and time in the `__DATE__`, `__TIME__`, and `__TIMESTAMP__` macros.

### `FORCE_CLANG_DIAGNOSTICS_CRASH`

### `CPATH`, `C_INCLUDE_PATH`, `CPLUS_INCLUDE_PATH`, `OBJC_INCLUDE_PATH`, `OBJCPLUS_INCLUDE_PATH`


## Misc

### `-fno-canonical-system-headers`

In April 2012, GCC [shortened include paths in diagnostics](https://gcc.gnu.org/PR52974) by checking whether the `lrealpath` output is shorter (see `maybe_shorter_path`).
The change unintentionally affected paths in dependency files, which confused Bazel's logic for verifying all `#include`d header files are listed as inputs to the action.

In November 2012, [commit 5dc99c4](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=5dc99c46786e91f2892c4318ed36054302e0c964) added `-f[no-]canonical-system-headers` and `--enable-canonical-system-headers` to restore the previous behavior and fix the dependency file output. See https://gcc.gnu.org/pipermail/gcc-patches/2012-September/346990.html for the motivation.

As far as I know, `-fno-canonical-system-headers` is still a niche option primarily used by Bazel when it detects that the compiler supports the option.
For `compile_commands.json` generation, the user should use a Clang configuration of Bazel, not a GCC one.
If a GCC configuration is picked, there may be a `-fno-canonical-system-headers` option, but I don't think the use case justifies an ignored Clang driver option.

### Clang Driver

Noticeable functions to read.

```
BuildCompilation
  getToolchain
  HandleImmediateArgs
  BuildInputs
  BuildActions
    handleArguments
  BuildJobs
    BuildJobsForAction
      ToolChain::SelectTool
      Clang::ConstructJob
        Clang::RenderTargetOptions
        renderDebugOptions
    Report -Wunused-command-line-argument
```

In `BuilInputs`, an `options::LinkerInput` argument is treated as an input of type `TY_Object`.

`tools::gnutools::Linker::ConstructJob`

If the final phase is linking, `Driver.cpp` does `Args.ClaimAllArgs(options::OPT_CompileOnly_Group);`

If an option has the `NoXarchOption` flag, ClangDriver will report an error if the option is used after `-Xarch_*` (originally for universal macOS binary, reused by offloading purposes `-Xarch_host`/etc).
The error checking only applies to a small set of options (e.g. `-o`, `--target=`) and is not very useful for most options, but `NoXarchOption` was improperly named `DriverOption` (commit aabb0b11a3c1d8a6bb859db80400cffdcc9b336f) and lured some contributors to add `NoXarchOption` to options that should not have the flag.

`-ccc-print-phases` dumps actions (phases). `BuildJobsForAction` builds `JobAction` and selects `Tool`s.
Some `JobAction`s can be combined, e.g. `CompileJobAction/BackendJobAction/AssembleJobAction` can be combined by using just one tool.
