layout: post
title: DWARF in reproducible builds
author: MaskRay
tags: [debug,gcc,llvm]
---

[Deterministic builds with clang and lld](https://blog.llvm.org/2019/11/deterministic-builds-with-clang-and-lld.html) describes several degrees of build determinism. Here is the description of local determinism:

> Like incremental basic determinism, but builds are also independent of the name of the build directory. Builds of the same source code on the same machine produce exactly the same output every time, independent of the location of the source checkout directory or the build directory.

<!-- more -->

This is often a stretch goal for a build system. For example, many autotools builds use absolute paths. CMake's Ninja generator uses absolute paths (<https://gitlab.kitware.com/cmake/cmake/-/issues/13894>).

Options discouraged by the post `-fdebug-prefix-map`, `-ffile-prefix-map`, and `-fmacro-prefix-map` are useful for such build systems.

## `-fdebug-compilation-dir=`

This option overrides the compilation directory in DWARF. `-fdebug-compilation-dir=.` is commonly used to set the compilation directory to the special directory `.`.
When input files are specified as relative paths, `-fdebug-compilation-dir=.` suffices.

```sh
% clang -c -g -fdebug-compilation-dir=. ../src/a.c
% llvm-dwarfdump a.o | grep -E '_TAG|_name|comp_dir|_decl_file'
0x0000000c: DW_TAG_compile_unit
              DW_AT_name        ("../src/a.c")
              DW_AT_comp_dir    (".")
0x00000023:   DW_TAG_subprogram
                DW_AT_name      ("main")
                DW_AT_decl_file ("./../src/a.c")
0x00000032:   DW_TAG_base_type
                DW_AT_name      ("int")
```

The environment variable `PWD`, if exists and is an absolute path, is used by GCC and Clang as a fallback when `-fdebug-compilation-dir=` is unspecified.
Therefore, we can invoke `PWD=/proc/self/cwd clang ...`.

## `-fdebug-prefix-map=`

See [Debug info path remapping](https://sourceware.org/pipermail/gcc-patches/2007-July/220228.html) for the original GCC patch.
See <https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html> for the documentation.

In Clang, the implementation requires changes to Clang CodeGen (mainly implemented by [Support Debug Info path remapping](436256a71316a1e6ad68ebee8439c88d75f974e9)), assembler (<https://reviews.llvm.org/D48988> <https://reviews.llvm.org/D48989>).

## Generated debug information for assembly source

When an assembler source file has no debug information/`.file` directive, GNU assembler synthesizes a `DW_TAG_compile_unit` entry.
This entry is generated when an assembler source file has at least one `.loc` directive without a `.debug_info` section.
The entry is mainly used to provide address ranges for the file.

```
% cat a.s
.globl  main
.type   main,@function
main:
  ret
.size   main, .-main
% gcc -g -c a.s
% llvm-dwarfdump a.o
a.o:    file format elf64-x86-64

.debug_info contents:
0x00000000: Compile Unit: length = 0x00000024, format = DWARF32, version = 0x0005, unit_type = DW_UT_compile, abbr_offset = 0x0000, addr_size = 0x08 (next unit at 0x00000028)

0x0000000c: DW_TAG_compile_unit
              DW_AT_stmt_list   (0x00000000)
              DW_AT_low_pc      (0x0000000000000000)
              DW_AT_high_pc     (0x0000000000000001)
              DW_AT_name        ("a.s")
              DW_AT_comp_dir    ("/tmp/c")
              DW_AT_producer    ("GNU AS 2.38")
              DW_AT_language    (DW_LANG_Mips_Assembler)
```

LLVM integrated assembler has a similar behavior. See <https://github.com/llvm/llvm-project/issues/37398> for the original feature request.

<https://github.com/llvm/llvm-project/issues/56609> reported that `-fdebug-prefix-map` did not work for generated DWARF v5 debug information.
There were two issues.

* `DW_AT_comp_dir` was not mapped. Implemented by <https://reviews.llvm.org/D131749>.
* `DW_TAG_compile_unit`'s `DW_AT_name` (also sued by `DW_TAG_label`'s `DW_AT_decl_file`) was not mapped. Implemented by <https://reviews.llvm.org/D131848>.
