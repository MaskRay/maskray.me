layout: post
title: A compact section header table for ELF
author: MaskRay
tags: [elf,linker]
---

ELF's design emphasizes [natural size and alignment guidelines for its control structures](/blog/2024-03-09-a-compact-relocation-format-for-elf).
However, this approach has substantial size drawbacks.

In a release build of llvm-project (`-O3 -ffunction-sections -fdata-sections`, the section header tables occupy 13.4% of the `.o` file size.

I propose an alternative section header table format that is signaled by `e_shentsize == 0` in the ELF header.
`e_shentsize == sizeof(Elf64_Shdr)` (or the 32-bit counterpart) selects the traditional section header table format.

<!-- more -->

The content bgins with a ULEB128-encoded value `nshdr`: the number of sections (including `SHN_UNDEF`).
`nshdr` section headers follow `nshdr`.

<!-- The compact section header table (located at `e_shoff`) begins with `nshdr` `Elf_Word` values.
These values specify the offset of each section header relative to `e_shoff`.
Following these offsets, `nshdr` section headers are encoded. -->

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

Example C++ code that decodes a section header:
```cpp
// readULEB128(const uint8_t *&p);

const uint8_t *sht = base + ehdr->e_shoff;
const uint8_t *p = sht + offsets[i];
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

While O(1) random access isn't supported, this is often addressed by applications building their own data representations.
Alternatively, for simpler applications, a prescan can be performed to determine the starting offset of each header beforehand.

In a release build of llvm-project (`-O3 -ffunction-sections -fdata-sections -Wa,--crel`, the traditional section header tables occupy 16.4% of the `.o` file size while the compact section header table drastically reduces the ratio to 4.7%.

## Experiments

I have developed a Clang/lld prototype that implements compact section header table and CREL.
<https://github.com/MaskRay/llvm-project/tree/demo-cshdr>

`.o size`  |sht size   | build
-----------+-----------+----------------------------
 136599656 |  18429824 | -O3
 112088088 |  18431616 | -O3 -Wa,--crel
  97517175 |   3860703 | -O3 -Wa,--crel,--cshdr
2166435360 | 260303360 | -g
1755222784 | 260305152 | -g -Wa,--crel
1557420523 |  62502891 | -g -Wa,--crel,--chsdr

<!--
for i in s2-custom{0,1,2} s2-custom-deb{0,1,2}; do ruby -e 'tot=totsht=0; Dir.glob("/tmp/out/'$i'/**/*.o").each{|f| s=File.size(f); sht=%x{/tmp/Rel/bin/llvm-readelf --elf-output-style=JSON -h #{f}|jq ".[]|.ElfHeader.SectionHeaderOffset"}.to_i; tot+=s; totsht+=s-sht }; printf "%10d | %10d | %s\n", tot, totsht, "'$i'"'; done
-->

## Alternatives

The WebAssembly object file format implemented by LLVM adopts the following design that does not use section headers.
Consumders need to perform a few random accesses to collect all section information.

```
# start section foo
section_id
size
content
# end section

# start_section bar
section_id
size
content
# end section
```

We could adopt a similar design by adding metadata beside the size field. We would be able to remove the `sh_offset` member.

We could take inspirations from DWARF `.debug_abbrev`: define a few shapes with fixed fields (e.g. `sh_type==SHT_PROGBITS`) and make a section header fill in the rest fields.

We could also adopt a scheme similar to `.subsections_via_symbols`, but using metadata instead of symbols to describe subsection boundaries.

## More ideas

Like other sections, [symbol table](/blog/2024-01-14-exploring-object-file-formats#symbols) and string table sections (`SHT_SYMTAB` and `SHT_STRTAB`) can be compressed through `SHF_COMPRESSED`.
However, compressing the dynamic symbol table (`.dynsym`) and its associated string table (`.dynstr`) is not recommended.

Symbol table sections have a non-zero `sh_entsize`, which remains unchanged after compression.

The string table, which stores symbol names (also section names in LLVM output), is typically much larger than the symbol table itself.
To reduce its size, we can utilize a text compression algorithm.
While compressing the string table, compressing the symbol table along with it might make sense, but using a compact encoding for the symbol table itself won't provide significant benefits.
