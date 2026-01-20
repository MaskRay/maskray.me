---
layout: post
title: A compact section header table for ELF
author: MaskRay
tags: [elf,linker]
---

Updated in 2025-11.

## Background: The ELF Section Header Table

In ELF files, the section header table is an array of 64-byte `Elf64_Shdr` structures (or 40-byte `Elf32_Shdr` for 32-bit).
Each structure describes name, type, flags, address, offset, size, link, info, alignment, and entry size.

```c
typedef struct {
  uint32_t sh_name;      // Name (string table index)
  uint32_t sh_type;      // Type
  uint64_t sh_flags;     // Flags
  uint64_t sh_addr;      // Virtual address
  uint64_t sh_offset;    // File offset
  uint64_t sh_size;      // Size in bytes
  uint32_t sh_link;      // Link to another section
  uint32_t sh_info;      // Extra information, possibly a section header table index
  uint64_t sh_addralign; // Alignment
  uint64_t sh_entsize;   // Entry size if section holds fixed-size entries
} Elf64_Shdr;            // Total: 64 bytes
```

The key inefficiency is that most fields are frequently zero or can use default values (e.g., `sh_type=SHT_PROGBITS` for most code/data sections), yet each header consumes a fixed 64 bytes.
With `-ffunction-sections` creating hundreds or thousands of sections per compilation unit, this overhead accumulates rapidly.

When building llvm-project with `-O3 -ffunction-sections -fdata-sections -Wa,--crel,--allow-experimental-crel`, the section header tables occupy 17.6% of the total `.o` file size.
In a `-g -Wa,--crel,--allow-experimental-crel` build, the section header tables occupy 13.9% of the total `.o` file size.

This overhead multiplies when the compiler creates a metadata section for every code section (e.g., for sanitizer, code coverage, or stack unwinding metadata).
```
.group
.text.f0
.meta.f0

.group
.text.f1
.meta.f1
```

## Solution overview

Building on my previous work on [compact relocations (CREL)](/blog/2024-03-09-a-compact-relocation-format-for-elf), I propose an alternative section header table format that achieves significant space savings through two key techniques:

1. **Variable-length integer encoding**: Instead of fixed-width fields, values are encoded using 1-9 bytes depending on magnitude
2. **Presence flags**: A bitmap indicates which fields differ from default values, allowing omission of zeros and common defaults

The format is backward compatible: existing tools continue using the traditional format (`e_shentsize != 0`), while updated tools can opt into the compact format (`e_shentsize == 0`).

The remainder of this article presents a formal specification suitable for inclusion in the ELF specification, followed by design rationale, evaluation, and discussion of alternatives.

## Proposal for the ELF specification

- [generic-abi proposal](https://groups.google.com/g/generic-abi/c/9DPPniRXFa8)
- [LLVM proposal](https://discourse.llvm.org/t/compact-section-header-table-for-elf/88821)

### Specification Modifications

In _Chapter 2, ELF Header_, section _Contents of the ELF Header_, modify the description of `e_shentsize`:

<blockquote>
<strong>e_shentsize</strong>

This member holds a section header's size in bytes, or the value zero to indicate a compact section header table format.
When non-zero, a section header is one entry in the section header table; all entries have the same size.
When zero, the section header table uses the compact encoding format described below.
</blockquote>

In _Chapter 3, Sections_, add a new subsection _3.3. Compact Section Header Table_ after _3.2. Section Header Table Entry_ and shift subsequent sections.

<strong>3.3. Compact Section Header Table</strong>

When `e_shentsize` equals zero, the section header table uses a compact encoding format.

<strong>Variable-Length Integer Encoding</strong>

The compact format employs a variable-length integer (unsigned CLEB128) encoding scheme that encodes 64-bit unsigned integers using 1 to 9 bytes.

This encoding is a variant of Little Endian Base 128 where the length information is determined by counting trailing zero bits in the first byte.
Specifically, if the first byte has *n-1* trailing zeros, then the encoded integer occupies *n* bytes total.
The special case of a zero first byte signals a 9-byte encoding.

The format supports the following encodings, where 'x' represents value bits:

```
xxxxxxx1: 7 value bits, 1 byte 
xxxxxx10 xxxxxxxx: 14 value bits, 2 bytes
xxxxx100 xxxxxxxx xxxxxxxx: 21 value bits, 3 bytes
xxxx1000 xxxxxxxx xxxxxxxx xxxxxxxx: 28 value bits, 4 bytes
xxx10000 xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx: 35 value bits, 5 bytes
xx100000 xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx: 42 value bits, 6 bytes
x1000000 xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx: 49 value bits, 7 bytes
10000000 xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx: 56 value bits, 8 bytes

00000000 xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx: 64 value bits, 9 bytes
# The last byte should not be 0.
```

(The 9-byte encoding is special and may encode values represented by shorter encodings. With the last byte restriction the encoding is [bijective](https://en.wikipedia.org/wiki/Bijective_numeration).)

The remaining bits in the first byte, plus all subsequent bytes, contain the actual value in little-endian order.

**Example:** Consider encoding the value 147 (decimal).
In binary, 147 = 10010011, which requires 8 significant bits.
Since 7 bits fit in one byte but 8 bits require two bytes, we use the 2-byte format (xxxxxx10 xxxxxxxx) which provides 14 value bits.

- Byte 0: 6 low-order bits of the value, shifted left by 2, with the length tag `10` in the two least significant bits
- Byte 1: Remaining high-order bits of the value

Calculation: Byte 0 = ((147 & 0x3f) << 2) | 0x02 = 0x4e; Byte 1 = (147 >> 6) = 0x02.
The encoded representation is `0x4e 0x02`.

**Example 2:** Consider encoding the value 0xfedcba9876543210, which requires 64 significant bits, exceeding the 56 bits available in the 8-byte format.
Therefore, we must use the 9-byte format (00000000 xxxxxxxx...).

- Byte 0: 0x00 (the tag indicating 9-byte format)
- Bytes 1-8: The 64-bit value in little-endian byte order

The encoded representation is `0x00 0x10 0x32 0x54 0x76 0x98 0xba 0xdc 0xfe`.

<strong>Compact Section Header Table Format</strong>

The compact section header table, located at file offset `e_shoff`, begins with a VarInt-encoded section count, immediately followed by that many compact section headers.

Each compact section header begins with a single-byte `presence` field indicating which `Elf_Shdr` members are explicitly encoded.
Fields are stored in the following order:

* `sh_name`: unsigned CLEB128 encoded (always present)
* `sh_offset`: unsigned CLEB128 encoded (always present)
* `sh_type`: unsigned CLEB128 encoded if `presence & 0x01`; otherwise defaults to `SHT_PROGBITS`
* `sh_flags`: unsigned CLEB128 encoded if `presence & 0x02`; otherwise defaults to 0
* `sh_addr`: unsigned CLEB128 encoded if `presence & 0x04`; otherwise defaults to 0
* `sh_size`: unsigned CLEB128 encoded if `presence & 0x08`; otherwise defaults to 0
* `sh_link`: unsigned CLEB128 encoded if `presence & 0x10`; otherwise defaults to 0
* `sh_info`: unsigned CLEB128 encoded if `presence & 0x20`; otherwise defaults to 0
* `sh_addralign`: uint8_t encoded as a log2 value if `presence & 0x40`; otherwise defaults to 1
* `sh_entsize`: unsigned CLEB128 encoded if `presence & 0x80`; otherwise defaults to 0

The `sh_addralign` field is encoded as the base-2 logarithm of the alignment value.
A default value of 1 (representing 2⁰) indicates no alignment constraint.
In the traditional format, `sh_addralign` may be 0 or a positive integral power of two, where both 0 and 1 mean no alignment constraint.
The compact encoding does not distinguish between these cases, treating all unspecified alignments as 1, which preserves the intended semantics.

### Reference implementation

The following pseudocode illustrates the decoding process for a section header:

```cpp
// getVarInt(const uint8_t *&p);

const uint8_t *sht = base + ehdr->e_shoff;
const uint8_t *p = sht + offsets[i];
uint8_t presence = *p++;
Elf_Shdr shdr = {};
shdr.sh_name = getVarInt(p);
shdr.sh_offset = getVarInt(p);
shdr.sh_type = presence & 0x01 ? getVarInt(p) : ELF::SHT_PROGBITS;
shdr.sh_flags = presence & 0x02 ? getVarInt(p) : 0;
shdr.sh_addr = presence & 0x04 ? getVarInt(p) : 0;
shdr.sh_size = presence & 0x08 ? getVarInt(p) : 0;
shdr.sh_link = presence & 0x10 ? getVarInt(p) : 0;
shdr.sh_info = presence & 0x20 ? getVarInt(p) : 0;
shdr.sh_addralign = presence & 0x40 ? (uint64_t)1 << *p++ : 1;
shdr.sh_entsize = presence & 0x80 ? getVarInt(p) : 0;
```

The following C code provides a reference implementation for encoding and decoding VarInt values:

```c
typedef uint64_t uu64 [[gnu::aligned(1)]];

// Normalize to little-endian, i.e. byte swap on big-endian.
static inline uint64_t norm_le64(uint64_t val) {
#if __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
  return __builtin_bswap64(val);
#else
  return val;
#endif
}

// Write variable-length unsigned 64-bit integer.
unsigned put_cleb128(unsigned char *buf, uint64_t x) {
  // Fast path for n == 1
  if (x < 128) {
    buf[0] = (x << 1) | 1;
    return 1;
  }

  unsigned sig_bits = 64 - stdc_leading_zeros(x);
  unsigned n = (sig_bits + 6) / 7;
  if (n > 8) {
    // 9 bytes: 00000000 xxxxxxxx ...
    buf[0] = 0x00;
    *(uu64 *)(buf + 1) = norm_le64(x);
    return 9;
  }

  uint64_t tagged = norm_le64((x << n) | ((uint64_t)1 << (n - 1)));
  memcpy(buf, &tagged, n);
  return n;
}

// Read variable-length unsigned 64-bit integer.
uint64_t get_cleb128(unsigned char **buf) {
  // Fast path for n == 1
  uint8_t b0 = (*buf)[0];
  if (b0 & 1) {
    *buf += 1;
    return b0 >> 1;
  }

  if (b0 == 0x00) {
    // 9 bytes: 00000000 xxxxxxxx ...
    *buf += 9;
    return norm_le64(*(uu64 *)(*buf - 8));
  }

  unsigned n = stdc_trailing_zeros(b0) + 1;
  uint64_t x = 0;
  memcpy(&x, *buf, n);
  *buf += n;
  x = (norm_le64(x) >> n) & (((uint64_t)1 << (7 * n)) - 1);
  return x;
}

// The implementation is suitable when reading up to 6 bytes after the last byte is ok.
uint64_t get_cleb128_unsafe(unsigned char **buf) {
  // Fast path for n == 1
  uint8_t b0 = (*buf)[0];
  if (b0 & 1) {
    *buf += 1;
    return b0 >> 1;
  }

  if (b0 == 0x00) {
    // 9 bytes: 00000000 xxxxxxxx ...
    *buf += 9;
    return norm_le64(*(uu64 *)(*buf - 8));
  }

  uint64_t x = norm_le64(*(uu64 *)*buf);
  unsigned n = stdc_trailing_zeros(b0) + 1;
  *buf += n;
  return x << (64 - 8 * n) >> (64 - 7 * n);
}
```

## Why this VarInt encoding?

The unsigned prefix-based variable-length integer encoding is identical to the one in [MLIR bytecode](https://mlir.llvm.org/docs/BytecodeFormat/#signed-variable-width-integers), rather than the more common Little Endian Base 128 (LEB128) encoding used in DWARF and WebAssembly.

**Advantages over LEB128:**

* The encoded length is determined from the first byte alone, eliminating the need to scan subsequent bytes for continuation bits.
  This allows an efficient implementation that dispatches on all available lengths.
* The maximum length is 9 rather than 10.

**Trade-offs:**

* The common one-byte path require a shift operation (to extract 7 bits from an 8-bit byte), whereas LEB128 can use the byte directly. However, this minor overhead is offset by significantly faster decoding for multi-byte values, which are common in section header tables.
* LEB128 is self-synchronizing, a property we don't need.

This format can be viewed as a little-endian variant of Chromium's PrefixVarint, which places the length tag in the most significant bits.
While PrefixVarint offers advantages for big-endian systems (single-byte efficiency matching LEB128), the little-endian approach adopted here is better suited for modern architectures.
PrefixVarint requires `stdc_leading_ones` for branch-free decoding of multi-byte values (2≤n≤8), which is less efficient than `stdc_trailing_zeros`, and necessitates byte-swapping on little-endian architectures.
<https://gist.github.com/MaskRay/80b705d903688870bac96da64e7e243b> provides implementations for both variants.

This proposal does not introduce variable-length signed integers. If we ever introduce them, we should use sign extension instead of zig-zag encoding.

## Performance characteristics

The variable-length encoding means O(1) random access to arbitrary section headers is not directly supported—accessing the *i*-th header requires decoding all preceding headers.
However, this limitation is often addressed by applications building their own in-memory data structures after an initial scan.
Alternatively, for simpler applications, a prescan can be performed to determine the starting offset of each header beforehand.

## Alternatives considered

Several alternative approaches were evaluated during the design of this format:

### WebAssembly-style inline metadata

The WebAssembly object file format implemented by LLVM embeds section metadata directly with each section, eliminating the section header table entirely:

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

**Pros**: Could eliminate the `sh_offset` field entirely, as sections are discovered sequentially.

**Cons**: Requires scanning the entire file to build a section index, making access to offsets expensive. This is particularly problematic for linkers that need to quickly locate specific sections. Paging in the section header table is more efficient for a parallel linking strategy.

### DWARF-style Abbreviation Tables

Inspired by DWARF's `.debug_abbrev`, this approach defines a small set of section "shapes" with predefined field layouts (e.g., a shape for typical `SHT_PROGBITS` sections). Each section header references a shape and fills in only the varying fields.

While this could achieve better compression for files with many similar sections, it adds complexity through an additional indirection layer and abbreviation table.

```
struct Csht_Template {
  uint8_t presence;       // presence of sh_addr/sh_size/sh_link/sh_info/sh_addralign
  varint sh_type;         // SHT_PROGBITS, SHT_NOBITS, etc.
  varint sh_flags;        // Compressed common flags
  varint sh_addralign_log2; // log2(alignment)
  varint sh_entsize;      // Usually 0
};

struct Csht_Entry {
  uint8_t template_id; // Which template
  
  varint sh_name;
  varint sh_offset;
  varint sh_size;
  
  // Optional sh_addr/sh_size/sh_link/sh_info/sh_addralign
};
```

I've written a Ruby program to estimate the size when enabling this optimization.

```ruby
  abbrev_table_size = cleb128_size(templates.size)
  templates.keys.each do |sig|
    sh_type, sh_flags, sh_addralign, sh_entsize = sig

    abbrev_table_size += 1 # presence of sh_addr/sh_size/sh_link/sh_info/sh_addralign
    abbrev_table_size += cleb128_size(sh_type)
    abbrev_table_size += cleb128_size(sh_flags)
    abbrev_table_size += 1 if sh_addralign > 1 # sh_addralign
    abbrev_table_size += cleb128_size(sh_entsize)
  end

  # Calculate total section entries size
  total_entries_size = 0
  section_headers.each do |shdr|
    # 1 byte for template_id
    entry_size = 1

    # Always present: sh_name and sh_offset
    entry_size += cleb128_size(shdr[:sh_name]) + cleb128_size(shdr[:sh_offset])

    # Variable fields not in template: sh_addr, sh_size, sh_link, sh_info
    entry_size += cleb128_size(shdr[:sh_addr]) if shdr[:sh_addr] != 0
    entry_size += cleb128_size(shdr[:sh_size]) if shdr[:sh_size] != 0
    entry_size += cleb128_size(shdr[:sh_link]) if shdr[:sh_link] != 0
    entry_size += cleb128_size(shdr[:sh_info]) if shdr[:sh_info] != 0

    total_entries_size += entry_size
  end

  abbrev_table_size + total_entries_size
```

When applied to a `-fno-unique-section-names --start-no-unused-arguments -Wa,--crel,--allow-experimental-crel --end-no-unused-arguments` build of `llvm-mc` and `opt`, the section header table is 71.6% as large as the one without this optimization.
However, this modest improvement doesn't justify the added complexity.

```
% ruby ~/Dev/object-file-size-analyzer/shdr_abbrev.rb /tmp/out/s2-custom-crel
Files: 1811
Total uncompressed: 20767104 bytes (17.9%)
Total compressed (zstd): 2024585 bytes (2.1%)
Total compressed (xz): 1719648 bytes (1.8%)
Total (compressed + 24): 2068049 bytes (2.1%)
Total (compressed + 64 + 24): 2183953 bytes (2.2%)
Total cshdr: 4282269 bytes (4.3%)
Total cshdr-abbrev: 3066326 bytes (3.1%)
Total file size: 115876616 bytes
```

### Mach-O `.subsections_via_symbols`

To work around the limitation of section count, Mach-O uses symbols as markers in the section data stream to describe subsection boundaries.

**Cons**:
Mach-O's subsection feature imposes restrictions on label differences.
Coalescing symbols or adjusting values in sections would change subsection boundaries, leading to complexity in binary manipulation tools.
Additionally, there is a loss of flexibility as subsection properties (type, flags, linked-to section) cannot be changed at least.

## Comparison with general compression

An alternative approach might be to apply general-purpose compression (e.g., zstd) to the section header table.
However, this introduces several practical challenges that make the compact format more suitable for this use case.

**Unavoidable decompression overhead:** Unlike `SHF_COMPRESSED` sections, where utilities can skip decompression if they don't need the section content, compressed section headers force every tool to decompress the entire table just to read basic file metadata.
While individual sections can often be ignored, the section header table itself is fundamental to file navigation.
This upfront cost is particularly problematic for lightweight binary utilities that would prefer to avoid dependencies on compression libraries like zstd.

**Runtime use case challenges:** While section header table bloat primarily affects relocatable files, the compact format should remain usable in executables and shared objects.
Programs may access their own executable (via `argv[0]`, `/proc/self/exe`, or by parsing `/proc/self/maps`) for self-profiling or enhanced crash reporting—use cases where `.symtab` provides richer symbol information than `.dynsym`.
For these runtime scenarios, integrating a general compression library introduces significant complexity and potential security concerns, whereas implementing the compact format is straightforward and self-contained.

**Memory overhead:** Many utilities maintain their own in-memory representation of the section header table.
With general compression, they would need to allocate a separate decompression buffer in addition to their internal data structures, effectively doubling memory usage.
The compact format, by contrast, can be decoded directly into the application's preferred representation without intermediate buffers.

## Evaluation

To validate the design and measure its effectiveness in practice, I implemented a working prototype in Clang and lld.
The implementation is available at <https://github.com/MaskRay/llvm-project/tree/demo-cshdr>.

An earlier implementation using LEB128 is available at <https://github.com/MaskRay/llvm-project/tree/demo-cshdr-2024>.

The following table shows measurements from building llvm-project with different options

`.o size`  |sht size   | build
-----------+-----------+-----------------------------------------------------------------------------------------
 142435560 |   20767296 | -O3 -ffunction-sections -fdata-sections
 117643000 |   20767296 | -O3 -ffunction-sections -fdata-sections -Wa,--crel,--allow-experimental-crel
 101229095 |    4351215 | -O3 -ffunction-sections -fdata-sections -Wa,--crel,--allow-experimental-crel,--cshdr
1595953680 |   22474624 | -O3 -ffunction-sections -fdata-sections -g
1278305368 |   22474624 | -O3 -ffunction-sections -fdata-sections -g -Wa,--crel,--allow-experimental-crel
1260602248 |    4766392 | -O3 -ffunction-sections -fdata-sections -g -Wa,--crel,--allow-experimental-crel,--cshdr
2627892024 |  294078080 | -g
2112127000 |  294078080 | -g -Wa,--crel,--allow-experimental-crel
1888962427 |   70911315 | -g -Wa,--crel,--allow-experimental-crel,--cshdr

<!--
for i in s2-custom-{none,crel,cshdr}; do ruby -e 'tot=totsht=0; Dir.glob("/tmp/out/'$i'/**/*.o").each{|f| s=File.size(f); sht=%x{/tmp/Rel/bin/llvm-readelf --elf-output-style=JSON -h #{f}|jq ".[]|.ElfHeader.SectionHeaderOffset"}.to_i; tot+=s; totsht+=s-sht }; printf "%10d | %10d | %s\n", tot, totsht, "'$i'"'; done
-->

## Future Work

The compact section header table is one component of a broader effort to reduce ELF object file overhead. Several complementary improvements are worth exploring:

### String Table Compression

The string table (`.strtab`), which stores section and symbol names, is typically much larger than the section header table itself.
Like other sections, [symbol table](/blog/2024-01-14-exploring-object-file-formats#symbols) and string table sections (`SHT_SYMTAB` and `SHT_STRTAB`) can be compressed through `SHF_COMPRESSED`.
Standard compression algorithms (zlib, zstd) can achieve significant savings here.

However, compressing the dynamic symbol table (`.dynsym`) and its associated string table (`.dynstr`) is not recommended, as this would impact runtime loading performance.

### Symbol Table Encoding

While symbol tables have a fixed entry size (`sh_entsize`), applying a compact encoding similar to section headers might yield modest savings.
However, since symbol table size is typically dominated by the string table, this is a lower priority optimization.
