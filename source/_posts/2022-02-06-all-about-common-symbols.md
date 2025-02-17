layout: post
title: All about COMMON symbols
author: MaskRay
tags: [binutils,linker,llvm]
---

## Programming language behavior

FORTRAN 77 COMMON blocks compiled to COMMON symbols.
You could declare a COMMON block in more than one file, with each specifying the number, type, and size of the variable.
The linker allocated enough space to satisfy the largest size.

<!-- more -->

This feature was somehow ported to C.
Unix C compilers traditionally permitted a variable using tentative definition in different compilation units and the linker would allocate enough space without reporting an error.

This behavior is constrast to both C and C++ standards, but GCC and Clang traditionally defaulted to `-fcommon` for C.
GCC since [10](https://gcc.gnu.org/gcc-10/changes.html) and Clang since [11](https://releases.llvm.org/11.0.0/tools/clang/docs/ReleaseNotes.html) default to `-fno-common`.

```c
% echo 'int x;' > a.c
% gcc -S -fcommon a.c -o - | grep -w x
        .comm   x,4,4
% gcc -S -fno-common a.c -o - | grep -w x
        .globl  x
        .type   x, @object
        .size   x, 4
x:
```

## Assembler behavior

The directive `.comm identifier, size[, alignment]` instructs the assembler to define a COMMON symbol with the specified size and the optional alignment.

In the ELF object file format, the symbol is represented as a `STT_OBJECT` `STB_GLOBAL` symbol whose `st_shndx` field holds `SHN_COMMON`.
In readelf, the `SHN_COMMON` value is shown as `COM`.

```c
typedef struct {
  Elf32_Word    st_name;
  Elf32_Addr    st_value;
  Elf32_Word    st_size;
  unsigned char st_info;
  unsigned char st_other;
  Elf32_Half    st_shndx;
} Elf32_Sym;

typedef struct {
  Elf64_Word    st_name;
  unsigned char st_info;
  unsigned char st_other;
  Elf64_Half    st_shndx;
  Elf64_Addr    st_value;
  Elf64_Xword   st_size;
} Elf64_Sym;
```

The `st_value` field holds the alignment. This is an interesting abuse.
Regular definitions are relative to a section (`st_value` is a section offset) and the section alignment (`sh_addralign`) is sufficient to encode the symbol alignment information.
For COMMON symbols, the section information is unavailable but fortunately `st_value` is vacant.

```sh
% cat a.s
.comm x,8,4  # size=8, alignment=4
% as a.s -o a.o
% readelf -Ws a.o

Symbol table '.symtab' contains 2 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000004     8 OBJECT  GLOBAL DEFAULT  COM x
```

The binding `STB_WEAK` is not allowed. Other types are not allowed:
```sh
% >err.s cat <<e
.comm x,4,4
.weak x
e
% as err.s
err.s: Assembler messages:
err.s: Error: symbol `x' can not be both weak and common
% >err.s cat <<e
.comm x,4,4
.type x,@function
e
% as err.s
err.s: Assembler messages:
err.s:2: Error: cannot change type of common symbol 'x'
```

The generic ABI supports `STT_COMMON` as another way to label a COMMON symbol. It says:

> Symbols with type `STT_COMMON` label uninitialized common blocks. In relocatable objects, these symbols are not allocated and must have the special section index `SHN_COMMON` (see below). In shared objects and executables these symbols must be allocated to some section in the defining object.
>
> In relocatable objects, symbols with type `STT_COMMON` are treated just as other symbols with index `SHN_COMMON`. If the link-editor allocates space for the `SHN_COMMON` symbol in an output section of the object it is producing, it must preserve the type of the output symbol as `STT_COMMON`.
>
> When the dynamic linker encounters a reference to a symbol that resolves to a definition of type `STT_COMMON`, it may (but is not required to) change its symbol resolution rules as follows: instead of binding the reference to the first symbol found with the given name, the dynamic linker searches for the first symbol with that name with type other than `STT_COMMON`. If no such symbol is found, it looks for the `STT_COMMON` definition of that name that has the largest size.

`--elf-stt-common=yes` causes GNU assembler to use `STT_COMMON`. It is super rare in the wild, though.
```sh
% as a.s --elf-stt-common=yes -o a.o
% readelf -Ws a.o

Symbol table '.symtab' contains 2 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000004     4 COMMON  GLOBAL DEFAULT  COM x
```

## Linker behavior

### Symbol resolution

The key is: a COMMON symbol does not lead to a duplicate definition error with any kind of definitions.
```text
# Two STB_GLOBAL definitions lead to a duplicate definition error.
% as -o b.o <<< '.data; .globl x; x:'
% ld.lld -e 0 b.o b.o
ld.lld: error: duplicate symbol: x
>>> defined at b.o:(.data+0x0)
>>> defined at b.o:(.data+0x0)
```

However, the size and alignment fields may be updated when two COMMON symbols are merged.
The quoted generic ABI text describes the behavior when a COMMON symbol has different sizes in relocatable objects. The output symbol gets the largest size.

Platforms differ in how the alignment is selected. GNU ld and ld.lld pick the largest alignment.
```sh
as -o a.o <<< '.comm x,8,4'
as -o b.o <<< '.comm x,4,8'
ld a.o b.o  # st_size==8, aligned by 8
```

Mach-O ld64 lets the copy with the largest size decide the alignment.

In ELF, for a definition in a relocatable object file, the precedence is `STB_GLOBAL > COMMON > STB_WEAK`.

> When the link editor combines several relocatable object files, it does not allow multiple definitions of `STB_GLOBAL` symbols with the same name. On the other hand, if a defined global symbol exists, the appearance of a weak symbol with the same name will not cause an error. The link editor honors the global definition and ignores the weak ones. Similarly, if a common symbol exists (that is, a symbol whose `st_shndx` field holds `SHN_COMMON`), the appearance of a weak symbol with the same name will not cause an error. The link editor honors the common definition and ignores the weak ones.

```sh
as -o a.o <<< '.comm x,8,4'
as -o b.o <<< '.data; .globl x; x: .space 16; .size x, 16'
as -o c.o <<< '.data; .weak x; x: .space 16; .size x, 16'
```
```sh
% ld.bfd -e 0 a.o b.o  # b.o wins (COMMON < STB_GLOBAL)
ld.bfd: warning: alignment 1 of symbol `x' in b.o is smaller than 4 in a.o
ld.bfd: warning: size of symbol `x' changed from 8 in a.o to 16 in b.o
% ld.bfd -e 0 a.o c.o  # a.o wins (COMMON > STB_WEAK)
```

When a common symbol is merged with a dynamic symbol, GNU ld displays different behaviors depending on whether the dynamic symbol is relative to a BSS section.
See the definition of `newdyncommon` in `bfd/elflink.c:_bfd_elf_merge_symbol` (originally `bfd/elflink.h` from "19990502 sourceware import").
```sh
as -o a.o <<< '.comm x,8,4'
as -o bss.o <<< '.bss; .globl x; x: .quad 0; .size x, .-x' && ld -shared bss.o -o bss.so
as -o data.o <<< '.data; .globl x; x: .quad 0; .size x, .-x' && ld -shared data.o -o data.so
ld.bfd a.o bss.so -o bfd.bss  # x is defined
ld.bfd a.o data.so -o bfd.data  # x is undefined
ld.lld a.o bss.so -o lld.bss  # x is defined
ld.lld a.o data.so -o lld.data  # x is defined
```

This special case does not make sense to me. In a relocatable object file, a COMMON symbol is stronger than a weak symbol.
Since a weak symbol takes precedence over a dynamic symbol, a COMMON symbol should as well.
gold and ld.lld unconditionally let the common symbol win.

When a common symbol is merged with a dynamic symbol, GNU ld and ld.lld (see [D71161](https://reviews.llvm.org/D71161)) increase `st_size` if the shared symbol has a larger `st_size`.
```sh
as -o a.o <<< '.comm x,8,4'
as -o b.o <<< '.comm x,4,8'
ld -shared a.o -o a.so
ld a.so b.o  # st_size==8, aligned by 4
```

GNU ld ported a strange rule from SUN's linker in 1999-12: [GNU-ld behaviour does not match native linker behaviour](https://sourceware.org/pipermail/binutils/1999-December/002952.html).

When a COMMON symbol in relocatable object files has a larger size than that of the shared object, if we allocate the COMMON symbol in the executable, with the help of symbol interposition, both accesses from the executable and the shared object will be bound to the larger object.
On the other hand, if we leave the symbol undefined in the executable, the accesses from the executable may go out-of-bounds.

> Here is a table showing when an element is pulled in from an archive with the Solaris 2.6 linker and ar program:

```text
main program\archive   undefined    common    defined

undefined                no          yes       yes
common                   no          no        yes
defined                  no          no        no
```

When a symbol is COMMON and ld sees an archive, ld checks whether the archive index provides a `STB_GLOBAL` definition of the symbol.
If yes, ld extracts the archive as well.
This is in contrary to the usual rule that only an undefined symbol leads to archive member extraction.

ld.lld since 12.0.0 has this behavior ([D86142](https://reviews.llvm.org/D86142)) with the enabled-by-default `--fortran-common` option.

Say `b0.a` and `b1.a` are mostly identical archives, but `b0.a` objects are compiled with `-fcommon` while `b1.a` objects are compiled with `-fno-common` .
If `a.o` references `b0.a`, this archive lookup behavior may cause a duplicate definition error for `ld a.o b0.a b1.a` while `b1.a` can be shadowed by `b0.a` without the rule.

```sh
echo 'extern int ret; int main() { return ret; }' > a.c
echo 'int ret; void foo() {}' > b.c
gcc -c a.c
gcc -c -fcommon b.c -o b0.o && rm -f b0.a && ar rc b0.a b0.o
gcc -c b.c -o b1.o && rm -f b1.a && ar rc b1.a b1.o
```
```text
# ret in b0.a(b0.o) is COMMON. b1.a(b1.o) is extracted to override the COMMON symbol with a STB_GLOBAL definition.
% gcc a.o b0.a b1.a
ld.lld: error: duplicate symbol: foo
>>> defined at b.c
>>>            b0.o:(foo) in archive b0.a
>>> defined at b.c
>>>            b1.o:(.text+0x0) in archive b1.a
collect2: error: ld returned 1 exit status
% gcc a.o b1.a b0.a  # b1.a shadows b0.a
```

What I am most concerned with is how to parallelize symbol resolution in the presence of this archive lookup rule.

GNU ld and ld.lld treat COMMON symbols as though they are in an input section named `COMMON`.
`*(COMMON)` in a linker script can match these symbols.

### Error-prone COMMON symbols

With `-fcommon`, due to the linker symbol resolution rule, a tentative definition `int x;` may be overridden by a `STB_GLOBAL` definition in another compilation unit.
This is error-prone since the user may assume an initial value of zero if unware of `int x = 1;`.

```sh
gcc -c -fcommon -xc - -o a.o <<< 'int x;'
gcc -c -xc - -o b.o <<< 'int x = 1;'
gcc -shared a.o b.o  # no diagnostic
```

GNU ld and ld.lld support `--warn-common` which detects the error-prone overridding.
```sh
% gcc -shared -fuse-ld=bfd -Wl,--warn-common a.o b.o
/usr/bin/ld.bfd: b.o: warning: definition of `x' overriding common from a.o
```

Some legacy code may inadvertently rely on COMMON symbols by having something like `int x;` in a header file.
Such code may not compile with `-fno-common`.

### `.bss` allocation

When producing an executable or shared object, the linker allocates space in `.bss` to hold COMMON symbols.
In GNU ld, COMMON symbols are placed after `.bss` and `.bss.*` input sections.

```text
  .bss            :
  {
   *(.dynbss)
   *(.bss .bss.* .gnu.linkonce.b.*)
   *(COMMON)
   /* Align here to ensure that the .bss section occupies space up to
      _end.  Align after .bss to ensure correct alignment even if the
      .bss section disappears because there are no input sections.
      FIXME: Why do we need it? When there is no .bss section, we do not
      pad the .data section.  */
   . = ALIGN(. != 0 ? 64 / 8 : 1);
  }
```

In a relocatable link, COMMON symbols remain COMMON.

## Run-time behavior

```c
// a.c
#include <stdio.h>
int x;
void foo();
int main() {
  foo();
  printf("%d\n", x);
}

// b.c
int x;
void foo() { x++; }
```

When `a.c` and `b.c` are in the same component (main executable or shared object), with `-fcommon`, it's clear that the two `x` resolves to the same copy and the output is 1.
```text
% gcc -fcommon a.c b.c && ./a.out
1
```

If `b.c` is compiled and linked into a different component, this works with the help of ELF symbol interposition.
When linking the shared object, `x` is preemptible (default visibility non-local binding) and its access requires GOT indirection.
When linking the executable, the linker exports `x` to the dynamic symbol table because it is used by an input shared object.
```text
% gcc -fpic -shared -fcommon b.c -o b.so && gcc -fcommon a.c ./b.so && ./a.out
1
% readelf -W --dyn-syms a.out | egrep 'Num:| x'
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     8: 0000000000003af8     4 OBJECT  GLOBAL DEFAULT   25 x
% readelf -W --dyn-syms b.so | egrep 'Num:| x'
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     5: 00000000000037a0     4 OBJECT  GLOBAL DEFAULT   20 x
% readelf -Wr --dyn-syms b.so | egrep 'Num:| x'
0000000000002770  0000000500000006 R_X86_64_GLOB_DAT      00000000000037a0 x + 0
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     5: 00000000000037a0     4 OBJECT  GLOBAL DEFAULT   20 x
```

If you make `x` non-preemptible (e.g. vi `-Bsymbolic`) in `b.so`, `b.so` will get its own copy.
```text
% gcc -fpic -shared -fcommon -Wl,-Bsymbolic b.c -o b.so && gcc -fcommon a.c ./b.so && ./a.out
0
```

### Miscellaneous related linker options

#### `--no-define-common`

In 2001-09, [optionally postpone assignment of Common](https://sourceware.org/pipermail/binutils/2001-September/014015.html) added this option to be used with `-shared`.

Here is my understanding: glibc around 2.1.3 used to have a ld.so bug that the ELF interposition might not work.
Using `--no-define-common` with shared objects can make COMMON symbols undefined and circumvent the bug.
```text
% gcc -fpic -Wl,--no-define-common -shared -fuse-ld=bfd -fcommon b.c -o b.so
% readelf -W --dyn-syms b.so | egrep 'Num:| x'
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     1: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  UND x
```

gold [confuses `--define-common` with `-d`/`FORCE_COMMON_ALLOCATION`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=0dfbdef4c43cfe12bb3e2505ebe5acc651a35c98) and implements `--define-common` with `-d` semantics.
Its `--no-define-common` is incompatible with GNU ld.

#### `-d`, `-dc`, `-dp`

In a relocatable link, a COMMON symbol remains COMMON in the output.
If `-dc` is specified, the linker will allocate space to COMMON symbols. `-d` and `-dp` are aliases for `-dc`.

```text
% gcc -fpic -fcommon -r -fuse-ld=bfd b.c -o b.ro && readelf -Ws b.ro | egrep 'Num:| x'
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     9: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM x
% gcc -fpic -fcommon -r -fuse-ld=bfd -Wl,-dc b.c -o b.ro && readelf -Ws b.ro | egrep 'Num:| x'
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     9: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    7 x
```

The output has a regular `STB_GLOBAL` definition. Linking the relocatable output with another which defines `x` will lead to a duplicate definition error.
```sh
ld.bfd -r b.o -o b.ro
ld.bfd b.ro b.ro  # ok. COMMON symbols are merged

ld.bfd -r -dc b.o -o b.ro
ld.bfd b.ro b.ro  # duplicate definition error
```

The options are obscure and might be used to work around some legacy programs.
If the relocatable output is fed into the linker again, ignoring `-dc` should usually work as well.
Only when the program inspects relocatable output by itself and does not recognize COMMON symbols, there may be a problem.
This implies that the program cannot process a relocatable object with COMMON symbols produced by the assembler.

For ld.lld, I removed `-dp` and ignored `-d`/`-dc` for 15.0.0: <https://github.com/llvm/llvm-project/issues/53660>.

#### `--sort-common`

By sorting COMMON symbols by decreasing alignment, some padding can be saved.
However, I think this hardly ever has any size benefit.
For example, musl specifies `--sort-common` by default.
With `-fcommon`, I see a 24 byte decrease of `.bss`. The total size of `.bss` is 11344 bytes.

```patch
@@ -2022,12 +2022,13 @@
 Allocating common symbols
 Common symbol       size              file

-__thread_list_lock  0x4               obj/src/env/__init_tls.lo
-__aio_fut           0x4               obj/src/aio/aio.lo
+__libc              0x68              obj/src/internal/libc.lo
 __stack_chk_guard   0x8               obj/src/env/__stack_chk_fail.lo
 __hwcap             0x8               obj/src/internal/libc.lo
-__libc              0x68              obj/src/internal/libc.lo
 optarg              0x8               obj/src/misc/getopt.lo
+__sysinfo           0x8               obj/src/internal/defsysinfo.lo
+__thread_list_lock  0x4               obj/src/env/__init_tls.lo
+__aio_fut           0x4               obj/src/aio/aio.lo
 getdate_err         0x4               obj/src/time/getdate.lo
 __abort_lock        0x4               obj/src/exit/abort_lock.lo
 h_errno             0x4               obj/src/network/h_errno.lo
@@ -2039,7 +2040,6 @@
 __aligned_alloc_replaced
                     0x4               obj/src/malloc/replaced.lo
 __eintr_valid_flag  0x4               obj/src/signal/sigaction.lo
-__sysinfo           0x8               obj/src/internal/defsysinfo.lo

 Discarded input sections

```

Actually, this can degrade performance if COMMON symbols in an object file have locality and `--sort-common` breaks the locality.

### `edata`, `end`, and `etext`

For legacy reasons GNU ld's internal linker script has `PROVIDE(edata = .);` and similar symbol assignments for the other two symbols.
In GNU ld, the definition precedence is: regular symbol assignment > relocatable object definition > `PROVIDE` symbol assignment.

If a relocatable object file defines `end`, it will take precedence over the internal linker script `PROVIDE(end = .);`.
This makes sense because the global variable `int end;` is valid in C and C++.

Before ld.lld 15, `int end;` compiled with `-fcommon` is overridden by the linker definition.
This will be fixed by [D120389](https://reviews.llvm.org/D120389).

## LLVM IR

In LLVM IR, a COMMON symbol has the ["common" linkage](https://llvm.org/docs/LangRef.html#linkage-types).
It is an interposable linkage and some optimizations are suppressed. For example:

* InstCombine assumes that the addresses of a `common global i8` and an `external global i32` may be the same.
* `llvm.objectsize` intrinsic does not know the size. This may lead to conservative assumptions for some `_chk` functions.
