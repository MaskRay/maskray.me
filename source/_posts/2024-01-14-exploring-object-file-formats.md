---
layout: post
title: Exploring object file formats
author: MaskRay
tags: [linker]
---

My journey with the LLVM project began with a deep dive into the world of lld and binary utilities.
Countless hours were spent unraveling the intricacies of object file formats and shaping LLVM's relevant components.
Though my interests have since broadened, object file formats remain a personal fascination, often drawing me into discussions around potential changes within LLVM.

This article compares several prominent object file formats, drawing upon my experience and insights.

At the heart of each format lies the representation of essential components like symbols, sections, and relocations.
For each control structure, We'll begin with ELF, a widely used format, before venturing into the landscapes of other notable formats.

<!-- more -->

## History of object file formats

Before delving into the technical side, I will share some notes about my archaeological journey.

### a.out

The a.out format was designed for PDP-11 and appeared on the first version of Unix.
The quantities were 16-bit, but can be naturally extended to 32-bit or 64-bit.

In Proceedings of the Summer 1990 USENIX Conference, _ELF: An Object File to Mitigate Mischievous Misoneism_ by James Q. Arnold provided some description.

> For 32-bit machines, the a.out format was extended in several ways. Most obviously, 16-bit quantities were enlarged to 32-bit values. The symbol table changed to allow names of unlimited length. Relocation entries also changed significantly. Larger programs and different relocation conventions made it necessary to associate a relocation entry with an explicit address, instead of relying on the implicit correspondence between program sections and relocation records.

Many Unix and Unix-like operating systems, including SunOS, HP-UX, BSD, and Linux, used a.out before switching to ELF.

The most noticeable extension is dynamic shared library support. (This feature is distinct from static shared library, where each shared library needs a fixed address in the address space.)
There are two flavors:

* In 1988, SunOS 4.0 was released with an extended a.out binary format with dynamic shared library support.
* In 1993, on NetBSD, <https://github.com/NetBSD/src/commit/97ca10e37476fb84a20a8ec4b0be3188db703670> (`A linker supporting shared libraries.`) and <https://github.com/NetBSD/src/commit/3d68d0acaed0a32f929b2c174146c62940005a18> (`A linker supporting shared libraries (run-time part).`) added shared library support similar to the SunOS scheme.

[FreeBSD a.out(5)](https://man.freebsd.org/cgi/man.cgi?a.out(5)) provides a nice description.

If you follow recent years' Linux kernel news, there were some discussions when Linux eventually [removed a.out support](https://git.kernel.org/linus/987f20a9dcce3989e48d87cff3952c095c994445) in 2022.

### COFF

a.out supports three fixed loadable sections TEXT, DATA, and BSS, which is too restrictive.
COFF introduces custom section support and allows up to 32767 sections. The ELF paper contains some remarks:

> Common Object File Format (COFF), was designed primarily to support electronic switching systems (the telephone network). Its distinguishing features were multiple sections (text, data, uninitialized memory, reserved memory, overlays, etc.), some support for multiple target processors, defined structures for symbol tables and relocations, and debugging information tailored for the C language.

According to `scnhdr.h` in System V Release 2 for NS32xxx, COFF was designed no later than 1982.
Then, System V Release 3 adopted COFF, which motivated a lot of follow-ups.

* Windows extended COFF to the [Portable Executable (PE) format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format). Symbian OS before 9 used PE as well.
* Texas Instruments [modified COFF](https://software-dl.ti.com/ccs/esd/documents/sdto_cgt_A-Brief-History-of-TI-Object-File-Formats.html) for its TI toolset and then switched to ELF.
* ECOFF used by Tru64 UNIX changed symbol representation.
* IBM developed XCOFF (COFF combined with the TOC module format concept, CSECT, etc) and used it for AIX.

Key drawbacks:

* Hard-wiring debugging information tailored for the C language into the symbol structure is complex, space-inefficient, and ugly.
* The auxiliary symbol record design is inflexible and inefficient.
* Not 32-bit-aligned symbol and section structures caused performance issue to earlier systems.
* No support for [weak symbols](/blog/2021-04-25-weak-symbol). PE implemented an inflexible weak definition mechanism called "weak externals".

### Mach-O

Carnegie Mellon University developed the Mach kernel as a proof of the microkernel concept.
The operating system used a format derived from a.out with minor modifications, named the Mach object file format. The abbreviation, Mach-O, is often used instead.
The NeXTSTEP operating system and then Darwin adopted Mach-O.

Dynamic shared library support on Mach-O came later than other object file formats.
In a NeXTSTEP manual released in 1995, I can find `MH_FVMLIB` (fixed virtual memory library, which appears to be a static shared library scheme) but not `MH_DYLIB` (used by modern macOS for .dylib files).

Key drawbacks:

* No [COMDAT](/blog/2021-07-25-comdat-and-section-group) support.
* Scarcity of relocation types.
* 255 section limit. `.subsections_via_symbols` has some downsides (discussed later).

In my opinion, Mach-O is the most limited among Mach-O/PE/ELF. However, I want to acknowledge certain innovative features like `.subsections_via_symbols` and `S_ATTR_LIVE_SUPPORT`.

### ELF

Frustrations and inherent constraints of COFF, coupled with a self-imposed byte order dilemma, AT&T introduced a groundbreaking format: Executable and Linking Format (ELF).
ELF revisited fixed content and hard-wired concepts in previous object file formats, removed unnecessary elements, and made control structures more flexible.

This pivotal shift was embraced by System V Release 4, marking a new era in object file format design.
In the 1990s, many Unix and Unix-like operating systems, including Solaris, IRIX, HP-UX, Linux, and FreeBSD, switched to ELF.

## Symbols

The minimum of a symbol control structure needs to encode the name, section, and value.
We can require that every symbol is defined in relation to some section. We can use a section index of zero to represent an undefined symbol.

In a minimum object file format with only few hard-coded sections (a.out), the section field can be omitted.
A type field can be used to decide whether the symbol can reference a function or a data object.

```c
// ELFCLASS32, 16 bytes
typedef struct {
  Elf32_Word	st_name;
  Elf32_Addr	st_value;
  Elf32_Word	st_size;
  unsigned char	st_info;
  unsigned char	st_other;
  Elf32_Half	st_shndx;
} Elf32_Sym;

// ELFCLASS64, 24 bytes
typedef struct {
  Elf64_Word st_name;     // index into the string table
  unsigned char st_info;  // type and binding
  unsigned char st_other; // visibility and others
  Elf64_Half st_shndx;    // section index
  Elf64_Addr st_value;
  Elf64_Xword st_size;
} Elf64_Sym;
```

The symbol name is represented as a 32-bit index into the string table.
A 32-bit integer suffices, while a 16-bit integer would be too small.

`st_shndx` uses a size-saving trick. The 16-bit member encodes a section index.
If the member is `SHN_XINDEX` (0xffff), then the actual value is contained in the associated section of type `SHT_SYMTAB_SHNDX`.
This is a very nice trick because the number of sections are almost always smaller than 0xff00.
In pathologic cases, there can be more sections, where a section of type `SHT_SYMTAB_SHNDX` is needed.

`st_info` specifies the symbol's type (4 bits) and binding (4 bits) attributes.
Types are allocated very conservatively and usually imply different linker behaviors.
The inherently different linker behaviors for symbol types are not that many.
So 4 bits seem small, they are sufficient in practice.
As we will learn, this is significantly smaller than COFF's type and storage class representation.
A symbol's binding is for the local/weak/global distinction.
The reserved 4 bits can accommodate more values, but only GNU reserves one value (`STB_GNU_UNIQUE`) (a misfeature in my opinion).

In COFF, function symbols can use an auxiliary symbol record to encode the size of function (`x_fsize`; `TotalSize` in PE).
In ELF, `st_size` is a fixed member, used for copy relocations and symbolizers.
If we eliminate copy relocations and don't need the symbolization heuristics, this field will become garbage.

Here is a demonstration if we remove `st_size`.
```c
// 16 bytes
struct Elf64_Sym_minimized {
  Elf64_Word st_name;     // index into the string table
  unsigned char st_info;  // type and binding
  unsigned char st_other; // visibility and others
  Elf64_Half st_shndx;    // section index
  Elf64_Addr st_value;
} Elf64_Sym;
```

### Symbols (a.out)

```c
// a.out (System V), 16 bytes
struct nlist {
  char n_name[8];
#if pdp11
  int n_type;
#else
  char n_type;
  char n_other;
  short n_desc;
#endif
  unsigned n_value;
};
```

a.out uses a `nlist` to represent a symbol table entry.
In the original format for PDP-11, the assembler generates symbols of at most 7 bytes.
`n_name[8]` can hold the name with a NUL end.
Unix's appreciation of shorter identifier names is related to this:)

To support longer names, extensions add a string table after the symbol table, and allow `n_name` to be interpreted as an index (`n_strx`) into the string table.
This member then becomes a size-saving trick by inlining a short name (8 bytes or less) into the structure.
Some variants, like binutils' 64-bit a.out format, use an index exclusively and removed `n_name`.

`n_type`, broken down into three sub-fields, describes whether a symbol is defined or undefined, external or local, and the symbol type.
The values listed on the FreeBSD manpage are also used on PDP-11.

For a defined symbol, `n_type` describes whether it is relative to TEXT, DATA, or BSS.

### Symbols (COFF)

```c
// COFF (System V Release 3), 18 bytes in the absence padding
struct syment {
  union {
    char _n_name[SYMNMLEN]; /* old COFF version */
    struct {
      long _n_zeroes; /* new == 0 */
      long _n_offset; /* offset into string table */
    } _n_n;
  } _n;
  unsigned long n_value; /* value of symbol */
  short n_scnum; /* section number */
  unsigned short n_type; /* type and derived type */
  char n_sclass; /* storage class */
  char n_numaux; /* number of aux. entries */
};
```

COFF adopts a.out's approach to save space in symbol names.
This likely made sense when most symbols were shorter.
However, with today's often lengthy symbol names, this inlining technique complicates code and increases the control structure size (from 4 to 8 bytes).

The section number is a 16-bit signed integer, supporting up to 32,767 sections. Positive values indicate a section index, while special values include:

* `N_UNDEF` (0): Undefined symbol (distinct from a.out's `n_type` representation).
* `N_ABS` (-1): Symbol has an absolute value.
* `N_DEBUG` (-2): Special debugging symbol (value is meaningless).

COFF's `n_type` and `n_sclass` encode C' type and storage class information.
PE assigns longer names to these types and storage classes longer names, e.g., `IMAGE_SYM_TYPE_CHAR/IMAGE_SYM_TYPE_SHORT`, `IMAGE_SYM_CLASS_AUTOMATIC/IMAGE_SYM_CLASS_EXTERNAL`.
While values are mostly consistent, minor differences exist:

* PE's `IMAGE_SYM_TYPE_VOID` (1) is different from System V Release 3's `#define T_ARG 1 /* function argument (only used by compiler) */`.
* PE's `IMAGE_SYM_CLASS_WEAK_EXTERNAL` (105) is different from System V Release 3's `#define C_ALIAS 105 /* duplicate tag */`.

Symbols with `C_EXT` (`IMAGE_SYM_CLASS_EXTERNAL`) are global and added to the linker's global symbol table, akin to ELF's `STB_GLOBAL` symbol binding.

System V ships a symbolic debugger (sdb), which utilizes `n_type` and `n_sclass`.
If we acknowledge that the debugging information format is outdated, `n_type` and `n_class` serve as a wasteful counterpart to to ELF's `st_info`.

`n_numaux` relates to Auxiliary Symbol Records, allowing extra information but introducing non-uniform symbol table entries.
While seemingly beneficial, their use cases are limited and could often be encoded using separate sections.
In PE, an auxiliary symbol record can represent weak definitions, but weak references are not supported.
They can also provide extra information to section symbols.

ECOFF defines Local Symbol Entry (SYMR) and External Symbol Entry (EXTR).

```c
typedef struct {
  coff_long value;
  coff_int iss;
  coff_uint st : 6;
  coff_uint sc : 5;
  coff_uint reserved : 1;
  coff_uint index : 20;
} SYMR, *pSYMR;

typedef struct {
  SYMR asym;
  coff_uint jmptbl:1;
  coff_uint cobol_main:1;
  coff_uint weakext:1;
  coff_uint reserved:29;
  coff_int ifd;
} EXTR, *pEXTR;
```

### Symbols (Mach-O)

```c
// Mach-O, 12 bytes
struct nlist {
  uint32_t n_strx;
  uint8_t n_type;
  uint8_t n_sect;
  int16_t n_desc;
  uint32_t n_value;
};

// Mach-O, 16 bytes
struct nlist_64 {
  uint32_t n_strx;
  uint8_t n_type;
  uint8_t n_sect;
  uint16_t n_desc;
  uint64_t n_value;
};
```

Mach-O's `nlist` and `nlist_64` are not that different from a.out's, with `n_other` changed to `n_sect` to indicate the section index.
The 8-bit n_sect field restricts representable sections to 255 without out-of-band data (discussed later).
If we extend `n_sect` to 32-bit, with alignment padding the structure size will increase to 24 bytes, the same as `Elf64_Sym`.

Like a.out, the `N_EXT` bit of `n_type` indicates an external symbol.
The `N_PEXT` bit indicates a private external symbol.

Key bits in `n_desc` are `N_WEAK_DEF`, `N_WEAK_REF`, and `N_ALT_ENTRY`.

## Sections

```c
// ELF, 40 bytes
typedef struct {
	Elf32_Word	sh_name;
	Elf32_Word	sh_type;
	Elf32_Word	sh_flags;
	Elf32_Addr	sh_addr;
	Elf32_Off	sh_offset;
	Elf32_Word	sh_size;
	Elf32_Word	sh_link;
	Elf32_Word	sh_info;
	Elf32_Word	sh_addralign;
	Elf32_Word	sh_entsize;
} Elf32_Shdr;

// ELF, 64 bytes
typedef struct {
	Elf64_Word	sh_name;
	Elf64_Word	sh_type;
	Elf64_Xword	sh_flags;
	Elf64_Addr	sh_addr;
	Elf64_Off	sh_offset;
	Elf64_Xword	sh_size;
	Elf64_Word	sh_link;
	Elf64_Word	sh_info;
	Elf64_Xword	sh_addralign;
	Elf64_Xword	sh_entsize;
} Elf64_Shdr;
```

The section name is represented as a 32-bit index into the string table.
If we use a 16-bit integer, a large number of section names with a symbol suffix (e.g. `.text.foo` `.text.bar`) could make the index overflow.

`sh_type` categorizes the section's contents and semantics. It avoids hard-coding magic names in many scenarios.
Technically a 16-bit type could work pretty well but was deemed insufficient for flexibility.

`sh_flags` describe miscellaneous attributes, e.g. writable and executable permissions, and whether the section should appear in a loadable segment.
This member is 32-bit in `Elf32_Shdr` while 64-bit in `Elf64_Shdr`.
In practice no architecture defines flags for bits 32 to 63, therefore this member is somewhat wasteful.

Location and size. `sh_offset` gives the byte offset from the beginning of the file to the first byte in the section.
To support object files larger than 4GiB, this member has to be 64-bit.
`sh_size` gives the section's size in bytes. A section type of `SHT_NOBITS` occupies no space in the file.
To support sections larger than 4GiB, this member has to be 64-bit.

Address and alignment.
`sh_addr` describes the address at which the section's first byte should reside for an executable or shared object.
It should be zero for relocatable files.
`sh_addralign` holds the address alignment. In practice this member must be a power of 2 even if the generic ABI does not require so.
This member is 64-bit in ELF64, which allows an alignment up to `2**63`.
In practice, an alignment larger than the page size (or the largest huge page size, if huge pages are enabled) does not make sense, and a maxiumm value of 2**31 is sufficient.
Therefore, we could use a log2 value to hold the alignment.

Connection information. `sh_link` holds a section index. `sh_info` holds either a section index or a symbol index.
If you recall that `st_shndx` is 16 bits for very solid reason, you will know that the two fields are somewhat wasteful.

For a table of fixed-size entries, `sh_entsize` holds the entry size in bytes.
In some use cases this member is not a power of two.
In practice, one byte suffices.

While ELF's section header structure is designed for flexibility, potential optimizations could reduce its size without significant loss of functionality.
By using smaller data types for `sh_flags`, `sh_link`, `sh_info`, and `sh_entsize` based on practical needs, we could make the structure significantly smaller.
```
// 32 bytes
struct Elf32_Shdr_minimized {
  Elf32_Word	sh_name;
  Elf32_Word	sh_type;    // Making this uint16_t and reordering it can decrease the size to 28 bytes
  Elf32_Word	sh_flags;
  Elf32_Addr	sh_addr;
  Elf32_Off	sh_offset;
  Elf32_Word	sh_size;
  uint8_t	sh_addralign;
  uint8_t	sh_entsize;
  Elf32_Half	sh_link;
  Elf32_Half	sh_info;
};

// 40 bytes
struct Elf64_Shdr_minimized {
  Elf64_Word sh_name;
  Elf64_Word sh_flags;
  Elf64_Addr sh_addr;
  Elf64_Off sh_offset;
  Elf64_Xword sh_size;
  Elf64_Half sh_type;
  uint8_t sh_addralign;
  uint8_t sh_entsize;
  Elf64_Half sh_link;
  Elf64_Half sh_info;
};
```

Reducing `sh_type` into 2 bytes loses flexibility a bit.
If this deems insufficient, we could take 3 bits from `sh_addralign` (by turning it into a bitfield) and give them to `sh_type`.

### Sections (COFF)

```
// COFF (System V Release 3), 40 bytes, when sizeof(long) == 4
struct scnhdr {
  char            s_name[8];      /* section name */
  long            s_paddr;        /* physical address */
  long            s_vaddr;        /* virtual address */
  long            s_size;         /* section size */
  long            s_scnptr;       /* file ptr to raw data for section */
  long            s_relptr;       /* file ptr to relocation */
  long            s_lnnoptr;      /* file ptr to line numbers */
  unsigned short  s_nreloc;       /* number of relocation entries */
  unsigned short  s_nlnno;        /* number of line number entries */
  long            s_flags;        /* flags */
};

// PE, 40 bytes
struct section {
  char Name[8];
  uint32_t VirtualSize;
  uint32_t VirtualAddress;
  uint32_t SizeOfRawData;
  uint32_t PointerToRawData;
  uint32_t PointerToRelocations;
  uint32_t PointerToLineNumbers;
  uint16_t NumberOfRelocations;
  uint16_t NumberOfLineNumbers;
  uint32_t Characteristics;
};
```

PE's section control structure demonstrates a minor modification compared to COFF, `s_paddr => VirtualSize`.

The presented structure measures as 40 bytes when `long` is 4 bytes.
If we extend `s_paddr, s_vaddr, s_size, s_scnptr, s_relptr, s_lnnoptr` to 8 bytes, the structure will be of 64 bytes.

The section name supports up to 8 bytes.
A longer name would require an extension similar to the symbol control structure.

Encoding both `s_paddr` and `s_vaddr` is wasteful.
ELF encodes the physical address in the segment and therefore removes the member from its section structure.

COFF embeds the location and size of relocations into the section structure.
This is actually pretty nice. A 16-bit `s_nreloc` may appear restritive but is sufficient for relocatable files.
In practice, the number of relocations can exceed 65536 for a single section using relocatable linking.

`s_lnnoptr` and `s_nlnno` point to line number entries, which relate addresses to source file line numbers.
The embedded nature is inflexible.
```c
/*  There is one line number entry for every "breakpointable" source line in a
section. Line numbers are grouped on a per function basis; the first entry in a
function grouping will have l_lnno = 0 and in place of physical address will be
the symbol table index of the function name. */
struct lineno {
  union {
    long l_symndx;	/* sym. table index of function name iff l_lnno == 0 */
    long l_paddr;   /* (physical) address of line number */
  } l_addr;
  unsigned short l_lnno ;	/* line number */
};
```

This simple format is deprecated.
In DWARF, special opcodes in line number information can encode the information in a more space-efficient way and present more information like the column number.

### Sections (Mach-O)

```c
// Mach-O, 80 bytes
struct section_64 {
  char sectname[16];
  char segname[16];
  uint64_t addr;
  uint64_t size;
  uint32_t offset;
  uint32_t align;
  uint32_t reloff;
  uint32_t nreloc;
  uint32_t flags;
  uint32_t reserved1; // index into the indirect symbol table specified by LC_DYSYMTAB
  uint32_t reserved2; // __TEXT,__stub: entry size; otherwise: 0
  uint32_t reserved3;
};
```

How does Mach-O end up with such a huge section structure? Let's find out...

A Mach-O binary is divided into segments, each housing one or more sections.
The section structure encodes the section name and the segment name, both can be up to 16 bytes.
This representation allows the section names to be read without a string table, but restrictive for descriptive names.
Section semantics are derived from the name (unlike ELF).

The segment name is redundantly encoded within the section structure.
We could derive the segment from the section name and flags, e.g., `S_ATTR_SOME_INSTRUCTIONS => __TEXT` , `S_ZEROFILL => ZeroFill __DATA` .

There is a severe limitation: maximum of 255 sections due to `nlist::n_sect` being a `uint8_t`.
This is apparently too restrictive.
Thankfully, an innovative feature `.subsections_via_symbols` overcomes the limitation.
The feature uses a monolithic section with "atoms" dividing it into pieces (subsections).
This is more size-efficient than ELF's `-ffunction-sections -fdata-sections -fno-unique-section-names`.
However, there are assembler limitations, difficulty of function prefix data, relocation processing complexity, and potential loss of ability to ensure that two non-local symbols are not reordered.

Like COFF, Mach-O embeds the location and size of relocations into the section structure.

`reserved1` and `reserved2` are used similarly to ELF's connection information.

`__TEXT,__stub` (like ELF's `.plt`), `__TEXT,__got` (like ELF's `.got`), `__TEXT,__la_symbol_ptr` (like ELF's `.got.plt`), and `__DATA,__thread_ptrs` set `reserved1` as an index into the indirect symbol table (the offset is specified by `indirectsymoff` in a `LC_DYSYMTAB` command).

For `__TEXT,__stub`, `reserved2` is the size of one entry, e.g., 6 for x86-64 (`jmpq *__la_symbol_ptr(%rip)`). This is analogous to ELF x86-64's [`DT_X86_64_PLTSZ`](/blog/2023-02-19-linker-notes-on-x86#mark-plt).
For other sections, `reserved2` is zero.

## Relocations

```c
// ELFCLASS32, 8 bytes
typedef struct {
  Elf32_Addr r_offset;
  Elf32_Word r_info;
} Elf32_Rel;
// ELFCLASS32, 12 bytes
typedef struct {
  Elf32_Addr r_offset;
  Elf32_Word r_info;
  Elf32_Sword r_addend;
} Elf32_Rela;

// ELFCLASS64, 16 bytes
typedef struct {
  Elf64_Addr r_offset;
  Elf64_Xword r_info;
} Elf64_Rel;
// ELFCLASS64, 24 bytes
typedef struct {
  Elf64_Addr r_offset;
  Elf64_Xword r_info;
  Elf64_Sxword r_addend;
} Elf64_Rela;
```

`r_info` specifies the symbol table index with respect to which the relocation must be made, and the type of relocation to apply.

* ELFCLASS32: 8-bit type, 24-bit symbol index
* ELFCLASS64: 32-bit type, 32-bit symbol index

There are two variants, REL and RELA. Let's quote the generic ABI:

> As specified previously, only Elf32_Rela and Elf64_Rela entries contain an explicit addend. Entries of type Elf32_Rel and Elf64_Rel store an implicit addend in the location to be modified. Depending on the processor architecture, one form or the other might be necessary or more convenient. Consequently, an implementation for a particular machine may use one form exclusively or either form depending on context.

Relocatable files need a lot of relocatable types while executables and shared objects need only a few.
The former is often called static relocations while the latter is called dynamic relocations.

Of the few dynamic relocation types, most do not need the addend member. lld provides an option `-z rel` to use `SHT_REL/DT_REL` dynamic relocations.

If we disregard the REL dynamic relocation scenario, then all modern architectures use RELA exclusively.
Most architectures encode the immediate with only few bits, which are inadequate for many relocatable file uses.

ELFCLASS64, with its 64-bit members, doubles the size compared to ELFCLASS32's 32-bit members.
Since relocations often comprise a substantial portion of object files, this size difference can lead to user concerns.
However, in practice, a 24-bit symbol index is often sufficient, even in 64-bit contexts.
Therefore, if a 64-bit architecture's relocation type requirements are less than 256, ELFCLASS32 can be a viable and more size-efficient option.

In March 2024, I proposed [CREL](/blog/2024-03-09-a-compact-relocation-format-for-elf) as an alternative relocation format.

### Relocations (a.out)

```c
// a.out (System V Release 2), 8 bytes
struct relocation_info {
  int	   r_address;
  unsigned r_symbolnum : 24,
           r_pcrel : 1,
           r_length : 2,
           r_extern : 1,
           r_pad : 4;
};
```

`r_symbolnum` mirrors ELF's `ELF32_R_SYM`.

The other bitfields, resembling ELF's `ELF32_R_TYPE`, but split into distinct fields:

* `r_pcrel`
* `r_length`
* others

Reserving dedicated semantics for individual bits can limit adaptability.
COFF and ELF opted to remove bitfields in favor of a type to provide greater flexibility.

### Relocations (COFF)

```c
// COFF, 10 bytes on disk, 12 bytes with alignment padding
struct reloc {
  long r_vaddr;
  long r_symndx;
  unsigned short r_type;
};
```

This format resembles ELF's `Elf32_Rel`.

`r_vaddr` gives the virtual address of the location at which to apply the relocation action.
If we interpret `r_vaddr` as an offset (as PE does) and restrict section size to 32 bits, we could reuse this structure for 64-bit architectures.

`r_symndx` is a 32-bit symbol table index.

`r_type` is a 16-bit relocation type, limited in number compared to ELF.

COFF generally supports fewer relocation types than ELF.
System V Release 3 defines very few relocations for each architecture.
In binutils, `include/coff/*.h` files define relocations for more architectures.

While ELF uses the REL/RELA for both relocatable files and executables, in PE image files, the import address table and base relocation table (`.reloc`) are a completely different design.

### Relocations (Mach-O)

```c
// Mach-O, 8 bytes
struct relocation_info {
   int32_t	r_address;	/* offset in the section to what is being
				   relocated */
   uint32_t     r_symbolnum:24,	/* symbol index if r_extern == 1 or section
				   ordinal if r_extern == 0 */
		r_pcrel:1, 	/* was relocated pc relative already */
		r_length:2,	/* 0=byte, 1=word, 2=long, 3=quad */
		r_extern:1,	/* does not include value of sym referenced */
		r_type:4;	/* if not 0, machine specific relocation type */
};

struct scattered_relocation_info {
#ifdef __BIG_ENDIAN__
  uint32_t r_scattered : 1, r_pcrel : 1, r_length : 2, r_type : 4,
      r_address : 24;
#else
  uint32_t r_address : 24, r_type : 4, r_length : 2, r_pcrel : 1,
      r_scattered : 1;
#endif
  int32_t r_value;
};
```

Mach-O's relocation structure closely mirrors a.out's with adapted `r_symbolnum` meaning.
When `r_extern == 0` (local), the `r_symbolnum` member references a section index instead of a symbol index.
This is to support custom sections, breaking the three-section limitation (text, data, and bss) of traditional a.out.

As aforementioned, dedicating bits to bitfields (`r_pcrel`, `r_length`, and `r_scattered` greatly restricted the number of relocation types.

Related to the relocation type limitation, a `.long foo - .` in a data section requires a pair of relocations, `SUBTRACTOR` and`/UNSIGNED`.
I have some notes on [Port LLVM XRay to Apple systems](/blog/2023-06-18-port-llvm-xray-to-apple-systems).

Mach-O uses a number of sections in the `__LINKEDIT` segment to communicate information to dyld.

## File header

### File header (a.out)

Dennis MacAlistair Ritchie's [`A.OUT (V)` manpage](https://www.bell-labs.com/usr/dmr/www/man51.pdf) (1971) describes the original a.out format.
The header contains 6 words.

* a "br .+14" instruction (205(8))
* The size of the program text
* The size of the symbol table
* The size of the relocation bits area
* The size of a data area
* A zero word (unused at present)

The text relocations are implicit.

Later versions introduced new magic numbers, separated text relocations and data relocations, and added an entry point (`a_entry`).

## Size comparison

TODO

## Size reduction opportunities

ELFCLASS32 structures are already compact, offering limited size reduction potential.
ELFCLASS64 structures, while flexible, can be optimized by sacrificing some flexibility (64-bit quantities).
The 64-bit symbol control structure is compact, but section and relocation's are quite wasteful if we can sacrifice some flexibility.

As the ELF paper acknowledges, "Relocatable and executable files do not necessarily have the same constraints, and we considered using two file formats. Eventually, we decided the two activities were similar enough that a single format would suffice."
There are more tools inspecting executables than relocatable files.
So, naturally, we might want to change just relocatable files.
Can we use ELFCLASS32 relocatable files for 64-bit architectures?

Well, x86-64 and AArch64 make a clear distinct of ELFCLASS32 and ELFCLASS64. ELFCLASS32 is for ILP32 (x32, aarch64_ilp32) while ELFCLASS64 is for LP64.
However, the discontinued Itanium architecture sets a precedent that ELFCLASS32 can be used for LP64 programs.
Quoting its psABI (_Intel Itanium Processorspecific Application Binary Interface (ABI)_).

> For Itanium architecture ILP32 relocatable (i.e. of type ET_REL) objects, the file class value in e_ident[EI_CLASS] must be ELFCLASS32. For LP64 relocatable objects, the file class value may be either ELFCLASS32 or ELFCLASS64, and a conforming linker must be able to process either or both classes. ET_EXEC or ET_DYN object file types must use ELFCLASS32 for ILP32 and ELFCLASS64 for LP64 programs.
>
> Addresses appearing in ELFCLASS32 relocatable objects for LP64 programs are implicitly extended to 64 bits by zero-extending.
>
> Note: Some constructs legal in LP64 programs, e.g. absolute 64-bit addresses outside the 32-bit range, may require use of an ELFCLASS64 relocatable object file.

Given the prior art, it seems promising to allow ELFCLASS32 when the code size concerns people.
Ideally there should be a marker to distinguish ILP32 and LP64-using-ELFCLASS32 object files.

The primary changes reside in the assembler and linker.
It's also important to ensure that binary manipulation programs (like objcopy) and dump tools are happy with them.

Further optimization potential lies in exploring the use of `Elf32_Rel` instead of `Elf32_Rela` for even smaller relocations.

### Replacing control structures

This approach is independent of whether ELFCLASS32 is adopted and can be applied to both ELFCLASS32 and ELFCLASS64.
The ELF paper is clear, "ELF allows extension and redefinition for other control structures."
However, caution is warranted due to the significant impact on the ecosystem as many tools rely on the existing structures.

One promising example is `Elf32_Shdr_minimized`, a custom structure reduced to 32 bytes from the standard `Elf32_Shdr`'s 40 bytes.
While I would be nervous, but if we reduce `sh_type` to a `uint16_t`, the structure size can reduce to 28 bytes.

## stabs

Earlier debuggers operated using a debugging information format called "stabs" (short for symbol table entries; dating back to at least UNIX/32V in 1979).
Stabs is encoded using extra symbol table entries in the a.out object file format.

```asm
.stabs "string",type,other,desc,value
.stabn type,other,desc,value
.stabd type,other,desc
```

Stabs was ported to COFF for System V Release 2, used on some machines.
System V Release 4 switched to ELF and abandoned stabs in favor of a newly developed format called DWARF.
Its debugger sdb was rewritten to support DWARF, and stabs was no longer supported.
(The first version of DWARF was later published by the UNIX International Programming Languages Special Interest Group (SIG) in January 1992.)

However, stabs continued to be used in other operating systems, including *BSD, AIX, and IRIX.
For example, the GNU assembler added stabs support for ELF (`n_strx` is 32-bit).

GCC 13 [removed stabs support](https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;h=7e0db0cdf01e9c885a29cb37415f5bc00d90c029).

Stabs is less efficient than DWARF.
When compiling a non-trivial program (so that the boilerplate in DWARF is less significant), you may observe that `.stab` and `.stabstr` consume more space than `.debug_*` sections, even if DWARF is more expressive and contains more information.

## Heterogeneity and challenge

While the diversity of operating systems and architectures poses complexity for application developers, the object file format heterogeneity presents a unique challenge for toolchain development, probably not very tangible by application developers and users.

Integrating features like Link Time Optimization (LTO), Profile-Guided Optimization (PGO), and sanitizers has complexity due to object file format-specific limitations and nuances.
While most developers primarily concern themselves with a specific format, they still need to tread carefully during development to avoid disruptions to other platforms.

## TODO

WebAssembly
