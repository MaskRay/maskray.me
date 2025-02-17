layout: post
title: Distribution of debug information
author: MaskRay
tags: [debug,gcc,llvm]
---

Updated in 2024-08.

Note: The article will likely get frequent updates in the next few days.

This article describes some approaches to distribute debug information. Commands below will use two simple C files for demonstration.

```sh
cat > a.c <<eof
void foo(int);
int main() { foo(42); }
eof
cat > b.c <<eof
#include <stdio.h>
void foo(int x) { printf("%d\n", x); }
eof
```

<!-- more -->

## Debug information in the executable or shared object

This is the simplest model. Debug information resides in the executable or shared object.

```sh
gcc -c -g a.c b.c
gcc a.o b.o -o a
```

The linker collects input debug sections, resolves relocations, does minimum merging (`SHF_STRING` merge for `.debug_str` and `.debug_line_str`), and combines them into output debug sections.

```text
               0                0       a7     1 .debug_abbrev
               0                0       38     1         a.o:(.debug_abbrev)
              38               38       6f     1         b.o:(.debug_abbrev)
               0                0       94     1 .debug_info
               0                0       37     1         a.o:(.debug_info)
              37               37       5d     1         b.o:(.debug_info)
               0                0       44     1 .debug_str_offsets
               0                0       1c     1         a.o:(.debug_str_offsets)
              1c               1c       28     1         b.o:(.debug_str_offsets)
               0                0       c1     1 .debug_str
               0                0       c1     1         <internal>:(.debug_str)
               0                0       28     1 .debug_addr
               0                0       10     1         a.o:(.debug_addr)
              10               10       18     1         b.o:(.debug_addr)
               0                0       bf     1 .debug_line
               0                0       5e     1         a.o:(.debug_line)
              5e               5e       61     1         b.o:(.debug_line)
               0                0        f     1 .debug_line_str
               0                0        f     1         <internal>:(.debug_line_str)
```

## Separate debug files

Debug information is large and not needed by many people. As a general size optimization, many distributions don't provide debug information in main software packages.
For debugging needs, distributions may provide debug information in separate packages, leveraging a debugger feature that debug information may reside in a separate file. See <https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html>.

objcopy from binutils-gdb can create a separate debug file (since 2003).
```sh
objcopy --only-keep-debug a a.debug
strip -S a -o a.stripped
```

eu-strip from elfutils can generate two output files in one invocation.
```sh
eu-strip -f a.debug a -o a.stripped
```

The elfutils way is convenient for simple use cases and is adopted by rpm. But there is general ambiguity whether an operation applies to one output file or both.
The `--only-keep-debug` way is orthogonal and integrates well with other features (e.g. `--compress-debug-sections`, `--remove-section`).
I favor `--only-keep-debug` and [implemented it](https://reviews.llvm.org/D67137) in llvm-objcopy.

When debugging `a.stripped` in gdb, use `add-symbol-file -o xxx a.debug` to load the separate debug file.

Solaris 11 Update 1 introduced a similar feature called Ancillary Object. See [Ancillary Objects: Separate Debug ELF Files For Solaris](http://www.linker-aliens.org/blogs/ali/entry/ancillary_objects_separate_debug_elf/).
They made a nice choice that `ld -z ancilliary` creates two output files, saving one objcopy command in the GNU linking model.
However, if you want extra objcopy options beside `--only-keep-debug`, you will still need an objcopy command.

In July 2024, mold added [`--separate-debug-info`](https://github.com/rui314/mold/issues/1294), which is like a variant of Solaris ld's `-z ancilliary`.

### `.gnu_debuglink`

`objcopy --add-gnu-debuglink=a.debug a.stripped` adds a non-`SHF_ALLOC` section `.gnu_debuglink` to `a.stripped`.
The section contains a filename (no directory information) and a four-byte CRC checksum.
```text
% objdump -g a.stripped
...
Contents of the .gnu_debuglink section (loaded from a.stripped):

  Separate debug info file: a.debug
  CRC value: 0x4d1e2a66
```

gdb has [supported `.gnu_debuglink` since 2003](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=5b5d99cf4d5ded93b60fd64c6069d45e3eeab1d3). It finds a debug file in the same directory of the executable and `.debug/` relative to the directory.
(I filed a [feature request](https://sourceware.org/bugzilla/show_bug.cgi?id=29584) for supporting zstd in 2022-09.)
```text
% gdb -ex q a.stripped
Reading symbols from a.stripped...
Reading symbols from /tmp/c/a.debug...
```

Directories specified by `debug-file-directory` are used as well. This option needs to be set before loading the inferior.
```text
% pwd
/tmp/c
% install -D -t debug/tmp/c a.debug
% rm a.debug
% gdb -iex 'set debug-file-directory debug' -ex q a.stripped
Reading symbols from a.stripped...
Reading symbols from debug//tmp/c/a.debug...
```

A debug file can be found by build ID. A build ID resides in an ELF note section.
Many Linux distributions configure GCC with `--enable-linker-build-id` to generate a build ID by default.
See [--build-id](/blog/2020-11-15-explain-gnu-linker-options) for the option.

```text
% readelf -Wn a.stripped
...
Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000008       NT_GNU_BUILD_ID (unique build ID bitstring)         Build ID: a3b3f0788440fd94
% install -D -T debug/tmp/c/a.debug debug/.build-id/a3/b3f0788440fd94.debug
% rm debug/tmp/c/a.debug
% gdb -nx -iex 'set debug-file-directory debug' -ex q a.stripped
Reading symbols from a.stripped...
Reading symbols from /tmp/c/debug/.build-id/a3/b3f0788440fd94.debug...
```

lldb uses `target.debug-file-search-paths` to locate a separate debug file. (TODO)

On Debian, if we install `hello-dbgsym`, a debug file will be available in `/usr/lib/debug`.
```text
% gdb -batch -ex 'show debug-file-directory'
The directory where separate debug symbols are searched for is "/usr/lib/debug".
% gdb -ex q =hello
Reading symbols from /usr/bin/hello...
Reading symbols from /usr/lib/debug/.build-id/ff/29703f105c66821e9b10149db8cff3b2e4043a.debug...
```

Debian uses [`dh_strip`](https://github.com/Debian/debhelper/blob/master/dh_strip) for packaging commands.
`dh_strip` uses `objcopy --only-keep-debug --compress-debug-sections` to [compress debug sections](/blog/2022-01-23-compressed-debug-sections).

## MiniDebugInfo

See <https://sourceware.org/gdb/onlinedocs/gdb/MiniDebugInfo.html> ([implemented in 2012](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=608e2dbbfefcec9aa3efc863ffcc889786ae93d7)).
When a binary contains `.gnu_debugdata`, gdb decompresses it with xz and loads it.
```sh
objcopy --only-keep-debug a a.debug
xz a.debug
objcopy -S --add-section=.gnu_debugdata=a.debug.xz a a.stripped
```
```
% gdb -ex q a.stripped
Reading symbols from a.stripped...
Reading symbols from .gnu_debugdata for /tmp/c/a.stripped...
```

Fedora uses the feature to improve symbolization of stack traces in the absence of debug information.
A Fedora MiniDebugInfo file mostly just provides note sections and a `.symtab` with symbols not in `.dynsym`.
Non-`SHF_ALLOC` `SHT_PROGBITS/SHT_NOTE/SHT_NOBITS` sections (e.g. `.comment`) are removed.
Note sections are duplicated in the original binary and the MiniDebugInfo file due to a small missing optimization in `eu-strip -f`.

See [Support MiniDebugInfo in rpm](https://bugzilla.redhat.com/show_bug.cgi?id=834073) for the original implementation.
The new implementation is in `scripts/find-debuginfo.in` in the [`debugedit`](https://sourceware.org/debugedit/) repository.
[Support for mini-debuginfo in LLDB](https://archive.fosdem.org/2020/schedule/event/debugging_mini/) introduces the lldb implementation.

Here is a simplified demonstration of what `scripts/find-debuginfo.in` does:
```sh
eu-strip --remove-comment -f a.mini a -o a.stripped
nm a -f sysv --defined-only | awk -F \| '$4 ~ "FUNC" {print $1}' | sort > a.symtab
nm a -f sysv --defined-only -D | sort > a.dynsym
comm -13 a.dynsym a.symtab > a.keepsyms
objcopy -S --keep-symbols=a.keepsyms a.mini
xz a.mini
objcopy -S --add-section=.gnu_debugdata=a.mini.xz a a.stripped
```

## DWARF supplementary object files

A DWARF supplementary object file contains debug sections which can be referenced by multiple executable and shared object files.
dwz provides `-m file` to extracts duplicate debug information into a supplementary object file and rewrite input files to reference the supplementary object file.
The supplementary object file may contain debugging information entries, strings, and macro descriptions.

```sh
print '#include <stdio.h>\nint main() { puts("hello"); }' | gcc -g -gdwarf-5 -xc - -o 1
cp 1 2
dwz --dwarf-5 -m 3 1 2
```

`3` is a supplementary object file with a few `.debug_*` sections. Its `.debug_sup` uses `is_supplementary=1`.

`1` gets a new section `.debug_sup` with `is_supplementary=0`. It may use `DW_FORM_ref_sup4 (DW_FORM_GNU_ref_alt), DW_FORM_strp_sup (DW_FORM_GNU_strp_alt), DW_MACRO_define_sup, DW_MACRO_undef_sup, DW_MACRO_import_sup` attributes to reference the supplementary object file.
In this simple example only `DW_FORM_ref_sup4` and `DW_FORM_strp_sup` attributes are used.

Before standardization in DWARF v5, the special section name `.gnu_debugaltlink` is used.

## Split DWARF object files

This was originally proposed as GCC [debug fission](https://gcc.gnu.org/wiki/DebugFission). Later in DWARF version 5 this feature was standardized as split DWARF object files, commonly abbreviated as "split DWARF".
The idea is to move the bulk of `.debug_*` sections into a separate file (`.dwo`) and leave just a small amount in the relocatable object file (`.o`).
`.dwo` files are not handled by the linker: this reduces the input section combining work and relocation work for the linker, leading to smaller link time and lower memory usage.
The smaller input has advantages for a distributed build farm.

```sh
% clang -c -g -gsplit-dwarf a.c b.c
% readelf -WS a.o | grep ' .debug_gnu_pub'
  [12] .debug_gnu_pubnames PROGBITS        0000000000000000 0000be 00001c 00      0   0  1
  [14] .debug_gnu_pubtypes PROGBITS        0000000000000000 0000da 00001b 00      0   0  1
% readelf -WS a.dwo | grep ']'
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 000155 000051 00      0   0  1
  [ 2] .debug_str_offsets.dwo PROGBITS        0000000000000000 000040 00001c 00   E  0   0  1
  [ 3] .debug_str.dwo    PROGBITS        0000000000000000 00005c 00009d 01 MSE  0   0  1
  [ 4] .debug_info.dwo   PROGBITS        0000000000000000 0000f9 00002e 00   E  0   0  1
  [ 5] .debug_abbrev.dwo PROGBITS        0000000000000000 000127 00002e 00   E  0   0  1
% clang -fuse-ld=lld -Wl,--gdb-index a.o b.o -o a
```
(Note: don't perform compiling and linking in one action: the DWO files' location may be surprising.)

`-ggnu-pubnames` implied by `-gsplit-dwarf` generates `.debug_gnu_pubnames` and `.debug_gnu_pubtypes` sections.
(`-ggnu-pubnames` can be used without `-gsplit-dwarf`, but it is uncommon.)

`--gdb-index` supported by some ELF linkers (gold, ld.lld, mold) causes the linker to read `.debug_gnu_pubnames`, `.debug_gnu_pubtypes`, and other debug sections (`.debug_info`, `.debug_addr`, `.debug_ranges`, `.debug_rnglists`, etc) to construct `.gdb_index`.
`.debug_gnu_pubnames` and `.debug_gnu_pubtypes` are then discarded.

When gdb loads a symbol file, it constructs an internal symbol table. [`.gdb_index`](https://sourceware.org/gdb/onlinedocs/gdb/Index-Files.html) can improve startup time.
With split DWARF, the `.dwo` files as referenced by the executable or shared object will be fully parsed on demand.

Clang supports `-gsplit-dwarf=single` to embed `.dwo` sections in the relocatable object file. The compilation will not produce a `.dwo` file.
This mode is convenient for single-machine linking.

lldb uses `target.debug-file-search-paths` for `.dwo` searching but does not use a directory structure. (TODO)

Distributing a number of `.dwo` files can be inconvenient. A [DWARF package file](https://gcc.gnu.org/wiki/DebugFissionDWP) (typically given the extension `.dwp`) can be used in replace of `.dwo` files.
(Note: currently DWP has [incomplete DWARF64 support](https://dwarfstd.org/ShowIssue.php?issue=220708.2).)
dwp from binutils-gdb/gold can build a DWARF package file. [llvm-dwp](https://discourse.llvm.org/t/implementing-a-dwp-tool-in-llvm/38799) is another implementation in llvm-project, but there are some memory usage scaling problems.
```sh
dwp a.dwo b.dwo -o a.dwp
```

It is a TODO to integrate DWARF compressing (e.g. dwz) into dwp.

gold is under-maintained in recent years. dwp does not support DWARF v5 yet.

```sh
# Note: older Bazel needs --features=per_object_debug_info
bazel -c dbg --fission=yes :a  # bazel-bin/_objs/a/a.pic.dwo
bazel -c dbg --fission=yes :a.dwp  # bazel-bin/a.dwp
```

## debuginfod

While gdb can hint that debug information is missing (e.g. `Missing separate debuginfos, use: dnf debuginfo-install xxx`), the manual step of installing the relevant debug information package is considered by many as inconvenient.
In 2019, elfutils introduced a new program [debuginfod](https://sourceware.org/elfutils/Debuginfod.html). Some distributions have hosted debug information in public servers.
See <https://www.redhat.com/en/blog/how-debuginfod-project-evolved-2021>.

Here is an example demonstrating what debuginfod does:

```sh
objcopy --only-keep-debug a a.debug
strip -S a -o a.stripped
tar cf a.tar.zst --zstd a.stripped a.debug
debuginfod -d debuginfod.sqlite -F -Z .tar.zst=zstdcat .
```

debuginfod listens on port 8002 by default. `-F` causes it to scan archives in the specified directory.
`-Z .tar.zst=zstdcat` tells it to use zstdcat to handle a `.tar.zst` file (e.g. an Arch Linux package).

For every archive member, debuginfod classifies the file as an regular executable/shared object (with at least one `SHF_ALLOC SHT_PROGBITS` section) or a debug file (with a `.debug_*` section).
By default debuginfod parses a `.debug_line`

`debuginfod-find` is a client.
```sh
buildid=$(readelf -n a | awk '/Build ID:/ {print $3}')
DEBUGINFOD_URLS='http://localhost:8002/' debuginfod-find debuginfo $buildid
DEBUGINFOD_URLS='http://localhost:8002/' debuginfod-find executable $buildid
DEBUGINFOD_URLS='http://localhost:8002/' debuginfod-find source $buildid /tmp/c/a.c
```

We can query the server manually:
```sh
curl -s localhost:8002/buildid/$buildid/debuginfo -o output && cmp a.debug output
curl -s localhost:8002/buildid/$buildid/executable -o output && cmp a.stripped output
curl -s localhost:8002/buildid/a3b3f0788440fd94/source/$(<<</tmp/c/a.c sed 's,/,%2F,g') | diff /tmp/c/a.c -
```

With [`set debuginfod enabled on`](https://sourceware.org/gdb/onlinedocs/gdb/Debuginfod-Settings.html), gdb can query a debuginfod server if a symbol file is not found.
As of today, a few programs support debuginfod, e.g. valgrind.

Main change for Arch Linux: [debuginfod: Implement role](https://gitlab.archlinux.org/archlinux/infrastructure/-/commit/194bc84bf4c698be3a21ebae41d03f35fc7d1b4c)

In llvm-project, llvm-debuginfod is an alternative implementation. It requires `LLVM_ENABLE_HTTPLIB=on`.

## Microsoft SymbolStore

This is similar to debuginfod. See <https://github.com/dotnet/symstore/blob/main/docs/specs/SSQP_Key_Conventions.md>.

## Apple's "lazy" DWARF scheme

See <http://wiki.dwarfstd.org/index.php?title=Apple%27s_%22Lazy%22_DWARF_Scheme>.

Apple platforms use a model different from Linux distributions. Apple ld64 does not combine input `__debug_*` sections into output sections.
Instead, unless `-S` is specified, debug map entries are emitted into the symbol table to record source files and relocatable object files.
Among the entries, `N_OS` and `N_OSO` record source files and relocatable object files.
`N_FUN` gives value/size for a function. `N_GSYM`/`N_STSYM` describe a global/static variable symbol.

When debugging a program with lldb, lldb parses DWARF from dSYM bundles and (for `N_OSO` entries) relocatable object files.

```sh
clang -c -g a.c b.c
clang a.o b.o -o a
dsymutil a
```

dsymutil can create a dSYM bundle from relocatable object files. It takes an executable name or if `-y` is specified a YAML debug map.
dsymutil finds two relocatable object files `a.o` and `b.o`, combines and optimizes their DWARF information into a dSYM file.
In addition, dsymutil builds an accelerator table (`.apple_*` or `.debug_names`).

## Windows PDB

TODO

## Appendix: debug information size

<https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html#index-feliminate-unused-debug-types>
