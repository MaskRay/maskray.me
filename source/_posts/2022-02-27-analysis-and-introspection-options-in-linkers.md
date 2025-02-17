layout: post
title: Analysis and introspection options in linkers
author: MaskRay
tags: [linker,llvm]
---

## Symbol related

### `--trace-symbol=<sym>`

Alias: `-y sym`

<!-- more -->

Print files which define or reference the specified non-local symbol.
There are three types of definitions: definition in a relocatable object file, definition in a shared object, definition in a lazy object file/archive.

```text
% ld.lld -y foo a1.a a.o a2.so
a1.a(a1.o): lazy definition of foo
a.o: reference to foo
a1.a(a1.o): definition of foo
a2.so: shared definition of foo
```

## File related

### `--trace`

Alias: `-t`

Print processed files: relocatable object files, shared objects, and extracted lazy object files/archive members.
Note that unextracted lazy object files/archive members are not printed.

The GNU ld behavior is a bit strange. You need to specify `-t` twice to get both the archive name and the member name.

```text
% ld.lld -t a.o b.so c.a
a.o
b.so
c.a(1.o)
% ld.bfd -t a.o b.so c.a
a.o
b.so
c.a
% ld.bfd -t -t a.o b.so c.a
a.o
b.so
(c.a)1.o
```

## Archive related

### `--print-archive-stats=<file>`

gold introduced `--print-symbol-counts=<file>` in 2008-06. The output includes the member count and extracted member count for each archive.
I added `--print-archive-stats=<file>` to ld.lld to dump the archive information, but in a tab-separated format.

```text
% ld.lld a.o aweak.a a1.a a1.a --print-archive-stats=-
members extracted       archive
1       0       aweak.a
3       2       a1.a
3       0       a1.a
```

### `--why-extract=<file>`

Print why each archive member/lazy object file is extracted. I added the option to ld.lld 14.
The output is in a tab-separated format.

```text
% ld.lld main.o a_b.a b_c.a c.a -o /dev/null --why-extract=- | tee stdout
reference       extracted       symbol
main.o  a_b.a(a_b.o)    a
a_b.a(a_b.o)    b_c.a(b_c.o)    b()
b_c.a(b_c.o)    c.a(c.o)        c()
```

It is easy to track a chain of references to one archive member:
```
% ruby -ane 'BEGIN{p={}}; p[$F[1]]=[$F[0],$F[2]] if $.>1; END{x="c.a(c.o)"; while y=p[x]; puts "#{y[0]} extracts #{x} to resolve #{y[1]}"; x=y[0] end}' stdout
b_c.a(b_c.o) extracts c.a(c.o) to resolve c()
a_b.a(a_b.o) extracts b_c.a(b_c.o) to resolve b()
main.o extracts a_b.a(a_b.o) to resolve a
```

ld64 has a similar option named `-why_load`.

## `-Map=<file>`

Print a link map to the file. To print to stdout, use `-M`.

The output includes output sections addresses, file offsets, input section to output section mapping, and symbol assignments.

GNU ld prints additional information: `Archive member included to satisfy reference by file (symbol)`, `Discarded input sections` (similar to `--print-gc-sections`), `Allocating common symbols`, `Memory Configuration` (related to memory regions), and `Linker script and memory map` (similar to `-t`).

If the option value is a directory or a name with `%`, the map filename will be constructed from the output filename.
```text
 -o foo.exe -Map=bar                  [Creates ./bar]
 -o ../dir/foo.exe -Map=bar           [Creates ./bar]
 -o foo.exe -Map=../dir               [Creates ../dir/foo.exe.map]
 -o ../dir2/foo.exe -Map=../dir       [Creates ../dir/foo.exe.map]
 -o foo.exe -Map=%                    [Creates ./foo.exe.map]
 -o ../dir/foo.exe -Map=%             [Creates ../dir/foo.exe.map]
 -o foo.exe -Map=%.bar                [Creates ./foo.exe.bar]
 -o ../dir/foo.exe -Map=%.bar         [Creates ../dir/foo.exe.bar]
 -o ../dir2/foo.exe -Map=../dir/%     [Creates ../dir/../dir2/foo.exe.map]
 -o ../dir2/foo.exe -Map=../dir/%.bar [Creates ../dir/../dir2/foo.exe.bar]
```

Some folks want a JSON output format with some stability guarantee.

## Cross references

### `--cref`

Print a cross reference table.
For each non-local defined or shared symbol, print the defined file on the first line and and referencing files in subsequent lines.
The format is a bit wasteful because there are 50 bytes before the `File` column.

```text
% ld.lld --cref a1.so a2.o a3.o a.a

Cross Reference Table

Symbol                                            File
foo                                               a1.so
                                                  a2.o
                                                  a3.o
bar                                               a2.o
                                                  a3.o
_start                                            a3.o
baz                                               a3.o
zed                                               a.a(aa.o)
                                                  a3.o
```

If `-Map` is specified, print along with the link map to the file.
I find this behavior a bit unfortunate, because:

* the information is independent from other pieces of `-Map`
* both pieces of information has large amount of output. Merging them makes the link action slower if the user just needs one piece of information

### `--print-dependencies`

mold recently added the option in an alternative format to `--cref`.

```text
# This is an output of the mold linker's --print-dependencies=full option.
#
# Each line consists of 4 fields, <input-section>, <output-section>,
# <symbol-type> and <symbol>, separated by tab characters. It indicates that
# <input-section> depends on <output-section> to use <symbol>. <symbol-type>
# is either "u" or "w" for regular or weak undefined, respectively.
#
# If you want to obtain dependency information per function granularity,
# compile source files with the -ffunction-sections compiler flag.
a3.o:(.text)    a1.so   u       foo
a3.o:(.text)    a2.o:(.text)    u       bar
a3.o:(.text)    a.a(aa.o):(.text)       u       zed
```

### `-why_live sym`

ld64 manpage says: `Logs a chain of references to symbol_name.  Only applicable with -dead_strip .  It can help debug why something that you think should be dead strip removed is not removed.`

This can usually be approximated by `ld.lld --why-extract=-`.

```
% ld.lld main.o a_b.a b_c.a c.a -o /dev/null --why-extract=- | tee stdout
reference       extracted       symbol
main.o  a_b.a(a_b.o)    a
a_b.a(a_b.o)    b_c.a(b_c.o)    b()
b_c.a(b_c.o)    c.a(c.o)        c()

% ruby -ane 'BEGIN{p={}}; p[$F[1]]=[$F[0],$F[2]] if $.>1; END{x="c.a(c.o)"; while y=p[x]; puts "#{y[0]} extracts #{x} to resolve #{y[1]}"; x=y[0] end}' stdout
b_c.a(b_c.o) extracts c.a(c.o) to resolve c()
a_b.a(a_b.o) extracts b_c.a(b_c.o) to resolve b()
main.o extracts a_b.a(a_b.o) to resolve a
```

## Statistics

### `--stats`

```text
% ld.bfd @response.txt --stats
ld.bfd: total time in link: 9.879890
% mold @response.txt --stats
   total_input_bytes=1161772844
         reloc_alloc=3983368
            all_syms=1185762
        defined_syms=747420
      input_sections=692453
      undefined_syms=397776
    regular_sections=319069
      merged_strings=214750
            num_fdes=169858
  removed_comdat_mem=140402
             comdats=131977
            dso_syms=10875
         parsed_objs=3099
            num_objs=2677
            num_cies=2665
       output_chunks=36
            num_dsos=10
     num_unique_cies=2
    string_fragments=0
      reloc_nonalloc=0
.rodata.cst estimation=5425 actual=4689
.rodata.str estimation=240124 actual=210057
.comment estimation=1628 actual=4
```

### `--time-trace`

```text
% ld.lld @response.txt --time-trace -o clang
% jq -r '.traceEvents[] | select(.name|contains("Total")) | "\(.dur/1000000) \(.name)"' < clang.time-trace
1.099133 Total ExecuteLinker
1.064616 Total Link
0.303137 Total Write output file
0.295151 Total Write sections
0.220898 Total Scan relocations
0.183298 Total Parse input files
0.06039 Total Merge/finalize input sections
0.0377 Total Add local symbols
...
% jq -r '.traceEvents[] | select(.name|contains("Write")) | "\(.dur/1000000) \(.name) \(.args)"' < clang.time-trace
0.130966 Write sections {"detail":".rela.dyn"}
0.001995 Write sections {"detail":".rela.plt"}
0.007248 Write sections {"detail":".dynsym"}
0.000967 Write sections {"detail":".gnu.hash"}
0.01489 Write sections {"detail":".hash"}
0.003147 Write sections {"detail":".dynstr"}
0.034617 Write sections {"detail":".rodata"}
0.030412 Write sections {"detail":".eh_frame"}
0.02487 Write sections {"detail":".text"}
...
```

### `mold --perf`

```text
% mold @response.txt --perf
     User   System     Real  Name
    4.161    0.533    0.254  all
    2.077    0.302    0.139    read_input_files
    2.100    0.235    0.114    total
    1.537    0.129    0.071      before_copy
    0.000    0.000    0.000        apply_exclude_libs
    0.142    0.051    0.007        do_resolve_symbols
    0.142    0.051    0.007          do_resolve_symbols
    0.245    0.012    0.013        register_section_pieces
    0.002    0.000    0.001        eliminate_comdats
    0.002    0.000    0.000        convert_common_symbols
...
```
