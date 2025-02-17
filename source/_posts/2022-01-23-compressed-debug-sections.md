layout: post
title: Compressed debug sections
author: MaskRay
tags: [binutils,linker,llvm]
---

Updated in 2024-05.

Binary sizes are important. Filesystem compression is ergonomic but typically does not leverage application information well.
Compressing allocable sections (text, data) increases program startup time and introduces memory overhead.
In addition, filesystem compression is not sufficiently portable.

Debug sections are large and contribute to a significant portion of the binary size.
Therefore, it is appealing to compress debug sections.

Here is a `-DCMAKE_BUILD_TYPE=Debug` build directory of llvm-project where I just ran `ninja clang` (on 2022-10-21).
Here are the total sizes of .o files, text sections, and debug sections.
It is typical that the debug information is often much larger than text sections.

```text
% stat -c %s **/*.o | awk '{s+=$1} END{print s}'
1464767464
% readelf -WS **/*.o | awk 'BEGIN{FPAT="\\[.*?\\]|\\S+"} $2~/\.text/{d += strtonum("0x"$6)} END{print d}'
210026370
% readelf -WS **/*.o | awk 'BEGIN{FPAT="\\[.*?\\]|\\S+"} $2~/\.debug_/{d += strtonum("0x"$6)} END{print d}'
631069751
% readelf -WS **/*.o | awk 'BEGIN{FPAT="\\[.*?\\]|\\S+"} $2~/\.rela\.debug_/{d += strtonum("0x"$6)} END{print d}'
78448968
```

Some assemblers and linkers offer a feature to compress debug sections.

llvm-objcopy supports `--compress-debug-sections=zlib` to compress debug sections. We can use the option to check what if we compress debug sections for the assembler.
```text
% for i in **/*.o; do /tmp/Rel/bin/llvm-objcopy --compress-debug-sections=zlib $i /tmp/c/o && readelf -WS /tmp/c/o | awk 'BEGIN{FPAT="\\[.*?\\]|\\S+"} $2~/\.debug_/{d += strtonum("0x"$6)} END{print d}'; done | awk '{s+=$1} END{print s}'
161691798
```

For debug sections, we have a compression ratio of 3.90! The total .o size is 995438992 bytes, 68% of the original.

Then let's check zstd.
```text
% for i in **/*.o; do /tmp/Rel/bin/llvm-objcopy --compress-debug-sections=zstd $i /tmp/c/o && readelf -WS /tmp/c/o | awk 'BEGIN{FPAT="\\[.*?\\]|\\S+"} $2~/\.debug_/{d += strtonum("0x"$6)} END{print d}'; done | awk '{s+=$1} END{print s}'
159341878
```

To check whether an object file has compressed debug sections, we can use `readelf`.

```
% readelf -S a.o
...
Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
...
  [ 5] .debug_abbrev     PROGBITS        0000000000000000 000080 000087 00   C  0   0  8
```

In the `readelf -S` output, the `Flg` column describes `sh_flags` where `C` indicates the `SHF_COMPRESSED` flag.

```
% readelf -t a.o
...
Section Headers:
  [Nr] Name
       Type            Address          Off    Size   ES   Lk Inf Al
       Flags
...
  [ 5] .debug_abbrev
       PROGBITS        0000000000000000 000080 000087 00   0   0  8
       [0000000000000800]: COMPRESSED
       ZLIB, 0000000000000093, 1
```

## History

In 2007-11, Craig Silverstein [added `--compress-debug-sections=zlib` to gold](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=9a0910c33e1a6962d475ee0a994fd1f5e446a888).
When the option was specified, gold compressed the content of a `.debug*` section with zlib and changed the section name to `.debug*.zlib.$uncompressed_size`.

<!-- more -->

In 2008-04, Craig Silverstein [changed the format](https://sourceware.org/pipermail/binutils/2008-April/055837.html) and contributed [Patch to handle compressed sections](https://sourceware.org/pipermail/gdb-patches/2008-March/056449.html) to gdb.
The compressed section was renamed to `.zdebug*`.

In 2010-06, Cary Coutant [added `--compress-debug-sections` to gas](https://sourceware.org/pipermail/binutils/2010-June/067606.html) and added reading support to objdump and readelf.

[ELF Section Compression](http://www.linker-aliens.org/blogs/ali/entry/elf_section_compression/) has a nice summary of the `.zdebug` format.
The article lists some problems with the format which led to a new format standardized by the generic ELF ABI in 2012.
I recommend that folks interested in the ELF format read this article.
My thinking of implementing ELF features has been influenced by profound discussions like this article and other discussions on the generic ABI mailing list.

In Solaris 11.2, its linker introduced `-z compress-sections` to compress candidate sections.

The generic ABI format led to modification to the existing assembler and linker options in binutils.
In 2015-04, H.J. Lu [added `--compress-debug-sections=[none|zlib|zlib-gnu|zlib-gabi]` to gas](https://sourceware.org/pipermail/binutils/2015-April/088496.html) and [added `--compress-debug-sections=[none|zlib|zlib-gnu|zlib-gabi]` to GNU ld](https://sourceware.org/pipermail/binutils/2015-April/088607.html).
In 2015-07, H.J. Lu [added `--compress-debug-sections=[none|zlib|zlib-gnu|zlib-gabi]` to gold](https://sourceware.org/bugzilla/show_bug.cgi?id=18322).
`zlib` and `zlib-gnu` indicated the `.zdebug` format while `zlib-gabi` indicated the generic ABI format.

In 2015-07, [[PATCH] Make default compression gABI compliant](https://sourceware.org/pipermail/binutils/2015-July/089559.html) (milestone: binutils 2.26) changed `zlib` to indicate the generic ABI format.

The assembler and linker `--compress-debug-sections=` options are long and difficult to use.
In 2014-06, Rainer Orth [added `-gz` and `-gz=[none|zlib|zlib-gnu]` to GCC](https://sourceware.org/pipermail/gcc-patches/2014-June/390257.html).

In 2022, I filed a `ELFCOMPRESS_ZSTD` proposal to generic-abi and it was accepted.
See [zstd compressed debug sections](/blog/2022-09-09-zstd-compressed-debug-sections).

## `.zdebug` remnant

Go 1.11 enabled `.zdebug` by default.
I filed <https://github.com/golang/go/issues/50796> after I realized Go was still using the legacy format. Meng Zhuo migrated it away in 2022-03.

## Usage

To use compressed debug sections, just remember one GCC/Clang driver option `-gz`(or a variant like `-gz=zlib`). The option combines two tasks.

### Assembling

```text
% clang -c -g -gz a.c
% clang -c -g -gz=zlib a.c
```

For object generation, `-gz=zlib` acts as `-Wa,--compress-debug-sections=zlib` and asks the assembler to compress debug sections in the output `.o` (and `.dwo` when `-gsplit-dwarf` is specified).
A compressed section has the `SHF_COMPRESSED` flag and its content begins with a compression header structure that identifies the compression algorithm.
```c
typedef struct {
	Elf32_Word	ch_type;
	Elf32_Word	ch_size;
	Elf32_Word	ch_addralign;
} Elf32_Chdr;

typedef struct {
	Elf64_Word	ch_type;
	Elf64_Word	ch_reserved;
	Elf64_Xword	ch_size;
	Elf64_Xword	ch_addralign;
} Elf64_Chdr;
```

As of 2022, only `ELFCOMPRESS_ZLIB` and `ELFCOMPRESS_ZSTD` are defined:

> ELFCOMPRESS_ZLIB - The section data is compressed with the ZLIB algorithm. The compressed ZLIB data bytes begin with the byte immediately following the compression header, and extend to the end of the section. Additional documentation for ZLIB may be found at http://zlib.net.
>
> ELFCOMPRESS_ZSTD - The section data is compressed with the Zstandard algoritm. The compressed Zstandard data bytes begin with the byte immediately following the compression header, and extend to the end of the section. Additional documentation for Zstandard may be found at http://www.zstandard.org

```text
% clang -c -g -gz a.c
% clang -c -g -gz=zlib a.c
```

Note that `-gz` does not imply emit debug information, which is controlled by `-g`. So you generally need both `-g -gz`.
This feature orthogonality is convenient when a subset of object files need to opt out debug information emission.

### Linking

```text
% clang -gz a.o
% clang -gz=zlib a.o
```

For linking, `-gz` acts as `-Wl,--compress-debug-sections=zlib` and asks the linker to compress debug sections in the linked image.
If you need decompression for linker input but don't need compression for the linker output, don't bother with `-gz`.
The linker recognizes compressed input and decompresses it automatically.

You may not want to use `-gz` if you combine assembly and linking in one step (`gcc -g -gz a.c` without `-S` or `-c`).
The intermediate `.o` file will be discarded. The assembler compressed debug sections will immediately be decompressed by the linker, causing wasted efforts.

binutils and GCC support `zlib-gabi`. This was added to parallel the renamed legacy format `zlib-gnu`.
Nowadays just use `zlib` and avoid `zlib-gabi`. ELF compresion has been standardized and new formats (e.g. zstd) do not use `-gabi`.

## Evaluation

I tested two `-DCMAKE_BUILD_TYPE=Debug` builds of clang.
`clang.dw4` is the uncompressed DWARF v4 build while `clang.dw5` is the uncompressed DWARF v5 build.
`objcopy --compress-debug-sections` compresses `.debug*` sections.
The effect is as if the linker was invoked with `--compress-debug-sections=zlib`.

```text
% objcopy --compress-debug-sections clang.dw4 clang.dw4.compressed
% objcopy --compress-debug-sections clang.dw5 clang.dw5.compressed
% bloaty clang.dw4.compressed -- clang.dw4
...
 -89.7%  -572Ki  [ = ]       0    .debug_loc
 -80.2% -3.33Mi  [ = ]       0    .debug_abbrev
 -92.6% -35.3Mi  [ = ]       0    .debug_ranges
 -72.9% -92.3Mi  [ = ]       0    .debug_line
 -84.2%  -175Mi  [ = ]       0    .debug_str
 -57.7%  -221Mi  [ = ]       0    .debug_info

% bloaty clang.dw5.compressed -- clang.dw5
...
 -69.8%    -572  [ = ]       0    .debug_ranges
 -63.4%  -134Ki  [ = ]       0    .debug_loclists
 -81.8%  -162Ki  [ = ]       0    .debug_line_str
 -32.2% -3.27Mi  [ = ]       0    .debug_rnglists
 -78.4% -4.49Mi  [ = ]       0    .debug_abbrev
 -33.2% -16.7Mi  [ = ]       0    .debug_str_offsets
 -88.8% -19.2Mi  [ = ]       0    .debug_addr
 -72.9% -89.8Mi  [ = ]       0    .debug_line
 -84.2%  -175Mi  [ = ]       0    .debug_str
 -55.9%  -177Mi  [ = ]       0    .debug_info
```

There are some interesting points about the compression ratios.

* `.debug_str` is a string table. It consists of mostly identifier characters and compresses really well.
* pre-DWARF-v5 `.debug_loc` and `.debug_ranges` compress well because their encodings are relatively inefficient.
* Compressed `.debug_rnglists` is larger than compressed `.debug_ranges`. `.debug_rnglists` uses a manually tuned byte-oriented encoding, which does not LZ77 here:)
* `.debug_info` does not compress well. This is probably not too bad because the raw encoding is pretty compact.

`-gsplit-dwarf{,=split}` moves a large portion of `.debug_{abbrev,info,loclists,rnglists,str,str_offsets}` (and pre-DWARF-v5 `.debug_loc`) into `.dwo` files.
For symbolization without `.dwo` files, the large `.debug_line` remains in `.o` files. `-gz` is still useful.

## Alignment

The generic ABI specifies

> The sh_size and sh_addralign fields of the section header for a compressed section reflect the requirements of the compressed section.

A rigid interpretation says that compressed `.debug_*` sections should have an alignment of 8 for ELFCLASS64.
BFD does uses 8-byte alignment while gold/lld/mold use 1-byte alignment. I do not know how the Solaris linker sets the alignment.

It looks like 1-byte alignment has been wildly accepted and utilized for a long time.
Removing alignment padding is particularly useful when there are many compressed output sections.

## Compressing more sections

In 2021-02, I filed a feature request [ld: Support compressing arbitrary sections (generalized --compress-debug-sections=)](https://sourceware.org/bugzilla/show_bug.cgi?id=27452).
This has not seen immediate use cases.

## Pros and cons

The pros are obvious: compression decreases sizes for either (or both) relocatable object files and linked images.

Compression imposes memory usage costs at many stages of development. The assembler, the linker, and the debugger need to allocate additional memory to hold the compressed or uncompressed data.

For object generation, the performance depends on the environment.
On one hand, compression performs more work and creates decompression work for the linker.
On the other hand, compressed data is smaller, so I/O is faster.
I tested some C++ source files. Other phases of the compiler dominate and `-gz` seems to barely affect the compile time.

For linking, compression may greatly increase link time.
In a project, a `.o` file content may go into multiple linked images.
The same data may be compressed multime times but it is nearly impossible to reuse a compression result because every linked image has a different post-relocation content.

In addition, compressed debug sections may make the debugger significantly slower.
Neither gdb nor lldb decompresses debug sections parallelly.

If you care about the file size and debugging is rare, compressed debug sections in linked images may be good to have.

## Linkers

zlib uses the DEFLATE compressed data format, which was not designed for parallel decompression.
Fortunately the linker does not need to deal with this task, because decompressing input sections are embarrassingly parallel.

However, compressing input sections is a major bottleneck.
I tested a `-DCMAKE_BUILD_TYPE=Debug` build of clang with 265MiB `SHF_ALLOC` sections and 920MiB uncompressed debug sections.
If I specify `--compress-debug-sections=zlib` in a `--threads=1` link, "Compress debug sections" takes 2/3 time.
In a `--threads=8` link, "Compress debug sections" takes nearly 70% time.

We basically have four choices to improve the situation.

* tune zlib parameters
* alternative compression format
* more optimized library
* parallel divide-and-conquer

The first has been done ([D70658](https://reviews.llvm.org/D70658)). ld.lld switched to compression level 1 (`Z_BEST_SPEED`) for `-O1` (default) and `-O0`.
The previous default (6) was known to not decrease much size while taking too much time.
From lld 19 onwards, the [default compression level for zlib](https://github.com/llvm/llvm-project/pull/90567) is `Z_BEST_SPEED`, independent of `-O` options.

For the second choice: using a better format is an ecosystem issue that requires significant undertaking and stakeholder buy-in.
zstd is better than zlib in all metrics: compression speed, decompression speed, and compression ratio.
However, it was not standardized (the only flag the generic ABI specifies was `ELFCOMPRESS_ZLIB`). It did not have debug producer/consumer support.

The third choice is difficult for a linker like lld.
Importing a library to llvm-project has a significantly high barrier.
A new CMake configuration has bad discoverability and benefits very few groups.
libdeflate is efficient and seems to do a good job, but I do not know how we can justify importing it into llvm-project.

The fourth choice is feasible.
Rui Ueyama told me that mold optimizes `--compress-debug-sections=zlib` with sharding.
I researched a bit. pigz has a great comment about how it leverages multi-threading: <https://github.com/madler/pigz/blob/master/pigz.c>.
It has some sophisticated features that a linker does (may) not need:

* unless `-i` is specified, the last 32KiB (window size) from the previous chunk is used as a dictionary (`deflateSetDictionary`) to improve compression ratio.
* sync marker even in the absence of `--rsyncable`

I submitted [[ELF] Parallelize --compress-debug-sections=zlib](https://reviews.llvm.org/D117853) (milestone: LLD 14.0.0) to improve the ld.lld algorithm.
By divide-and-conquer, we need to stitch compressed streams together and add the zlib header and the trailer (checksum).
The zlib stream has the following structure per ([RFC1950](https://datatracker.ietf.org/doc/html/rfc1950)).
We don't use a preset dictionary, so a field after the 2-byte header is omitted.

```text
byte 0: CMF
byte 1: FLG
compressed data
last 4 bytes: Alder-32 checksum
```

The compressed data is encoded in the DEFLATE compressed data format (RFC1951). It consists of several shards.
```text
shard 0: blocks produced by deflate(&strm0, Z_SYNC_FLUSH)
shard 1: blocks produced by deflate(&strm1, Z_SYNC_FLUSH)
...
last shard: blocks produced by deflate(&strmN_1, Z_FINISH)
```

DEFLATE blocks are a bit sequence. We need to ensure every shard starts at a byte boundary for concatenation.
We use `Z_SYNC_FLUSH` for all shards but the last to flush the output to a byte boundary.
(`Z_FULL_FLUSH` can be used as well, but `Z_FULL_FLUSH` clears the hash table which just wastes time.)

The last block requires the BFINAL flag.
We call `deflate` with `Z_FINISH` to set the flag as well as flush the output to a byte boundary.
Under the hood, all of `Z_SYNC_FLUSH`, `Z_FULL_FLUSH`, and `Z_FINISH` emit a non-compressed block (called stored block in zlib).
RFC1951 says "Any bits of input up to the next byte boundary are ignored."

After compressed data, the last 4 bytes of the zlib stream are an Adler-32 checksum.
`adler32_combine` can combine two Adler-32 checksums, i.e. `adler32_combine(adler32(A), adler32(B)) = adler32(cat(A, B))`.

The final step is to write DEFLATE blocks in parallel.

Can we do better? At one time, the compressed data is stored in two places.
One in the allocated memory holding the compressed shard, the other in the memory mapped output file.
It will be nice if we can avoid memory allocation.
Unfortunately we need to compute the section size, otherwise we do not know the offsets of following sections and the section header table (which is usually placed after the last section).
There is no good way estimating the compressed section size without doing the compression.
Technically if the section header table along with `.symtab`/`.shstrtab`/`.strtab` is moved before debug sections, we can compress the debug compression and append them to the output file.
The output file will unfortunately be unconventional and this will not work when a linker script specifies exact orders of sections.

```text
SECTIONS {
  ...
  .debug_info 0 : { *(.debug_info) }
  .strtab 0 : { *(.strtab) }
```

It is just too hacky to do so much to just save a little memory.
In addition, `.symtab`/`.shstrtab`/`.strtab`/section header table at the end of the file has the nice property that an optimized strip program can alter just these bytes if it perform symbol-only operations, though I don't know any strip leverages this property.

I [notified binutils](https://sourceware.org/bugzilla/show_bug.cgi?id=28812) about using a parallel implementation.
dwp packages multiple `.dwo` files and performs a linker-like task. dwp does not support `--compress-debug-sections` yet but I do now know whether anyone has such a need.

[dwz](https://sourceware.org/dwz/) is a DWARF optimization and duplicate removal tool.
There is a feature request for [adding `--compress-debug-sections`](https://sourceware.org/bugzilla/show_bug.cgi?id=24822).

## binutils configuration

`--enable-compressed-debug-sections=all` changes gas and ld to default to `--compress-debug-sections=zlib`.

### More ideas

Q: For compressed input and uncompressed output, can we avoid temporary buffer holding the uncompressed output?

It's possible but there will be difficulties in resolving relocations.
My estimation is that avoiding the temporary buffer may not provide significant speed-up/memory savings.

Q: For compressed input and compressed output, can we not re-compress its data?

The answer is yes in the absence of relocations, but the changes are involved and it may not be a worthwhile trade-off.
zlib provides [examples/gzjoin.c](https://github.com/madler/zlib/blob/master/examples/gzjoin.c) which concatenates multiple gzip strems (gzip uses DEFLATE as well) without re-compression.
It needs to decompress input to locate the last block in each input stream.
If a linker leverage this optimization:

* it needs to port the involved code
* it needs to retain compressed data. This is difficult in ld.lld because features like `--gdb-index` will update the `InputSection` member (originally pointing to the compressed data) to point to the uncompressed data. Retaining both will need an extra member in the section representation, increasing memory usage.

Q: If storage systems offer built-in compression, is debug section compression still useful?

Such a comparison will be useful. I have not heard that anyone has a quantitative comparison.

The storage can do compression, but applications can usually make better decisions.

* File transfer overhead: Storage systems may need to compress or decompress data multiple times, potentially consuming more CPU resources than toolchain compression.
* Efficiency: Toolchains can interleave compression with other tasks, leading to better CPU utilization. In particular, the linker can leverage parallelism and interleave compression with other passes.
* More informed decisions: Toolchain-level compression has access to more contextual information about the data being compressed (debug info vs code/data, build configurations, etc), allowing for better compression strategies.
* Accounting purposes: Distributed build systems might pose limits on uncompressed input sizes. Adapting these systems to count compressed input might require significant modifications, and the feasibility is uncertain.
