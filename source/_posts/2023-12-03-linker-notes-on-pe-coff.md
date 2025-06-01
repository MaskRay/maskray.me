layout: post
title: Linker notes on PE/COFF
author: MaskRay
tags: [binutils,linker,windows]
---

Updated in 2025-05.

This article describes linker notes about Portable Executable (PE) and Common Object File Format (COFF) used on Windows and UEFI environments.

In ELF, an object file can be a relocatable file, an executable file, or a shared object file.
On Windows, the term "object file" usually refers to relocatable files like ELF.
Such files use the Common Object File Format (COFF) while image files (e.g. executables and DLLs) use the Portable Executable (PE) format.

## A look at linkers

MSVC Linker is the reference linker on Windows.

GNU ld supports Windows thanks to the MinGW effort.
Here are some common BFD emulations for PE targets:

* `i386pe`: For 32-bit Intel x86 architectures.
* `i386pep`: For 64-bit Intel x86-64 architectures.
* `arm64pe`: For 64-bit ARM (AArch64) architectures.

For instance, you can use a command like `ld.bfd -m i386pep a.o b.o` to link two object files into a PE executable.
Many options on Linux can also be used when targeting PE/COFF.

The LLVM linker offers two modes of operation that are relevant to PE/COFF:

* `lld-link`: In this mode, lld emulates the MSVC linker, making it a fast viable alternative.
* `ld.lld`: When invoked with a PE emulation (like `-m i386pep`), lld emulates GNU ld. Code in `lld/MinGW` translates GNU ld options to link.exe options.

GNU ld implements [relocatable linking](/blog/2022-11-21-relocatable-linking) that is not available in other linkers.

## Input files

The input files to the linker can be object files, archive files, and import libraries.
GNU ld and lld-link allow linking against DLL files without an import library.

<!-- more -->

### Object files

### Import files

An import file (`.lib`) is a special archive file.
Each member represents a symbol to be imported.
The symbol `__imp_$sym` is inserted to the global symbol table.

The import header has a `Type` field indicating `IMPORT_OBJECT_CODE/IMPORT_OBJECT_DATA/IMPORT_OBJECT_CONST`.

For an import type of `IMPORT_OBJECT_DATA`, the symbol `$sym` is defined as an alias for `__imp_$sym`.

For an import type of `IMPORT_OBJECT_CODE`, the symbol `$sym` is defined as an import thunk, which is like a PLT entry in ELF.

GNU ld and lld-link allow linking against DLL files without an import library.
The behavior is as if the linker synthesizes an import library from a DLL file.

## Symbols

An object file contributes defined and undefined symbols.
An import file contributes defined symbols in a DLL that can be referenced by `__imp_$sym`.

A defined symbol can be any of the following kinds:

* special (ignored in the global symbol table)
* common (section number is `IMAGE_SYM_UNDEFINED` and value is not 0)
* absolute (section number is -1)
* regular (section number is positive)

An undefined symbol has a storage class of `IMAGE_SYM_CLASS_EXTERNAL`, a section number of `IMAGE_SYM_UNDEFINED` (zero), and a value of zero.

An undefined symbol with a storage class of `IMAGE_SYM_CLASS_WEAK_EXTERNAL` is a weak external symbol.
It is followed by an auxiliary record:

* When the `Characteristics` is `IMAGE_WEAK_EXTERN_SEARCH_ALIAS`, the weak external symbol behaves like a weak definition in ELF.
* (ARM64EC specific) When the `Characteristics` is `IMAGE_WEAK_EXTERN_SEARCH_ANTI_DEPENDENCY`, the weak external symbol is an anti-dependency alias.

PE requires explicit annotations for exported symbols and imported symbols in DLL files.
There are differences between code symbols and function symbols.

### COMDAT

Refer to [COMDAT and section group](https://maskray.me/blog/2021-07-25-comdat-and-section-group).

### Exported symbols

The linker builds `.edata`. The specification says:

> The export data section, named `.edata`, contains information about symbols that other images can access through dynamic linking. Exported symbols are generally found in DLLs, but DLLs can also import symbols.

### Imported code symbols

```c
// b.dll
__declspec(dllexport) void f() {}

// a.exe
void local(void) {}
void __declspec(dllimport) f(void);
int main(void) {
  local();
  f();
}
```

Linking `b.dll` gives us `b.lib` (see "Import files" above).
```asm
# b.dll
.globl f
f:

.section        .drectve,"yni"
.ascii  " -export:f"
```

`a.obj` has two function calls. The call to `f` references the prefixed symbol `__imp_f`.
```asm
# a.obj
  callq   local
  callq   *__imp_f(%rip)
```

`call *__imp_f(%rip)` is like `-fno-plt` codegen for ELF. In this case when we know that `f` is defined elsewhere, the generated code is more efficient.

When linking `a.exe`, we need to make the import file `b.lib` as an input file.
The linker parses the import file and creates a definition for `__imp_f` pointing to the import address table entry.

TODO import table

Actually, when `__imp_f` is defined, the unprefixed symbol `f` is also defined.
Normally, the unprefixed `f` is unused and will be discarded.
However, if the user code calls the unprefixed symbol (e.g. `call f`; like ELF `-fplt`), the `f` definition will be retained in the linker output and point to a thunk:
```asm
  call f  # generated code without using dllimport

f:  # x86-64 thunk
  jmpq *__imp_f(%rip)
```

Different architectures have different thunk implementations.
```asm
// x86-32 and x86-64
jmp *0x0   // references an entry in the import address table

// AArch32
mov.w ip, #0
mov.t ip, #0
ldr.w pc, [ip]

// AArch64
adrp x16, #0
ldr x16, [x16]
br x16
```

TODO link.exe will issue a warning.

MSVC supports an undocumented (as of May 2025) option `/d2ImportCallOptimization` that records the indirect call offsets into the `.impcall` section (<https://github.com/llvm/llvm-project/pull/121516>).
The Windows kernel loader will rewrite indirect calls to direct calls.

### Imported data symbols

```c
// b.dll
__declspec(dllexport) int var;

// a.exe
int local_var;
__declspec(dllimport) extern int var;
int main() { return local_var + var;  }
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

If `dllimport` is not specified, we get a referenced to the unprefixed symbol:
```asm
  movq    var(%rip), %rax
```

link.exe will report an error.

MinGW implements runtime pseudo relocations to patch the text section so that absolute pointers and relative offsets to the symbol will be rewritten to bind to the actual definition.
```asm
  movq var(%rip), %rax  # the runtime will rewrite this to point to the definition in b.dll
```

If the variable is defined out of the +-2GiB range from the current location, the runtime pseudo relocation can't fix the issue.
See [crt: Check pseudo relocations for overflows and error out clearly](https://github.com/mingw-w64/mingw-w64/commit/ca35236d9799af8a3d2f9baa35b60e6c11abeb24).

For a non-definition declaration, GCC conservatively thinks the variable may be defined in a DLL and generate indirection.
This is similar to a GOT code sequence in ELF.
```c
extern int extern_var;
int main() { return extern_var; }
```

```asm
// MSVC
  movl    extern_var(%rip), %eax

// GCC
  movq    .refptr.extern_var(%rip), %rax
  movl    (%rax), %eax

  .section        .rdata$.refptr.extern_var,"dr",discard,.refptr.extern_var
  .p2align        3, 0x0
  .globl  .refptr.extern_var
.refptr.extern_var:
  .quad   extern_var
```

## Non-dllexport definition and dllimport

A `dllimport` symbol referenced by an object file is normally satisfied by an import file.
link.exe allows another object file to provide the definition.
In such a case, link.exe will issue a warning ([Linker Tools Warning LNK4217](https://learn.microsoft.com/en-us/cpp/error-messages/tool-errors/linker-tools-warning-lnk4217)).
lld-link has implemented this feature for compatibility.

```sh
echo '__declspec(dllimport) int foo(); int main() { return foo(); }' > a.cc
echo 'int foo() { return 42; }' > b.cc
clang-cl -c a.cc b.cc
lld-link -nodefaultlib -entry:main a.obj b.obj
```
```
lld-link: warning: a.obj: locally defined symbol imported: int __cdecl foo(void) (defined in b.obj) [LNK4217]
```

## MinGW

MinGW provides auto exporting and auto importing features to make PE DLL files work like ELF shared objects.
When producing a DLL file, if no symbol is chosen to be exported, almost all symbols are exported by default (`--export-all-symbols`).

If an undefined symbol `$sym` is unresolved and `__imp_$sym` is defined, `$sym` will be aliased to `__imp_$sym`.
TODO: example

If the symbol `.refptr.$sym` is present, it will be aliased to  `__imp_$sym` as well.
mingw-w64 defaults to `-mcmodel=medium` and uses `.refptr.$sym`.
TODO: example

https://github.com/ziglang/zig/issues/9845

## Manual `__imp_` definition

The user can define `__imp_` instead of letting the linker does.

https://github.com/llvm/llvm-project/issues/57982
```
$ cat lto-dllimp1.c
void __declspec(dllimport) importedFunc(void);
void other(void);

void entry(void) {
    importedFunc();
    other();
}
$ cat lto-dllimp2.c
static void importedFuncReplacement(void) {
}
void (*__imp_importedFunc)(void) = importedFuncReplacement;

void other(void) {
}
```

## Range extension thunks

TODO

## Shared library comparison with ELF

The design of share libraries has major advancements around 1988.
Before 1988, there were shared libraries implementations in a.out and COFF objec file formats, but they had severe limitations, such as fixed addresses and the requirement of extra files like import files.

Such limitations are evidenced in _1986 Summer USENIX Technical Conference & Exhibition Proceedings_, _Shared Libraries on UNIX System V_ from AT&T.
Its shared library (presumably using the COFF object file format) must have a fixed virtual address, which is called "static shared library" in _Linkers and Loaders_'s term.

In 1988, SunOS 4.0 was released with an extended a.out binary format with dynamic shared library support.
Unlike previous static shared library schemes, the a.out shared libraries are position independent and can be loaded at different addresses.
The dynamic linker source code is available somewhere and I find that its GOT and PLT schemes are exacly like what we have for ELF today.

AT&T and Sun collaborated to create the first System V release 4 ABI (using ELF).
AT&T contributed the ELF object format.
Sun contributed all of the dynamic linking implementation from SunOS 4.x.
In 1992, SunOS 5.0 (Solaris 2.0) switched to ELF.

For ELF, the designers tried to make shared libraries similar to static libraries.
There is no need to annotate export and import symbols to work with shared libraries.

I cannot find more information about System V release 3's shared library support, but the Windows DLL is assuredly inspired by it, given that the PE object file format is based on COFF and the PE specification refers to COFF in numerous places.

So, is the shared library design in ELF more advanced? It is.
However, two aspects are worth deep thoughts.

* The manual export and import annotations have its stregth.
* Choices made to make ELF shared libraries flexible had major downsides.
  + Performance downside due to symbol interposition on the compiler side. See [_-fno-semantic-interposition_](/blog/2021-05-09-fno-semantic-interposition)
  + Performance downside due to symbol interposition on the linker and loader side. See [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic)
  + Underlinking problems exacebated by the `-z undefs` default in linkers. See [Dependency related linker options](/static/2021-06-13-dependency-related-linker-options).

## Limitations

The number of symbols cannot exceed 65535.
Several open-source projects have faced problems that a DLL file cannot export more than 65535 symbols.
(GNU ld has a diagnostic `error: export ordinal too large:`).

A section header has only 8 bytes for the name field. link.exe truncates long section names to 8 bytes.
For a section with a long name and the `IMAGE_SCN_MEM_DISCARDABLE` flag, lld uses a non-standard string table and issues a warning.

COMDAT limitation: MSVC link.exe will report a [duplicate symbol error](https://maskray.me/blog/2021-07-25-comdat-and-section-group) (error LNK2005) for an external symbol defined in an `IMAGE_COMDAT_SELECT_ASSOCIATIVE` section, even if it would be discarded after handling the leader symbol.
