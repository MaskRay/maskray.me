layout: post
title: "Light ELF: exploring potential size reduction"
author: MaskRay
tags: [elf,linker]
---

ELF's design emphasizes [natural size and alignment guidelines for its control structures](https://maskray.me/blog/2024-03-09-a-compact-relocation-format-for-elf).
While ensured efficient processing in the old days, this can lead to larger file sizes.
I propose "Light ELF" (`EV_LIGHT`, version 2) – a whimsical exploration inspired by Light Elves of Tolkien's legendarium (who had seen the light of the Two Trees in Valinor).

<!-- more -->

In a light ELF file, the `e_version` member of the ELF header is set to 2.
`EV_CURRENT` remains 1 for backward compatibility.

```c
#define EV_NONE 0
#define EV_CURRENT 1
#define EV_LIGHT 2
```

When linking a program, traditional ELF (version 1) and light ELF (version 2) files can be mixed together.

## Relocations

Light ELF utilizes [CREL](/blog/2024-03-09-a-compact-relocation-format-for-elf) for relocations. [REL and RELA](/blog/2024-01-14-exploring-object-file-formats#relocations) from traditional ELF are unused.

Existing lazy binding schemes rely on random access to relocation entries within the `DT_JMPREL` table.
Due to CREL's sequential nature, keeping lazy binding requires a memory allocation that holds decoded JUMP_SLOT relocations.

## Section header table

Traditional ELF (version 1) [section header tables](/blog/2024-01-14-exploring-object-file-formats#sections) can be large.
Light ELF addresses this through a compact section header table format signaled by `e_shentsize == 0` in the ELF header.

[A compact section header table for ELF](/blog/2024-03-31-a-compact-section-header-table-for-elf) contains the detail.
Its current version is copied below for your convenience.

`nshdr` denotes the number of sections (including `SHN_UNDEF`).
The section header table (located at `e_shoff`) begins with `nshdr` `Elf_Word` values.
These values specify the offset of each section header relative to `e_shoff`.

Following these offsets, `nshdr` section headers are encoded.
Each header begins with a `presence` byte indicating which subsequent `Elf_Shdr` members use explicit values vs. defaults:

* `sh_name`, ULEB128 encoded
* `sh_type`, ULEB128 encoded (if `presence & 1`), defaults to `SHT_PROGBITS`
* `sh_flags`, ULEB128 encoded (if `presence & 2`), defaults to 0
* `sh_addr`, ULEB128 encoded (if `presence & 4`), defaults to 0
* `sh_offset`, ULEB128 encoded
* `sh_size`, ULEB128 encoded (if `presence & 8`), defaults to 0
* `sh_link`, ULEB128 encoded (if `presence & 16`), defaults to 0
* `sh_info`, ULEB128 encoded (if `presence & 32`), defaults to 0
* `sh_addralign`, ULEB128 encoded as log2 value (if `presence & 64`), defaults to 1
* `sh_entsize`, ULEB128 encoded (if `presence & 128`), defaults to 0

In traditional ELF, `sh_addralign` can be 0 or a positive integral power of two, where 0 and 1 mean the section has no alignment constraints.
While the compact encoding cannot encode `sh_addralign` value of 0, there is no loss of generality.

Example C++ code that decodes a specific section header:
```cpp
// readULEB128(const uint8_t *&p);

const uint8_t *sht = base + ehdr->e_shoff;
const uint8_t *p = sht + ((Elf_Word*)sht)[i];
uint8_t presence = *p++;
Elf_Shdr shdr = {};
shdr.sh_name = readULEB128(p);
shdr.sh_type = presence & 1 ? readULEB128(p) : ELF::SHT_PROGBITS;
shdr.sh_flags = presence & 2 ? readULEB128(p) : 0;
shdr.sh_addr = presence & 4 ? readULEB128(p) : 0;
shdr.sh_offset = readULEB128(p);
shdr.sh_size = presence & 8 ? readULEB128(p) : 0;
shdr.sh_link = presence & 16 ? readULEB128(p) : 0;
shdr.sh_info = presence & 32 ? readULEB128(p) : 0;
shdr.sh_addralign = presence & 64 ? 1UL << readULEB128(p) : 1;
shdr.sh_entsize = presence & 128 ? readULEB128(p) : 0;
```

While the current format allows for O(1) in-place random access of section headers using offsets at the beginning of the table, this access pattern seems uncommon in practice.
At least, I haven't encountered (or remembered) any instances within the llvm-project codebase.
Therefore, I'm considering removing this functionality.

In a release build of llvm-project (`-O3 -ffunction-sections -fdata-sections -Wa,--crel`, the traditional section header tables occupy 16.4% of the `.o` file size while the compact section header table drastically reduces the ratio to 4.7%.

## Symbol table

Like other sections, [symbol table](/blog/2024-01-14-exploring-object-file-formats#symbols) and string table sections (`SHT_SYMTAB` and `SHT_STRTAB`) can be [compressed through `SHF_COMPRESSED`](/blog/2023-07-07-compressed-arbitrary-sections).
However, compressing the dynamic symbol table (`.dynsym`) and its associated string table (`.dynstr`) is not recommended.

Symbol table sections have a non-zero `sh_entsize`, which remains unchanged after compression.

The string table, which stores symbol names (also section names in LLVM output), is typically much larger than the symbol table itself.
To reduce its size, we can utilize a text compression algorithm.
While compressing the string table, compressing the symbol table along with it might make sense, but using a compact encoding for the symbol table itself won't provide significant benefits.

## Program headers

Program headers, while individually large (each `Elf64_Phdr` is 56 bytes) and no random access is needed, typically have a limited quantity within an executable or shared object.
Consequently, their overall size contribution is relatively small.
Light ELF maintains the existing format.

## Section compression

Compressed sections face a challenge due to header overhead especially for ELFCLASS64.

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

The overhead and [alignment padding](/blog/2022-01-23-compressed-debug-sections#alignment) limit the effectiveness when used with features like `-ffunction-sections` and `-fdata-sections` that generate many smaller sections.
For example, I have found that the large `Elf64_Chdr` makes evaluating compressed `.rela.*` sections difficult.
Light ELF addresses this challenge by introducing an header format of smaller footprint:

* `ch_type`, ULEB128 encoded
* `ch_size`, ULEB128 encoded
* `ch_addralign`, ULEB128 encoded as log2 value

This approach allows Light ELF to represent the header information in just 3 bytes for smaller sections, compared to the 24 bytes required by the traditional format.
The content is no longer guaranteed to be word-aligned, a property that most compression libraries don't require anyway.

Furthermore, [compressed sections with the `SHF_ALLOC` flag](https://groups.google.com/g/generic-abi/c/HUVhliUrTG0) are allowed. 
Using them outside of relocatable files needs caution, though.

## Experiments

I have developed a Clang/lld prototype that implements compact section header table and CREL (<https://github.com/MaskRay/llvm-project/tree/april-2024>).

`.o size`  |sht size   | build
-----------+-----------+----------------------------
 136599656 |  18429824 | -O3
 112088088 |  18431616 | -O3 -Wa,--crel
  97517175 |   3860703 | -O3 -Wa,--crel,--cshdr
2166435360 | 260303360 | -g
1755222784 | 260305152 | -g -Wa,--crel
1557420523 |  62502891 | -g -Wa,--crel,--chsdr

WebAssembly has a dense object file format. We can make a comparison with WebAssembly.

```sh
clang -c --target=wasm64 a.cc
clang -c -Wa,--crel,--allow-experimental-crel,--cshdr -fno-asynchronous-unwind-tables -fno-exceptions -fno-ident -fno-addrsig -fno-unique-section-names a.cc
```

## Light ELF: a thought experiment

By now, you might have realized that post is about a joke.
While bumping `e_version` and modifying `Elf_Chdr` might not be feasible, it's interesting to consider the possibilities of compact section headers and compressed symbol/string tables.
Perhaps this can spark some interesting discussions!
