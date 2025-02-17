layout: post
title: zstd compressed debug sections
author: MaskRay
tags: [binutils,llvm]
---

Updated in 2022-10.

In January I wrote [Compressed debug sections](/blog/2022-01-23-compressed-debug-sections).
The venerable zlib shows its age and there are replacements which are better in every metric except adoption and a larger memory footprint.
The obvious choice was Zstandard, but I was not so confident about adoptinig it and solving the ecosystem issue.
At any rate, I slowly removed some legacy `.zdebug` support from llvm-project so that a new format could be more easily introduced.

<!-- more -->

In June, Cole Kissane posted [[RFC] Zstandard as a second compression method to LLVM](https://discourse.llvm.org/t/rfc-zstandard-as-a-second-compression-method-to-llvm/63399) on LLVM discourse forums.
I learned that other folks were investigating a better compression format for ELF compressed debug sections and told myself: it's high time to propose `ELFCOMPRESS_ZSTD` to the generic System V Application Binary Interface (generic ABI).

ELF is an elegant format which has passed the test of time. Many things created by the forefathers from 30 years ago carry over and are still used today.
Every new feature, even a small addition like introducing a new constant has to pass a significant high bar for acceptance.
There were many discussions on [Add new ch_type value: ELFCOMPRESS_ZSTD](https://groups.google.com/g/generic-abi/c/satyPkuMisk).

Personally I think a selected format need to have these properties:

* It has an open compression algorithm and implementation.
* It provides significant benefits (compression speed, decompression speed, compression ratio) with a decent memory footprint and complexity.
* It has full backward compatibility. In 20 years I want to be able to decompress a debug section created today.
* It has a wide range of and active use cases. When the format value is standardized, consumers are willing to add support.
* It has good documentation.
* It's easy to use.

A compression format satisfying all these properties are rare. ELF does not like introducing a lot of options for one feature.
It's not an experiment site for every new fancy compression format. We are wary of platform fragmentation and consumers don't like support a number of formats each claiming to be a good choice at a slightly different angle.
See the appendix for my recent test of many compression utilities.

I made many arguments in the proposal thread. It took about one month and `ELFCOMPRESS_ZSTD` was [accepted in 2022-07](https://groups.google.com/g/generic-abi/c/satyPkuMisk/m/KwTF_U8rBAAJ).

## Toolchain support

The next step is to add toolchain support. The most important pieces are assemblers, linkers, and debuggers.
Many other pieces are needed as well.

Toolchain components:

* binutils: [all implemented as of 2022-11](https://sourceware.org/bugzilla/show_bug.cgi?id=29397)
  + addr2line: symbolization needs to decompress debug sections
  + gas: compress debug sections
  + ld, gold: decompress compressed input sections and compress output debug sections. [Implemented](https://sourceware.org/bugzilla/show_bug.cgi?id=29641)
  + dwp: decompress compressed `.dwo`. dwp uses gold's code
  + nm: `--line-numbers` uses debug information
  + objcopy: `--decompress-debug-sections` and `--compress-debug-sections=zstd`
  + objdump: `--dwarf` decompresses compressed debug sections
  + readelf: `--debug-dump` and `--decompress` decompress compressed sections. [feature request](https://sourceware.org/bugzilla/show_bug.cgi?id=29640)
* gdb: [implemented](https://sourceware.org/bugzilla/show_bug.cgi?id=29397)
  + decompress compressed debug sections in executables, shared objects, separate debug files, and `.dwo` files. [Feature request](https://sourceware.org/bugzilla/show_bug.cgi?id=29563)
  + MiniDebugInfo section `.gnu_debugdata` is compressed with xz. [zstd feature request](https://sourceware.org/bugzilla/show_bug.cgi?id=29584)
* GCC: [13.0 will support `-gz=zstd`](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=106897)
* llvm-project: all implemented as of 2022-09 (milestone: 16.0.0). The default `LLVM_ENABLE_ZSTD=on` needs a [CMake config file](https://github.com/facebook/zstd/issues/3271) to take effects.
  + Clang: compress `.o` and (if split DWARF is enabled) `.dwo` with level 5
  + llvm-objcopy: `--decompress-debug-sections` and `--compress-debug-sections=zstd` (level 5). Implemented in [D130458](https://reviews.llvm.org/D130458) (ELFCLASS64) and [D134385](https://reviews.llvm.org/D134385) (ELFCLASS32)
  + ld.lld: decompress `ELFCOMPRESS_ZSTD` input sections ([D129406](https://reviews.llvm.org/D129406)) and compress output debug sections with level 3 ([D133548](https://reviews.llvm.org/D133548), [D133679](https://reviews.llvm.org/D133679))
  + llvm-dwarfdump: use LLVMObject API to decompress `ELFCOMPRESS_ZSTD` input sections ([D134116](https://reviews.llvm.org/D134116))
  + llvm-dwp: use LLVMObject API
  + llvm-symbolizer: use LLVMObject API
  + lldb: use LLVMObject API
* elfutils: [implemented in 2022-12](https://sourceware.org/bugzilla/show_bug.cgi?id=29565)
* mold: [implemented in 2022-09](https://github.com/rui314/mold/issues/700)

Other languages:

* Go: [cmd/link feature request](https://github.com/golang/go/issues/55107)

Other utilities:

* bloaty: its `-d compileunits` parses DWARF. [Feature request](https://github.com/google/bloaty/issues/318)
* dwz

### llvm-project support

On the llvm-project side, there was a lot of debate on how the API should look like.
In the week of 2022-09-09 we (Cole Kissane, David Blaikie, I) reached an agreement that the free function style compression API was acceptable.
I have pushed some changes and `llvm-objcopy --compress-debug-sections=zstd`, `clang -gz=std`, `ld.lld --compress-debug-sections=zstd` are available now.
Note that I chose to implement llvm-objcopy support before others so that I could test other components with llvm-objcopy.

```
% cat a.cc
#include <iostream>
int main() { std::cout << "zstd"; }
% clang -c -g -gz a.cc
% readelf -x .debug_info a.o

Hex dump of section '.debug_info':
 NOTE: This section has relocations against it, but these have NOT been applied to this dump.
  0x00000000 02000000 00000000 1a180000 00000000 ................
  0x00000010 01000000 00000000 28b52ffd 601a17ed ........(./.`...
...
```

`ELFCOMPRESS_ZSTD` (2) can be identified by the first 4 bytes. In a little-endian object file, it displays as `02000000`.

If llvm-objcopy is built with zstd support, use `--decompress-debug-sections` to decompress an object file:
```
% llvm-objcopy --decompress-debug-sections a.o a.o.decompressed
% readelf -x .debug_info a.o.decompressed

Hex dump of section '.debug_info':
 NOTE: This section has relocations against it, but these have NOT been applied to this dump.
  0x00000000 16180000 05000108 00000000 01002100 ..............!.
  0x00000010 01000000 00000000 00020000 00000000 ................
...
```

On the llvm-project side we reached full feature readiness in 2022-10.

It would be nice that someone picks up the work items on the GNU side so that many Linux distributions can start investigating the adoption of zstd compressed debug sections.

### GNU toolchain support

The main changes were for the binutils-gdb repository. This work turned out to be much more challenging than my work for llvm-project.

The entry points of zstd compression features were in binutils, gas, and ld. binutils and ld use bfd, so we needed to update bfd.

I created `config/zstd.m4` by following `config/zlib.m4`. `AC_ZSTD` in `config/zstd.m4` defines `ZLIB_CFLAGS` and `ZLIB_LDLIBS`.
After plumbing it into `bfd/configure.ac` and `bfd/Makefile.am`, I needed to adding `AC_ZSTD` to every top-level project which uses bfd as bfd is linked as an archive and there is no good transitive dependency support.

Here was the change for `bfd/Makefile.am`. The pattern needed to be repeated in many other directories.
```
--- a/bfd/Makefile.am
+++ b/bfd/Makefile.am
@@ -60 +60 @@ NO_WERROR = @NO_WERROR@
-AM_CFLAGS = $(WARN_CFLAGS) $(ZLIBINC)
+AM_CFLAGS = $(WARN_CFLAGS) $(ZLIBINC) $(ZSTD_CFLAGS)
@@ -779 +779 @@ libbfd_la_DEPENDENCIES = $(OFILES) ofiles
-libbfd_la_LIBADD = `cat ofiles` @SHARED_LIBADD@ $(LIBDL) $(ZLIB)
+libbfd_la_LIBADD = `cat ofiles` @SHARED_LIBADD@ $(LIBDL) $(ZLIB) $(ZSTD_LIBS)
```

```text
% rg -l --sort=path ZSTD_LIBS
bfd/Makefile.am
bfd/Makefile.in
bfd/configure
binutils/Makefile.am
binutils/Makefile.in
binutils/configure
gas/Makefile.am
gas/Makefile.in
gas/configure
gdb/Makefile.in
gdb/acinclude.m4
gdb/configure
ld/Makefile.am
ld/Makefile.in
ld/configure
libctf/Makefile.in
libctf/configure
libctf/configure.ac
sim/Makefile.in
sim/arch-subdir.mk.in
sim/common/Make-common.in
sim/configure
sim/ppc/Makefile.in
```

Remember to update auto-generated files with the appropriate versions of autoconf and automake:
```sh
PATH=~/projects/automake-1.15.1/bin:$PATH ~/projects/autoconf-2.69/bin/autoreconf -vf bfd binutils gas ld libctf sim
```

```sh
make -C bfd headers
```

Some bfd/ file changes require updating `bfd/bfd-in2.h` with `make -C $build/bfd headers`.
```c
/* DO NOT EDIT!  -*- buffer-read-only: t -*-  This file is automatically
   generated from "bfd-in.h", "init.c", "opncls.c", "libbfd.c",
   "bfdio.c", "bfdwin.c", "section.c", "archures.c", "reloc.c",
   "syms.c", "bfd.c", "archive.c", "corefile.c", "targets.c", "format.c",
   "linker.c", "simple.c" and "compress.c".
   Run "make headers" in your build bfd/ to regenerate.  */

/* Main header file for the bfd library -- portable access to object files.
```


## Appendix

(Conducted the experiment in 2022-10.) I have a `-DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD=all` build of trunk clang.
The 3 largest DWARF v5 debug sections are `.debug_info`, `.debug_str`, and `.debug_line`.
```
% bloaty clang-16
    FILE SIZE        VM SIZE
 --------------  --------------
  32.3%   451Mi   0.0%       0    .debug_info
  16.4%   229Mi   0.0%       0    .debug_str
  11.4%   159Mi   0.0%       0    .debug_line
  11.3%   157Mi  55.0%   157Mi    .text
   8.0%   112Mi   0.0%       0    .strtab
   5.7%  80.2Mi   0.0%       0    .debug_str_offsets
   3.6%  50.4Mi  17.6%  50.4Mi    .rodata
   2.4%  33.6Mi   0.0%       0    .debug_addr
   2.2%  30.8Mi  10.7%  30.8Mi    .eh_frame
   1.7%  24.0Mi   0.0%       0    .symtab
   1.0%  13.6Mi   4.7%  13.6Mi    .rela.dyn
   1.0%  13.4Mi   0.0%       0    .debug_rnglists
   0.8%  11.0Mi   3.8%  11.0Mi    .dynstr
   0.7%  10.5Mi   3.7%  10.5Mi    .data.rel.ro
   0.6%  8.05Mi   0.0%       0    .debug_abbrev
   0.5%  7.69Mi   2.7%  7.69Mi    .eh_frame_hdr
   0.2%  2.79Mi   1.0%  2.79Mi    .dynsym
   0.1%   848Ki   0.3%   848Ki    .gnu.hash
   0.1%   827Ki   0.1%   263Ki    [21 Others]
   0.0%       0   0.2%   544Ki    .bss
   0.0%   497Ki   0.2%   497Ki    .data
 100.0%  1.37Gi 100.0%   286Mi    TOTAL
```

`ninja -t commands bin/clang` dumps the compiler driver command which links the executable.
Invoke the command with `-fuse-ld=lld -Wl,--prproduce=/tmp/clang-debug.tar` to get a tarball. Use `llvm-objcopy --dump-section` to extract a section.

```sh
cd /tmp
tar xf clang-debug.tar
cd clang-debug
ld.lld @response.txt

llvm-objcopy --dump-section .debug_info=debug_info clang-16 /dev/null
llvm-objcopy --dump-section .debug_str=debug_str clang-16 /dev/null
llvm-objcopy --dump-section .debug_line=debug_line clang-16 /dev/null
```

I have tried brotli, bzip2, gzip, lz4, lzo, pigz, xz, zstd, and manually verified that zstd is the best considering compression speed, decompression speed, and compression ratio.
Figuring out API for all these libraries will be inconvenient. So I take a shortcut: install these compression utilities with the package manager and hope that they use similar compiler driver options and the comparison is relative fair.

Here are some results:

```sh
#!/bin/zsh
i=$1
o0=$i.o0
o1=$i.o1

measure() {
  rm -f $o0 $o1
  local a=$(/usr/bin/time -f '%e\t%M\t' ${=1} 2>&1)
  ${=2}
  local b=$(/usr/bin/time -f '%e\t%M\t' ${=3} 2>&1)
  stat -c "$a$b%s"$'\t'"$1" $o0
}
measure_no_o() {
  rm -f $o0 $o1
  local a=$(/usr/bin/time -f '%e\t%M\t' ${=1} > $o0 2> t0)
  ${=2}
  local b=$(/usr/bin/time -f '%e\t%M\t' ${=3} > $o1 2> t1)
  stat -c "$(<t0)$(<t1)%s"$'\t'"$1" $o0
}

do_pigz() { measure "pigz -zfk -p 1 -S .o0 $i" "cp -f $o0 $o1.gz" "pigz -dfk -p 1 $o1.gz"; }
do_brotli() { measure "brotli -f $* $i -o $o0" "" "brotli -df $o0 -o $o1"; }
do_bzip2() { measure_no_o "bzip2 -cf $* $i" "" "bzip2 -dcf $o0"; }
do_gzip() { measure "gzip -fk -S .o0 $* $i" "" "cp -f $o0 $o1.gz" "gzip -dfk $o1.gz"; }
do_lz4() { measure "lz4 -fq $* $i $o0" "" "lz4 -dfq $o0 $o1"; }
do_lzop() { measure "lzop -f $* $i -o $o0" "" "lzop -df $o0 -o $o1"; }
do_xz() { measure_no_o "xz -c $* $i" "" "xz -dc $o0"; }
do_zstd() { measure "zstd -fq $@ $i -o $o0" "" "zstd -dfq $o0 -o $o1"; }

print 'comp\tRSS\tdecomp\tRSS\tsize\tcommand'
do_pigz
do_brotli -q 1; do_brotli -q 3; do_brotli -q 5; do_brotli -q 9
do_bzip2 -1; do_bzip2 -3
do_gzip -1; do_gzip -3
do_lz4 --fast; do_lz4 -1; do_lz4 -3; do_lz4 -5; do_lz4 -9
do_lzop -1; do_lzop -3; do_lzop -9
do_xz -0; do_xz -1; do_xz -3; do_xz -6
do_zstd --fast; do_zstd -1; do_zstd -3; do_zstd -5; do_zstd -9; do_zstd -18
```

```text
% numactl -C 20 ./bench.sh debug_info
comp    RSS     decomp  RSS     size    command
19.99   2568    2.37    2212    212342615       pigz -zfk -p 1 -S .o0 debug_info
2.44    4044    2.48    19188   244890939       brotli -f -q 1 debug_info -o debug_info.o0
5.92    36300   2.22    19264   211420890       brotli -f -q 3 debug_info -o debug_info.o0
22.77   110516  2.11    21128   193129564       brotli -f -q 5 debug_info -o debug_info.o0
239.99  114140  2.12    20988   191699881       brotli -f -q 9 debug_info -o debug_info.o0
36.24   2396    15.71   1604    212700609       bzip2 -cf -1 debug_info
34.29   3640    16.52   2396    221881700       bzip2 -cf -3 debug_info
9.16    2008    0.14    2604    227302010       gzip -fk -S .o0 -1 debug_info
11.57   2072    0.14    2536    221444175       gzip -fk -S .o0 -3 debug_info
1.16    8872    0.49    9116    310918343       lz4 -fq --fast debug_info debug_info.o0
1.25    8828    0.48    9168    295430369       lz4 -fq -1 debug_info debug_info.o0
6.85    8904    0.49    8896    265793678       lz4 -fq -3 debug_info debug_info.o0
8.60    9016    0.49    9000    263660439       lz4 -fq -5 debug_info debug_info.o0
14.26   8936    0.48    8848    262365618       lz4 -fq -9 debug_info debug_info.o0
1.42    2244    1.24    1720    283798959       lzop -f -1 debug_info -o debug_info.o0
1.43    2088    1.25    1632    282736380       lzop -f -3 debug_info -o debug_info.o0
71.94   2504    1.32    1756    240402339       lzop -f -9 debug_info -o debug_info.o0
27.76   5024    11.23   2560    164560076       xz -c -0 debug_info
34.19   10992   10.97   3292    163894892       xz -c -1 debug_info
56.63   34032   10.83   6296    164369948       xz -c -3 debug_info
149.35  97648   10.75   10400   149764760       xz -c -6 debug_info
2.55    12696   0.70    2916    252027101       zstd -fq --fast debug_info -o debug_info.o0
2.97    12528   0.83    2964    234661626       zstd -fq -1 debug_info -o debug_info.o0
3.95    41764   0.90    4568    215717860       zstd -fq -3 debug_info -o debug_info.o0
5.93    43960   0.91    4572    213222742       zstd -fq -5 debug_info -o debug_info.o0
9.96    90436   0.89    6668    208886078       zstd -fq -9 debug_info -o debug_info.o0
110.59  198708  1.46    10588   178996922       zstd -fq -18 debug_info -o debug_info.o0
% numactl -C 20 ./bench.sh debug_str
comp    RSS     decomp  RSS     size    command
5.19    2552    0.68    2204    41086567        pigz -zfk -p 1 -S .o0 debug_str
0.75    3680    0.50    19156   45784253        brotli -f -q 1 debug_str -o debug_str.o0
1.58    36320   0.42    19116   37832846        brotli -f -q 3 debug_str -o debug_str.o0
4.05    52508   0.38    20012   30553722        brotli -f -q 5 debug_str -o debug_str.o0
18.87   80704   0.34    19924   24386096        brotli -f -q 9 debug_str -o debug_str.o0
16.70   2416    4.67    1600    37520606        bzip2 -cf -1 debug_str
16.96   3736    4.82    2392    31299002        bzip2 -cf -3 debug_str
2.46    1968    0.03    2528    52866269        gzip -fk -S .o0 -1 debug_str
2.89    1996    0.03    2560    48526083        gzip -fk -S .o0 -3 debug_str
0.52    6776    0.23    6944    68238485        lz4 -fq --fast debug_str debug_str.o0
0.53    6644    0.23    7004    65392601        lz4 -fq -1 debug_str debug_str.o0
1.86    6540    0.23    6596    47813656        lz4 -fq -3 debug_str debug_str.o0
2.70    6636    0.23    6528    46341752        lz4 -fq -5 debug_str debug_str.o0
4.92    6568    0.23    6684    45862077        lz4 -fq -9 debug_str debug_str.o0
0.56    2168    0.57    1672    71677076        lzop -f -1 debug_str -o debug_str.o0
0.56    2128    0.57    1604    71133429        lzop -f -3 debug_str -o debug_str.o0
15.73   2488    0.51    1812    47430843        lzop -f -9 debug_str -o debug_str.o0
6.87    4940    2.46    2496    38516552        xz -c -0 debug_str
7.46    10892   2.05    3256    33223964        xz -c -1 debug_str
17.02   34020   1.77    6280    29506364        xz -c -3 debug_str
68.33   97664   1.32    10396   21296880        xz -c -6 debug_str
1.14    11304   0.29    2920    50492121        zstd -fq --fast debug_str -o debug_str.o0
1.16    11096   0.29    3000    42965990        zstd -fq -1 debug_str -o debug_str.o0
1.26    37292   0.31    4512    37978434        zstd -fq -3 debug_str -o debug_str.o0
2.18    39716   0.30    4580    33960116        zstd -fq -5 debug_str -o debug_str.o0
3.87    82580   0.28    6312    29911833        zstd -fq -9 debug_str -o debug_str.o0
66.85   186704  0.28    10668   23900220        zstd -fq -18 debug_str -o debug_str.o0
% numactl -C 20 ./bench.sh debug_line
comp    RSS     decomp  RSS     size    command
5.08    2524    0.63    2200    45210969        pigz -zfk -p 1 -S .o0 debug_line
0.56    3800    0.48    19104   42533806        brotli -f -q 1 debug_line -o debug_line.o0
1.15    36304   0.36    19272   30144639        brotli -f -q 3 debug_line -o debug_line.o0
3.38    56988   0.31    20344   23170193        brotli -f -q 5 debug_line -o debug_line.o0
18.64   86040   0.30    20032   20805984        brotli -f -q 9 debug_line -o debug_line.o0
12.74   2348    3.93    1548    40423655        bzip2 -cf -1 debug_line
13.48   3940    3.93    2364    34569267        bzip2 -cf -3 debug_line
2.19    1992    0.03    2584    51136260        gzip -fk -S .o0 -1 debug_line
2.73    1988    0.03    2440    49092753        gzip -fk -S .o0 -3 debug_line
0.33    7436    0.16    7848    66771211        lz4 -fq --fast debug_line debug_line.o0
0.36    7556    0.16    7656    64913560        lz4 -fq -1 debug_line debug_line.o0
1.55    7204    0.16    7288    49279291        lz4 -fq -3 debug_line debug_line.o0
2.13    7172    0.16    7212    48268337        lz4 -fq -5 debug_line debug_line.o0
3.46    7168    0.16    7080    47963094        lz4 -fq -9 debug_line debug_line.o0
0.42    2164    0.39    1636    67304513        lzop -f -1 debug_line -o debug_line.o0
0.42    2040    0.39    1672    66569062        lzop -f -3 debug_line -o debug_line.o0
24.08   2512    0.40    1704    47788969        lzop -f -9 debug_line -o debug_line.o0
5.44    4884    2.07    2544    31059860        xz -c -0 debug_line
5.71    10980   1.61    3272    24860272        xz -c -1 debug_line
12.29   33944   1.37    6344    21432224        xz -c -3 debug_line
45.04   97644   1.27    10480   18536212        xz -c -6 debug_line
0.67    11688   0.20    2940    50842109        zstd -fq --fast debug_line -o debug_line.o0
0.73    11400   0.21    3012    38849471        zstd -fq -1 debug_line -o debug_line.o0
0.87    37992   0.22    4544    28827098        zstd -fq -3 debug_line -o debug_line.o0
1.30    40180   0.22    4552    26554847        zstd -fq -5 debug_line -o debug_line.o0
2.34    83552   0.21    6624    23464249        zstd -fq -9 debug_line -o debug_line.o0
43.67   188676  0.22    10584   20195067        zstd -fq -18 debug_line -o debug_line.o0
```

When compressing debug sections, zstd and brotli are significantly better than the other choices.
zstd slightly outperforms brotli in compression speed and compression ratio while being much fast at decompression.

`xz -3` has a great compression ratio (higher levels are too slow).
zstd and brotli with higher levels are extremely slow and can hardly achieve the xz compression ratio, but their decompression speed may compensate for that.

zlib (used by pigz) and bzip2 look pretty bad.

For zstd, the built-in parallel compression support is a plus.
