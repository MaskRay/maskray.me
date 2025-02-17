layout: post
title: Everything I know about GNU toolchain
author: MaskRay
tags: [binutils,gcc,glibc]
---

As mainly an LLVM person, I occasionally contribute to GNU toolchain projects.
This is sometimes for fun, sometimes for investigating why an (usually ancient) feature works in a particular way,
sometimes for pushing forward a toolchain feature with the mind of both communities,
or sometimes just for getting sense of how things work with mailing list+GNU make.

<!-- more -->

For a debug build, I normally place my build directory `out/debug` directly under the project root.

Dependency chain of a port: GCC needs binutils. Linux kernel needs GCC. glibc needs Linux kernel.

## binutils

* Repository: <https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git> (write access: `ssh://sourceware.org/git/binutils-gdb.git`)
* Repository for documentation: <https://sourceware.org/cgit/binutils-htdocs/>
* Wiki: <https://sourceware.org/binutils/wiki/>
* Mailing list: <https://sourceware.org/pipermail/binutils>
* Bugzilla: <https://sourceware.org/bugzilla/> (notifications: <https://lists.gnu.org/archive/html/bug-binutils/>)
* Main tools: as (`gas/`, GNU assembler), ld (`ld/`, GNU ld), gold (`gold/`, GNU gold)
* Branches: `binutils-2_35-branch`, `binutils-2_36-branch`

Target `all` builds targets `all-host` and `all-target`.
When running configure, by default most top-level directories `binutils gas gdb gdbserver ld libctf` are all enabled.
You can disable some components via `--disable-*`. `--enable-gold` is needed to enable gold.

```sh
mkdir -p out/debug; cd out/debug
../../configure --target=x86_64-linux-gnu --prefix=/tmp/opt/binutils-debug --disable-gdb --disable-gdbserver
```

For cross compiling, make sure your have `$target-{gcc,as,ld}`.

For many tools (binutils, gdb, ld), `--enable-targets=all` will build every supported architectures and binary formats.
However, one gas build can only support one architecture. ld has a default emulation and needs `-m` to support other architectures
(``aarch64 architecture of input file `a.o' is incompatible with i386:x86-64 output``).
Many tests are generic and can be run on many targets, but a `--enable-targets=all` build only tests its default target.

```sh
# binutils (binutils/*)
make -C out/debug all-binutils
# gas (gas/as-new)
make -C out/debug all-gas
# ld (ld/ld-new)
make -C out/debug all-ld

# Build all enabled tools.
make -C out/debug all
```

Build with Clang:
```sh
mkdir -p out/clang-debug; cd out/clang-debug
../../configure CC=~/Stable/bin/clang CXX=~/Stable/bin/clang++ CFLAGS='-O0 -g' CXXFLAGS='-O0 -g'
```

About security aspect, "don't run any of binutils as root" is sufficient advice (Alan Modra).

### Test

GNU Test Framework DejaGnu is based on Expect, which is in turn based on Tcl.

To run tests:
```sh
make -C out/debug check-binutils
# Find the result in (summary) out/debug/binutils/binutils.sum and (details) out/debug/binutils/binutils.log

make -C out/debug check-gas
# Find the result in (summary) out/debug/gas/testsuite/gas.sum and (details) out/debug/gas/testsuite/gas.log

make -C out/debug check-ld

# Test all enabled tools.
make -C out/debug check-all
```

For ld, tests are listed in `.exp` files under `ld/testsuite`.
A single test normally consists of a `.d` file and several associated `.s` files.

To run the tests in `ld/testsuite/ld-shared/shared.exp`:
```sh
make -C out/debug check-ld RUNTESTFLAGS=ld-shared/shared.exp
```

### Misc

* A bot updates `bfd/version.h` (`BFD_VERSION_DATE`) daily.
* Test coverage is low.

```
cat > .dir-locals-2.el <<e
((nil . ((indent-tabs-mode . t)
         (tab-width . 8)))
 (c-mode . ((c-file-style . "gnu"))))
e
echo 'BasedOnStyle: GNU' > .clang-format
```

## gdb

* Mailing list: <https://sourceware.org/pipermail/gdb> <https://sourceware.org/pipermail/gdb-prs>

gdb resides in the binutils-gdb repository.
configure enables gdb and gdbserver by default. You just need to make sure `--disable-gdb --disable-gdbserver` is not on the configure line.

Run gdb under the build directory:
```sh
gdb/gdb -data-directory gdb/data-directory
```

To run the tests in `gdb/testsuite/gdb.dwarf2/dw2-abs-hi-pc.exp`:

```sh
make check-gdb RUNTESTFLAGS=gdb.dwarf2/dw2-abs-hi-pc.exp

# cd $build/gdb/testsuite/outputs/gdb.dwarf2/dw2-abs-hi-pc
```

## glibc

* Repository: <https://sourceware.org/git/gitweb.cgi?p=glibc.git>
* Wiki: <https://sourceware.org/glibc/wiki/>
* Bugzilla: <https://sourceware.org/bugzilla/>
* Mailing lists: `{libc-announce,libc-alpha,libc-locale,libc-stable,libc-help}@sourceware.org`
* Patchwork: <https://patchwork.sourceware.org/project/glibc/list/> [Patch Review Workflow](https://sourceware.org/glibc/wiki/Patch%20Review%20Workflow?highlight=%28git-pw%29)

(Mostly) an implementation of the user-space side of standard C/POSIX functions with Linux extensions.

A very unfortunate fact: glibc can only be built with `-O2`, not `-O0` or `-O1`.
If you want to have an un-optimized debug build, deleting an object file and recompiling it with `-g` usually works.
Another workaround is `#pragma GCC optimize ("O0")`.

The `-O2` issue is probably related to (1) expected inlining and (2) avoiding dynamic relocations.

To cross compile for aarch64:
```zsh
../../configure --prefix=/tmp/glibc/aarch64 --host=aarch64-linux-gnu
```
If you don't have a C++ compiler. Specify `CXX=false`.

To cross compile for i686:
```zsh
../../configure --prefix=/tmp/glibc/i686 --host=i686-linux-gnu CC='gcc -m32' CXX='g++ -m32' && make -j 50 && make -j 50 install && cp -f /usr/lib/i386-linux-gnu/{libgcc_s.so.1,libstdc++.so.6} /tmp/glibc/i686/
```
For cross compiling, `run-built-tests` is yes only if `test-wrapper` is set.

To build a directory:
```zsh
make -r -C ~/Dev/glibc/stdlib objdir=$PWD subdir=stdlib subdir_lib
```

The run a program with the built dynamic loader:
```zsh
$build/testrun.sh $program

# Debug a test. --direct can avoid spawning a new process.
cgdb -ex 'set exec-wrapper env LD_LIBRARY_PATH=.:./math:./elf:./dlfcn:./nss:./nis:./rt:./resolv:./mathvec:./support:./crypt:./nptl' elf/tst-nodelete --direct
```

To test one directory (say, `libio`), delete `libio/*.out` files and run
```zsh
make -r -C ~/Dev/glibc/libio objdir=$PWD check -j 20
```

(
```sh
../../configure --prefix=/tmp/glibc/lld && make -j 50 && make -j 50 install && 'cp' -f /usr/lib/x86_64-linux-gnu/libgcc_s.so.1 /tmp/glibc/lld/lib/
```
)

Delete failed tests so that the next `make check` invocation will re-run them:
```sh
rg -lg '*.test-result' '^FAIL' | while read i; do rm -f ${i/.test-result/.out} $i; done
```

### GRTE

In a llvm-project build directory, build clang, lld, and `clang_rt.crtbegin*.o`
```sh
ninja -C /tmp/out/custom1 clang crt lld
```

```sh
git checkout origin/google/grte/v5-2.27/master
mkdir -p out/grte && cd out/grte
../../configure --prefix=/tmp/grte/play --disable-werror --disable-float128 --with-clang --with-lld --enable-static-pie CC=/tmp/out/custom1/bin/clang CXX=/tmp/out/custom1/bin/clang++
make -j 30
make -j 30 install
```

### build-many-glibcs.py

Run the following commands to populate `/tmp/glibc-many` with toolchains.
Caution: please make sure the target file system has tens of gigabytes.

Preparation:
```sh
scripts/build-many-glibcs.py /tmp/glibc-many checkout --shallow
scripts/build-many-glibcs.py /tmp/glibc-many host-libraries

# Build a bootstrap GCC (static-only, C-only, --with-newlib).
# /tmp/glibc-many/src/gcc/gcc/configure --srcdir=/tmp/glibc-many/src/gcc/gcc --prefix=/tmp/glibc-many/install/compilers/aarch64-linux-gnu --with-sysroot=/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot --with-gmp=/tmp/glibc-many/install/host-libraries --with-mpfr=/tmp/glibc-many/install/host-libraries --with-mpc=/tmp/glibc-many/install/host-libraries --enable-shared --enable-threads --enable-languages=c,c++,lto ... --build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --target=aarch64-glibc-linux-gnu ...
scripts/build-many-glibcs.py /tmp/glibc-many compilers aarch64-linux-gnu
scripts/build-many-glibcs.py /tmp/glibc-many compilers powerpc64le-linux-gnu
scripts/build-many-glibcs.py /tmp/glibc-many compilers sparc64-linux-gnu
```

* `--shallow` passes `--depth 1` to the git clone command.
* `--keep all` keeps intermediary build directories intact. You may want this option to investigate build issues.

The `glibcs` command will delete the glibc build directory, build glibc, and run `make check`.

```sh
# Build glibc using bootstrap GCC in <path>/install/compilers/aarch64-linux-gnu/bin/aarch64-linux-gcc
# Then build GCC using the built glibc.
# /tmp/glibc-many/src/glibc/configure --prefix=/usr --enable-profile --build=x86_64-pc-linux-gnu --host=aarch64-glibc-linux-gnu CC=aarch64-glibc-linux-gnu-gcc CXX=aarch64-glibc-linux-gnu-g++ AR=aarch64-glibc-linux-gnu-ar AS=aarch64-glibc-linux-gnu-as LD=aarch64-glibc-linux-gnu-ld NM=aarch64-glibc-linux-gnu-nm OBJCOPY=aarch64-glibc-linux-gnu-objcopy OBJDUMP=aarch64-glibc-linux-gnu-objdump RANLIB=aarch64-glibc-linux-gnu-ranlib READELF=aarch64-glibc-linux-gnu-readelf STRIP=aarch64-glibc-linux-gnu-strip
# Find built glibc in <path>/install/glibcs/aarch64-linux-gnu
scripts/build-many-glibcs.py /tmp/glibc-many glibcs aarch64-linux-gnu
# Find the logs and test results under /tmp/glibc-many/logs/glibcs/aarch64-linux-gnu/

scripts/build-many-glibcs.py /tmp/glibc-many glibcs powerpc64le-linux-gnu

scripts/build-many-glibcs.py /tmp/glibc-many glibcs sparc64-linux-gnu
```

For the `glibcs` command, add `--full-gcc` to build C++.

```sh
many=/tmp/glibc-many
$many/install/compilers/aarch64-linux-gnu/bin/aarch64-glibc-linux-gnu-g++ -Wl,--dynamic-linker=$many/install/glibcs/aarch64-linux-gnu/lib/ld-linux-aarch64.so.1 -Wl,-rpath=$many/install/compilers/aarch64-linux-gnu/sysroot/lib64:$many/install/compilers/aarch64-linux-gnu/aarch64-glibc-linux-gnu/lib64 a.cc
./a.out

$many/install/compilers/riscv64-linux-gnu/bin/riscv64-glibc-linux-gnu-g++ -Wl,--dynamic-linker=$many/install/glibcs/riscv64-linux-gnu/lib/ld-linux-riscv64.so.1 -Wl,-rpath=$many/install/compilers/riscv64-linux-gnu/sysroot/lib64:$many/install/compilers/riscv64-linux-gnu/riscv64-glibc-linux-gnu/lib64 a.cc
./a.out

$many/install/compilers/x86_64-linux-gnu/bin/x86_64-glibc-linux-gnu-gcc -Wl,--dynamic-linker=$many/install/glibcs/x86_64-linux-gnu/lib64/ld-linux-x86-64.so.2 -Wl,-rpath=$many/install/compilers/x86_64-linux-gnu/sysroot/lib64:$many/install/compilers/x86_64-linux-gnu/x86_64-glibc-linux-gnu/lib64 a.c
./a.out
```

"On build-many-glibcs.py and most stage1 compiler bootstrap, gcc is build statically against newlib.
 the static linked gcc (with a lot of disabled features) is then used to build glibc and then the stage2 gcc (which will then have all the features that rely on libc enabled)
 so the stage1 gcc *might* not have the require started files"

During development, some interesting targets:

```sh
make -C out/debug check-abi
```

Building with Clang is not an option.

* Clang does not support GCC nested functions [BZ #27220](https://sourceware.org/bugzilla/show_bug.cgi?id=27220)
* x86 `PRESERVE_BND_REGS_PREFIX`: integrated assembler does not support the `bnd` prefix.
* `sysdeps/powerpc/powerpc64/Makefile`: Clang does not support `-ffixed-vrsave -ffixed-vscr`

### Misc

To regenerate `configure` from `configure.ac`: <https://sourceware.org/glibc/wiki/Regeneration>.
Consider installing autoconf/automake somewhere with the required version.

```sh
~/projects/autoconf-2.69/bin/autoconf

PATH=~/projects/autoconf-2.69/bin:$PATH ~/projects/automake-1.15.1/bin/automake
```

In `elf/`, `.o` objects use `-fpie -DPIC -DMODULE_NAME=libc`, while `.os` objects use `-fPIC -DPIC -DMODULE_NAME=rtld`.

## GCC

* Repository for documentation: <https://gcc.gnu.org/cgit/gcc-wwwdocs/>
* Mailing lists: `gcc-{patches,regression}@sourceware.org`
* Branches: `releases/gcc-8`, `releases/gcc-9`, `releases/gcc-10`
* [Read-write Git access](https://gcc.gnu.org/gitwrite.html)

`--disable-bootstrap` is the most important, otherwise you will get a stage 2 build.
It is not clear what `make` does when you touch a source file. It definitely rebuilds stage1,
but it is not clear to me how well stage2 dependency is handled.
Anyway, touching a source file causes a total build is not what you desire.

```sh
../../configure --prefix=/tmp/gcc/release --disable-bootstrap --enable-languages=c,c++ --disable-multilib
make -j 30

# Incremental build
make -C gcc cc1 cc1plus xgcc
make -C x86_64-pc-linux-gnu/libstdc++-v3

../../configure --prefix=/tmp/gcc/debug --disable-bootstrap --enable-languages=c,c++ --disable-multilib CFLAGS='-O0 -g' CXXFLAGS='-O0 -g'
```

To build a GCC targeting riscv64-linux-gnu, we can run `scripts/build-many-glibcs.py` from glibc first, then invoke:
```sh
../../configure --prefix=/tmp/gcc/rv --target=riscv64-linux-gnu --with-sysroot=/tmp/glibc-many/install/compilers/riscv64-linux-gnu-rv64imafdc-lp64d/sysroot --with-arch=rv64imafdc --with-abi=lp64d --disable-multilib --disable-libsanitizer --enable-languages=c,c++ CFLAGS='-O0 -g3 -gsplit-dwarf' CXXFLAGS='-O0 -g3 -gsplit-dwarf' LDFLAGS='-fuse-ld=lld -Wl,--gdb-index'
```

To build a GCC targeting other targets:
```sh
../../configure --prefix=/tmp/gcc/aarch64 --target=aarch64-linux-gnu --with-sysroot=/tmp/glibc-many/install/compilers/aarch64-linux-gnu/sysroot --disable-libsanitizer --enable-languages=c,c++ CFLAGS='-O0 -g3 -gsplit-dwarf' CXXFLAGS='-O0 -g3 -gsplit-dwarf' LDFLAGS='-fuse-ld=lld -Wl,--gdb-index'
../../configure --prefix=/tmp/gcc/arm --target=arm-linux-gnueabi --with-sysroot=/tmp/glibc-many/install/compilers/arm-linux-gnueabi/sysroot --disable-libsanitizer --enable-languages=c,c++ CFLAGS='-O0 -g3 -gsplit-dwarf' CXXFLAGS='-O0 -g3 -gsplit-dwarf' LDFLAGS='-fuse-ld=lld -Wl,--gdb-index'
```

If you have a system cross compiler, the steps will be easier. However, `dg-do run` tests would fail as we don't specify `--sysroot=/tmp/gcc/arm/arm-linux-gnueabi -Wl,--sysroot=`.
```
rsync -a /usr/arm-linux-gnueabi/{include,lib} /tmp/gcc/arm/arm-linux-gnueabi/
../../configure --prefix=/tmp/gcc/sh --target=arm-linux-gnueabi --disable-libsanitizer --enable-languages=c,c++ CFLAGS='-O0 -g3 -gsplit-dwarf' CXXFLAGS='-O0 -g3 -gsplit-dwarf' LDFLAGS='-fuse-ld=lld -Wl,--gdb-index'

rsync -a /usr/sh4-linux-gnu/{include,lib} /tmp/gcc/sh/sh4-linux-gnu/
../../configure --prefix=/tmp/gcc/sh --target=sh4-linux-gnu --disable-libsanitizer --enable-languages=c,c++ CFLAGS='-O0 -g3 -gsplit-dwarf' CXXFLAGS='-O0 -g3 -gsplit-dwarf' LDFLAGS='-fuse-ld=lld -Wl,--gdb-index'
```

To run tests:
```
make -C out/release check-gcc

# Test just gcc/testsuite/gcc.target/i386/i386.exp and gcc/testsuite/g++.target/i386/i386.exp
make -C out/release check-gcc RUNTESTFLAGS='i386.exp'

# Test one file
make -C out/release check-gcc RUNTESTFLAGS='i386.exp=large-data.c'
```

Some architectures support multilib (aarch64/powerpc64/x86/etc). On x86 you can use `--with-multilib='m32;m64'`, but you need to make sure the needed headers/libraries are available.

Use built libstdc++ and libgcc.
```sh
$build/gcc/xg++ -B $build/release/gcc forced1.C -L $build/x86_64-pc-linux-gnu/libstdc++-v3/src/.libs -Wl,-rpath=$build/x86_64-pc-linux-gnu/libstdc++-v3/src/.libs,-rpath=$build/x86_64-pc-linux-gnu/libgcc

$prefix/bin/g++ -Wl,-rpath=$prefix/lib64 a.cc
```

```
gcc/incpath.c include path
```

To build older GCC versions, pass `--disable-libsanitizer` to `configure`, as older sanitizers may be unbuildable due to `linux/*` files.

### Misc

* A bot updates `ChangeLog` files daily. `Daily bump.`

<https://sourceware.org/git/?p=builder.git;a=summary> provides continuous integration for some projects.

* To debug a cc1 command, `gcc -c a.c -wrapper gdb,--args`

## Unlisted

autotools, bison, m4, make, ...

## Contributing

[GNU Coding Standards](https://www.gnu.org/prep/standards/). Emacs has good built-in support.
clang-format's support is not as good.

Legally significant changes need [Copyright Papers](https://www.gnu.org/prep/maintain/html_node/Copyright-Papers.html).

```vim
setlocal cin cino=>4,n-2,{2,^-2,:2,=2,g0,h2,p5,t0,+2,(0,u0,w1,m1 sw=2 ts=8 sts=2 noet tw=79 colorcolumn=80
setlocal fo-=ro fo+=cql
```

In GDB, `source gcc/gdbhooks.py`
