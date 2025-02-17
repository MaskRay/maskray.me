layout: post
title: Symbol processing
author: MaskRay
tags: [binutils,llvm,linker]
---

UNDER CONSTRUCTION (COFF, Mach-O)

Symbol processing is a major step in a linker.
In most binary formats, the linker maintains a global symbol table and performs symbols resolution for each input file (object file, shared object, archive, LLVM bitcode file).
Some command line options can define/undefine symbols as well.
The symbol resolution can affect archive processing and many subsequent steps (LTO, relocation processing, as-needed shared objects, etc).

<!-- more -->

# ELF

## Symbol tables

An object file can have optional symbol tables.

A relocatable object file almost always has a symbol table, which is represented by a section `.symtab` of type `SHT_SYMTAB`.
The symbol table is sometimes called a "static symbol table".

An executable or shared object almost always has a dynamic symbol table, which is represented by a section `.dynsym` of type `SHT_DYNSYM`.
The dynamic symbol table specifies defined and undefined symbols, which can be seen as its export and import lists.
They are needed by runtime relocation processing and symbol binding.
A position dependent statically linked executable usually has no dynamic symbol table, because (1) it usually does not need dynamic relocations and (2) there is only one component and every needed symbol is defined internally, no need for symbol binding.

An executable or shared object may have an optional symbol table of type `SHT_SYMTAB`.
ld produces the symbol table (`.symtab`) by default.
The static symbol table is a superset of the dynamic symbol table and has many entries (local symbols and other non-exported symbols) not needed by runtime.
It has value for symbolization without debug information but otherwise is not useful.
Therefore an executable or shared object is usually post-processed by `strip --strip-all` which can remove `.symtab` along with `.strtab` and debug sections.

An archive (also named a static library) is like a tarball. It almost always contains multiple relocatable object files.

## Symbols

A symbol table holds an array of entries. Each symbol table entry indicates a symbol.
Let's look at the representations of a 32-bit ELF object file and a 64-bit ELF object file:

```c
typedef struct {
  uint32_t st_name;
  uint32_t st_value;
  uint32_t st_size;
  uint8_t  st_info;
  uint8_t  st_other;
  uint16_t st_shndx;
} Elf32_Sym;

typedef struct {
  uint32_t st_name;
  uint8_t  st_info;
  uint8_t  st_other;
  uint16_t st_shndx;
  uint64_t st_value;
  uint64_t st_size;
} Elf64_Sym;
```

Here is the description from the ELF specification:

* `st_name`: This member holds an index into the object file's symbol string table, which holds the character representations of the symbol names. If the value is non-zero, it represents a string table index that gives the symbol name. Otherwise, the symbol table entry has no name.
* `st_value`: This member gives the value of the associated symbol. Depending on the context, this may be an absolute value, an address, and so on; details appear below.
* `st_size`: Many symbols have associated sizes. For example, a data object's size is the number of bytes contained in the object. This member holds 0 if the symbol has no size or an unknown size.
* `st_info`: This member specifies the symbol's type and binding attributes. A list of the values and meanings appears below. The following code shows how to manipulate the values for both 32 and 64-bit objects.
* `st_other`: This member currently specifies a symbol's visibility. A list of the values and meanings appears below. The following code shows how to manipulate the values for both 32 and 64-bit objects. Other bits contain 0 and have no defined meaning.
* `st_shndx`: Every symbol table entry is defined in relation to some section. This member holds the relevant section header table index. As the `sh_link` and sh_info interpretation table and the related text describe, some section indexes indicate special meanings. If this member contains `SHN_XINDEX`, then the actual section header index is too large to fit in this field. The actual value is contained in the associated section of type `SHT_SYMTAB_SHNDX`.

Explanation:

`st_name` indicates the name.

`st_shndx` and `st_value` indicate whether the symbol is defined or undefined, and the associated section and the offset if defined.
If `st_shndx==SHN_UNDEF`, we say the symbol is undefined. For an undefined symbol `foo`, we often say the object file references `foo`.
If `st_shndx!=SHN_UNDEF`, we say the symbol is defined.

Some `st_shndx` values are special.
If `st_shndx==SHN_ABS`, this is an absolute symbol.
If `st_shndx==SHN_COMMON`, this is a common symbol (FORTRAN `COMMON` blocks or C tentative definitions). The binding must be `STB_GLOBAL`. A common symbol can also be represented with the type `STT_COMMON` but that is uncommon.

`st_info` encodes the type and the binding.
Among types, `STT_FILE`, `STT_SECTION` and `STT_TLS` are special. Most symbols are of type `STT_NOTYPE`, `STT_OBJECT`, and `STT_FUNC`. Other types are uncommon.
The binding is a very important attribute. All of `STB_LOCAL`, `STB_GLOBAL`, and `STB_WEAK` are important.
A symbol of binding `STB_LOCAL` is often called a local symbol.
A local symbol must be defined. It is not visible outside the object file, therefore it doesn't contribute to the global symbol table.
`STB_WEAK` represents a weak symbol. See [Weak symbol](/blog/2021-04-25-weak-symbol) for details.
`STB_GLOBAL` represents a regular symbol visible outside the object file.
Both weak and global symbols contribute to the global symbol table.

`st_other` encodes the visibility. The other bits are used by ppc64 ELFv2, AArch64, MIPS, etc.
The visibility attribute represents different symbol resolution strategies for a non-local symbol.
The linker only uses the information for a relocatable object file, not for a shared object.

A `STV_HIDDEN` or `STV_INTERNAL` symbol will be made `STB_LOCAL` in the linker output. This provides a mechanism to ensure a relocatable object file symbol will not be visible to other components.
A `STV_PROTECTED` symbol provides a way to defeat performance loss due to symbol interposition for a relocatable object file which will be linked into a shared object.
`STV_DEFAULT` is the default.

[SGI introduced `STV_INTERNAL`](https://groups.google.com/g/generic-abi/c/AahK3BVmYWQ/m/oylW2Jkvpy0J) for their link-time interprocedural optimization.
They are more restricted than `STV_HIDDEN` in that they are not accessed outside of the compile unit.

If multiple relocatable object files have a non-local symbol, the most constraining visibility will be the visibility in the output.
The attributes, ordered from least to most constraining, are: `STV_DEFAULT`, `STV_PROTECTED`, `STV_HIDDEN`, and `STV_INTERNAL`.
For a non-definition declaration in C/C++, we can make it `STV_PROTECTED` or `STV_HIDDEN` to ensure the symbol must be defined in the component.
Actually, if every undefined is `STV_PROTECTED` by default, the model will be similar to PE/COFF's non-export by default model.

`st_size` just wastes space. The only minor use case is to provide a hint to `.symtab` based symbolization.

## Symbol resolution

The following pseudocode gives a summary of input file processing in the linker.

```text
for file in input {
  if file is a relocatable object file/bitcode file surrounded by --start-lib {
    for sym in file's non-local symbols {
      ... // Lazy object file extraction may happen
    }
  } else if file is an archive not surrounded by --whole-archive {
    for sym in file's index {
      ... // Archive member extraction may happen
    }

  } else if file is a shared object {
    for sym in file's .dynsym { ... }

  } else if file is a relocatable object file/bitcode file not surrounded by --start-lib {
    for sym in file's non-local symbols { ... }
  } else if file is an archive surrounded by --whole-archive {
    for member in file {
      if member is a relocatable object file/bitcode file {
        handle member as a regular relocatable object file/bitcode file
      }
    }

  } else {
    handle file as a linker script
  }
}
```

The linker maintains a global symbol table for non-local symbols (`STB_GLOBAL`, `STB_WEAK`, or `STB_GNU_UNIQUE`).
The table can be seen as a collection mapping names to states.
Each state encodes the symbol kind. In the following list, I place the ld.lld internal struct name before the description.

* `Undefined`: An undefined symbol only referenced by shared objects (different from the below in that this does not need an output symbol table entry)
* `Undefined`: An undefined symbol referenced by at least one relocatable object file/bitcode file
* `LazySymbol`: An entry in an archive index or a definition in a relocatable object file/bitcode file inside a pair of `--start-lib` `--end-lib`
* `Shared`: A definition in a shared object
* `CommonSymbol`: A common symbol in a relocatable object file or LLVM bitcode file
* `Defined`: A definition in a relocatable object file or LLVM bitcode file

The kinds are listed in increasing precedence.
In general, if the symbol in the current input file has a higher precedence than the global symbol table entry, the global symbol table entry will be overwritten.
There are special cases such as archive processing and common symbols taking precedence over defined weak symbols.

Note the first `Undefined` kind. Such an undefined symbol only referenced by shared objects will not contribute a symbol table entry to the output.
It is needed for several purposes:

* Archive member extraction
* [`--no-allow-shlib-undefined`](/blog/2021-06-13-dependency-related-linker-options#no-allow-shlib-undefined)
* Mark the symbol as exported if it ends up defined

If a `Shared` is defined by multiple shared objects, the first shared object wins.

`CommonSymbol` can be regarded a special `Defined` which has a precedence lower than `STB_GLOBAL` but higher than `STB_WEAK`.

Among `Defined` symbols, a `STB_GLOBAL` overrides a `STB_WEAK`. If two `STB_GLOBAL` occur, the linker will report a duplicate definition error.
There is an exception for two `SHN_ABS` symbols with the same value.

I will illustrate the symbol resolution rules with examples.
As an abbreviation, I use "insert `foo`" to mean "insert `foo` to the global symbol table".

### Duplicate definitions between relocatable object files

Both `a.o` and `b.o` define `STB_GLOBAL` `foo`.
The linker command line is `ld a.o b.o`.

* For `a.o`, insert `foo` as a `Defined`.
* For `b.o`, notice that `foo` exists. Both are `STB_GLOBAL` => duplicate definition error.

### STB_GLOBAL overrides STB_WEAK between relocatable object files

`a.o` defines `STB_GLOBAL` `foo`. `c.o` defines `STB_WEAK` `foo`.
The linker command line is `ld a.o c.o`.

* For `a.o`, insert `foo` as a `Defined`.
* For `c.o`, notice that `foo` exists. The existing `STB_GLOBAL` definition wins.

In the output, the definition will come from `a.o`.

For `ld c.o a.o`, the incoming `STB_GLOBAL` definition will override the existing `STB_WEAK` definition.

Note: the `STB_GLOBAL` overriding `STB_WEAK` rule is between two relocatable object files.

### Common overrides STB_WEAK between relocatable object files

`c.o` defines `STB_WEAK` `foo`. `d.o` defines `STB_GLOBAL SHN_COMMON` `foo`.
The linker command line is `ld c.o d.o`.

* For `c.o`, insert `foo` as a `Defined`.
* For `d.o`, the incoming `CommonSymbol` wins.

The linker command line is `ld d.o c.o`.

* For `d.o`, insert `foo` as a `CommonSymbol`.
* For `c.o`, notice that `foo` exists. The incoming `STB_WEAK` definition is ignored.

### Relocatable object file overriding shared object

`a.so` defines `STB_GLOBAL` `foo`. `c.o` defines `STB_WEAK` `foo`.
The linker command line is `ld a.so c.o`.

* For `a.so`, insert `foo` as a `Shared`.
* For `c.o`, notice that `foo` exists. The relocatable object file definition wins.

For `ld c.o a.so`, the definition in `a.so` will be ignored.

Note: the binding in a shared object is ignored for symbol resolution.
The `STB_GLOBAL` overriding `STB_WEAK` rule does not apply, because a shared object is involved.

### First shared object wins

`a.so` defines `STB_GLOBAL` `foo`. `c.so` defines `STB_WEAK` `foo`.
The linker command line is `ld c.so a.so`.

* For `c.so`, insert `foo` as a `Shared`.
* For `a.so`, notice that `foo` exists. The first shared object wins.

Note: the binding in a shared object is ignored for symbol resolution.
The `STB_GLOBAL` overriding `STB_WEAK` rule does not apply, because two shared objects are involved.

### An undefined symbol in a shared object does not change the binding

`w.o` references `STB_WEAK` `foo`. `x.so` references `STB_GLOBAL` `foo`.
The linker command line is `ld w.o x.so`.

* For `w.o`, insert `foo` as an `STB_WEAK` `Undefined`.
* For `x.so`, notice that `foo` exists. Ignore the incoming symbol.

The output binding will be `STB_WEAK`.
For an executable link, `-z defs` is the default. The linker will report an error.

## Archive processing

Archive processing is the most difficult part of the symbol resolution.
The complexity comes from the two states an archive member has.

* not extracted (lld calls this state "lazy")
* extracted (non-lazy), like a regular relocatable object file

Initially every archive member is in a lazy state.
A defined non-local symbol is called a lazy symbol.
The undefined symbols do not participate in symbol resolution.
After symbol resolution, if an archive member stays in the lazy state, its symbols will be ignored and not affect the output symbol table.
The link behavior is as if the archive member was not present.

When the linker scans an archive, if a lazy symbol can be used to resolve an `STB_GLOBAL` undefined symbol referenced by a relocatable object file, bitcode file, or shared object, the linker will extract the member defining it.
Upon extraction, the member turns into a non-lazy state. It will be processed like a regular relocatable object file/bitcode file.

As a replacement for thin archives, gold and ld.lld support `--start-lib` and `--end-lib`.
A relocatable object file/bitcode file surrounded by the options is said lazy in the ld.lld representation.
Such a lazy object file is similar to an archive with just one member.
A symbol defined by such a lazy object file is also a lazy symbol.

Here is the simplified symbol resolution algorithm focusing on archive member extraction.
Basically the linker maintains a set of non-weak (must be `STB_GLOBAL`) undefined symbols.
All relocatable object files, bitcode files, and shared objects can contribute to the set.
For an incoming archive, each member is inspected.
If a member resolves a `STB_GLOBAL` undefined symbol, the member will be extracted.
(I emphasize `STB_GLOBAL` because a `STB_WEAK` undefined symbol does not trigger extraction. In Mach-O, a weak reference triggers extraction as well.)

```text
// Used by traditional Unix ELF linkers
undef = {};
for file in input {
  if file is a relocatable object file/bitcode file not surrounded by --start-lib {
    for sym in file's non-local symbols {
      if sym.isDefined() { undef.erase(sym); }
      else if !sym.isWeak() { undef.insert(sym); }
    }
  } else if file is a shared object {
    for sym in file's SHT_DYNSYM section {
      if sym.isDefined() { undef.erase(sym); }
      else if !sym.isWeak() { undef.insert(sym); }
    }
  } else if file is an archive not surrounded by --whole-archive/--start-group {
    bool cont = true;
    while (cont) {
      cont = false;
      for member in file {
        if member is not a regular object file || member is extracted { continue; }

        // A member is extracted if it defines a previously undefined non-weak symbol.
        // See below for archive index.
        bool extract = false;
        for sym in member's non-local symbols
          if sym.isDefined() && undef.count(sym)
            extract = true;
        if !extract { continue; }

        // Treat `member` as a regular relocatable object file/bitcode file.
        // Duplicate definition errors are now possible.
        cont = true;
        for sym in member's non-local symbols {
          if sym.isDefined() { undef.erase(sym); }
          else if !sym.isWeak() { undef.insert(sym); }
        }
      }

      // The extraction of one member may cause other members to be extracted.
      // This is an iterative process.
    }
  } else {
    ...
  }
}
```

As you see, the `while (cont)` loop is a bottleneck.
Typically linkers require an index for each archive.
The index is a list of (name, member) pairs for defined non-local symbols.
See [Archives and --start-lib](/blog/2022-01-16-archives-and-start-lib) for the background.
With the index, the `if sym.isDefined() && undef.count(sym)` part can be optimized.

Some options can alter the archive semantics.

For an archive following `--whole-archive`, its members enter the non-lazy state directly.

The archives surrounded by a pair of `--start-group` and `--end-group` can be perceived as a single archive.
More precisely, the linker will consider all the archives surrounded by `--start-group` and `--end-group` in the `while (cont)` loop.

Some takeaway:

The symbol definition precedence is roughly: relocatable object file/bitcode file > shared object > archive.
An archive definition loses to a shared definition, but it can win over the shared definition if extraction happens.

Without `--whole-archive`, an archive may contribute zero or some members to the relocatable object file/bitcode file set.

If the defined symbol set of an archive member is a subset of a preceding relocatable object file/bitcode file/shared object or an archive member which is guaranteed to be extracted,
the archive member will be ignored.
The behavior is somewhat similar to symbol interposition. See [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic) for details.

For `ld ... definition.a reference.o`, the definition may be entirely dropped and a later `reference.o` will lead to an undefined symbol error.

For an archive member to be extracted, the dependents need to appear before the archive or within another member of the archive.
This property leads to a loose topological order requirement.

An external program 'lorder' or build system's integrated topological sorting feature is needed to order archives.
In the old days, the program "lorder" (<https://www.gnu.org/software/coreutils/manual/html_node/tsort-background.html>) was used.

The archive order matters (loss of commutativity) and can hide brittle build problems due to duplicate definitions.
Minor ordering tweaks can cause subtle behavior changes (symbol resolution).

When providing an interceptor library (a library providing overriding definitions), you usually want to make it optional, i.e. the intercepted library does not have a dependency on the interceptor.
However, when linking both the intercepted archive and the interceptor archive, you need to be careful with the order, which usually requires some special plumbing in the build system.
A common approach is `--whole-archive`, which unfortunately loses the nice lazy property.

I will use some examples to explain the symbol resolution rules related to lazy symbols.

### Shared object overriding archive

Both `a.so` and `b.a(b.o)` define `foo`.
The linker command line is `ld a.so b.a`.

* For `a.so`, insert `foo` as a `Shared`.
* For `b.a`, try inserting every symbol from the index. `foo` is already a shared definition, so `a.so` wins. `b.a(b.o)` is not extracted.

`ld a.so --start-lib b.o --end-lib` is similar.

See [LLD selects symbol from later dynamic library over earlier archive, for LTO-generated libcall](https://github.com/llvm/llvm-project/issues/57207) for a phenomenon.

### Extracted archive member overriding shared object

`0.o` has a `STB_GLOBAL` undefined symbol `bar`. `a.o` defines `foo`. `b.a(b.o)` defines `foo` and `bar`.
The linker command line is `ld 0.o a.so b.a`.

* For `0.o`, insert `bar` as an `Undefined`.
* For `a.so`, insert `foo` as a `Shared`.
* For `b.a`, try inserting every symbol from the index. `foo` is already a shared definition, so inserting `foo` is ignored. However, `bar` in the archive index can resolve the `Undefined` entry in the global symbol table, so the member providing `bar` (`b.o`) is extracted.
* `b.a(b.o)` is added like a relocatable object file. Its `foo` definition overrides the `Shared` entry in the global symbol table. Its `bar` definition overrides the `Undefined` entry.

Note: in Mach-O, if the `a.so` definition is weak, `b.a` is loaded even if it is weak. A non-weak dylib definition has the same precedence as a weak/non-weak archive definition.

The linker command line is `ld 0.o b.a a.so`.

* For `0.o`, insert `bar` as an `Undefined`.
* For `b.a`, insert `foo` as a `LazyArchive`. `bar` in the archive index can resolve the `Undefined` entry, so `b.a(b.o)` is extracted.
* `b.a(b.o)` is added like a relocatable object file. Its `foo` definition overrides the `LazyArchive` entry in the global symbol table. Its `bar` definitions overrides the `Undefined` entry.
* For `a.so`, its shared definition loses to the `Defined` entry in the global symbol table.

### Relocatable object file suppressing archive member extraction

`m1.o` defines `memcpy`. `libc.a(memcpy.o)` defines `memcpy`.
The linker command line is `ld ... m1.o -lc`.

* For `m1.o`, insert `memcpy` as a `Defined`.
* For `libc.a`, try inserting every symbol from the index. Some members are extracted. However, because no symbol defined by `memcpy.o` is `Undefined` in the global symbol table, `memcpy.o` is not extracted.

As a result, `m1.o` succeeds in shadowing `libc.a(memcpy.o)`.
In practice, `m1.o` may be an object file providing optimized `memcpy` routines.

`m2.o` defines `memcmp`. `libc.a(memcmp.o)` defines `memcmp` and `STB_WEAK` `bcmp`.
The linker command line is `ld ... m2.o -lc`.

* `bcmp` is referenced and thus inserted as an `Undefined`.
* For `m2.o`, insert `memcpy` as a `Defined`.
* For `libc.a`, `bcmp` in the archive index can resolve the `Undefined` entry, therefore `libc.a(memcmp.o)` is extracted.
* `libc.a(memcmp.o)` is added like a relocatable object file. Its `STB_GLOBAL` `memcmp` definition causes a duplicate definition error with the existing `STB_GLOBAL` definition.

The example above demonstrates that if we want to shadow an archive member, it is better to ensure all its symbols which may be referenced are defined.
I.e. The defined symbols of the interceptor should be a superset of the archive member being shadowed.

## Symbol order

```
% cat order.s
# REQUIRES: x86
# RUN: rm -rf %t && split-file %s %t
# RUN: cd %t
# RUN: clang -c A.s a.s b.s c.s
# RUN: llvm-ar rc A.a A.o
# RUN: llvm-ar rc a.a a.o
# RUN: llvm-ar rc b.a b.o
# RUN: llvm-ar rc c.a c.o
# RUN: ld.lld -e A1 A.o b.o c.o -o A
# RUN: ld.lld -e A1 A.a b.a c.a -o A_archive
# RUN: ld.lld -e a1 a.o b.o c.o -o a
# RUN: ld.lld -e a1 a.a b.a c.a -o a_archive

#--- A.s
.globl A1, b2, c1, A2
A1:
  call b2
  call c1
A2:

#--- a.s
.globl a1, b1, c1, a2
a1:
  call b1
  call c1
a2:

#--- b.s
.globl b1, b2, c1
b1:
b2:
  call c1

#--- c.s
.globl c1, c2
c1:
c2:
```

Let's say we have two applications A (A.o+b.o+c.o) and a (a.o+b.o+c.o).
A.o and a.o reference different definitions in b.o: b2 vs b1.

ld.lld's symbol order is dependent on the first occurrence.
The forward symtab of A consists of symbols in A.o+b.o+c.o with duplicates removed.
Let's imagine a backward order if we link A using this reversed order: c.o, b.o, A.o.

* The forward symtab of A:  A1 b2 c1 A2 b1 c2.
* The forward symtab of a:  a1 b1 c1 a2 b2 c2.
* The backward symtab of A: c1 c2 b1 b2 A1 A2.
* The backward symtab of a: c1 c2 b1 b2 a1 a2.

It's clear that backward symtabs are more similar.

However, in some use cases, most input files are archives (or `--start-lib/--end-lib` surrounded relocatable files).
After my <https://reviews.llvm.org/D95985> in 2021, `A_archive` and `a_archive` have similar symtabs as well!

* A_archive: A1 A2 b1 b2 c1 c2
* a_archive: a1 a2 b1 b2 c1 c2

## a.out

a.out is the ancestor of Mach-O, COFF, and ELF.

There are a text segment and a data segment.
There are no flexible COFF/ELFF style sections, so no need for a ELF-style `st_shndx`.

```c
struct nlist {
  union {
     char *n_name;
     long n_strx;
  } n_un;
  unsigned char n_type;
  char          n_other;
  short         n_desc;
  unsigned long n_value;
};
```

* `n_type`: `N_UNDF, N_ABS, N_TEXT, N_DATA, N_BSS, N_FN`. The `N_EXT` bit is used to represent an external symbol.

## PE/COFF

The 8-byte short name and the 2-byte type just waste space.

The storage class basically maps to ELF's binding.
There are many useless values.
A weak definition needs an inconvenient auxiliary symbol record.
A weak reference cannot be expressed.

The auxiliary symbol record representation makes the symbol table non-uniform, i.e. not an array of fixed-length entries.
This makes random-access inconvenient.

Note: there is no 64-bit representation.

```c
struct Symbol {
  union {
    char short_name[8];
    struct {
      uint32_t zeroes; // 0
      uint32_t offset;
    };
  };
  uint32_t value;
  uint16_t section_number;
  uint16_t type;
  uint8_t  storage_class;
  uint8_t  num_aux;
};
```

With some exceptions, an unresolved undefined symbol leads to a linker error, even if it is not referenced.

A COFF file contributes defined and undefined symbols.

A defined symbol can be any of the following kinds:

* special (ignored in the global symbol table)
* common (section number is `IMAGE_SYM_UNDEFINED` and value is not 0)
* absolute (section number is -1)
* regular (section number is positive)

An undefined symbol has a storage class of `IMAGE_SYM_CLASS_EXTERNAL`, a section number of `IMAGE_SYM_UNDEFINED` (zero), and a value of zero.

An undefined symbol with a storage class of `IMAGE_SYM_CLASS_WEAK_EXTERNAL` is a weak external, which is actually like a weak definition in ELF.

### Code symbols

```c
// b.dll
__declspec(dllexport) void f() {}
// a.exe
__declspec(dllimport) void use() { f(); }
```

```asm
# b.dll
.globl f
f:

.section        .drectve,"yni"
.ascii  " -export:f"
```

The linker parses the import file and creates a definition for `__imp_f` pointing to the import address table entry and creates a definition `f` pointing to the thunk.
Normally, `f` is unused and will be discarded.
```asm
  call      *__imp_f(%rip)
```

However, if the user code does `call f` instead, the `f` definition will be retained:
```asm
  call f

f:
  jmpq *__imp_f(%rip)
```

`call      *__imp_f(%rip)` is like `-fno-plt` codegen for ELF.

### Data symbols

```c
// b.dll
__declspec(dllexport) int var;
// a.exe
__declspec(dllimport) extern int var;
int foo() { return var; }
```

```asm
# b.dll
.bss
.globl var
var:

.section        .drectve,"yni"
.ascii  " -export:var,data"
```

The linker parses the import file and creates a definition for `__imp_var` pointing to the import address table entry.
Unlike a code symbol, the linker does not create a definition for `var` (without the `__imp_` prefix).

With a `dllimport`:
```asm
  movq    __imp_var(%rip), %rax
  movl    (%rax), %eax
```

If `dllimport` is not specified,
```asm
  movq var(%rip), %rax
```

MinGW supports runtime pseudo relocations to patch the text section so that absolute and PC-relative references to the symbol will be bound to the actual definition.
```asm
  movq var(%rip), %rax  # the runtime will rewrite this to point to the definition in b.dll
```

## Mach-O

The 256-section limitation is unfortunate.
As a workaround, subsections were invented.

```c
struct nlist {
  uint32_t n_strx;
  uint8_t n_type;
  uint8_t n_sect;
  uint16_t n_desc;
  uint32_t n_value;
};

struct nlist_64 {
  uint32_t n_strx;
  uint8_t n_type;
  uint8_t n_sect;
  uint16_t n_desc; // N_ARM_THUMB_DEF, REFERENCED_DYNAMICALLY, N_WEAK_DEF, N_WEAK_REF, N_NO_DEAD_STRIP, N_ALT_ENTRY, ...
  uint64_t n_value;
};
```
