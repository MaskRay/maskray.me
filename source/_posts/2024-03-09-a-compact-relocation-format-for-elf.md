layout: post
title: A compact relocation format for ELF
author: MaskRay
tags: [linker]
---

This article introduces CREL (previously known as RELLEB), a new relocation format offering incredible size reduction ([LLVM implementation in my fork](https://github.com/MaskRay/llvm-project/tree/demo-crel)).

ELF's design emphasizes natural size and alignment guidelines for its control structures. This principle, outlined in Proceedings of the Summer 1990 USENIX Conference, _ELF: An Object File to Mitigate Mischievous Misoneism_, promotes ease of random access for structures like program headers, section headers, and symbols.

> All data structures that the object file format defines follow the "natural" size and alignment guidelines for the relevant class. If necessary, data structures contain explicit padding to ensure 4-byte alignment for 4-byte objects, to force structure sizes to a multiple of four, etc. Data also have suitable alignment from the beginning of the file. Thus, for example, a structure containing an Elf32_Addr member will be aligned on a 4-byte boundary within the file. Other classes would have appropriately scaled definitions. To illustrate, the 64-bit class would define Elf64 Addr as an 8-byte object, aligned on an 8-byte boundary.
> Following the strictest alignment for each object allows the format to work on any machine in a class. That is, all ELF structures on all 32-bit machines have congruent templates. For portability, ELF uses neither bit-fields nor floating-point values, because their representations vary, even among pro- cessors with the same byte order. Of course the programs in an ELF file may use these types, but the format itself does not.

<!-- more -->

While beneficial for many control structures, the natural size guideline presents significant drawbacks for relocations.
Since relocations are typically processed sequentially, they don't gain the same random-access advantages.
The large 24-byte `Elf64_Rela` structure highlights the drawback.
For a detailed comparison of relocation formats, see [Exploring object file formats#Relocations](https://maskray.me/blog/2024-01-14-exploring-object-file-formats#relocations).

Furthermore, `Elf32_Rel` and `Elf32_Rela` sacrifice flexibility to maintain a smaller size, limiting relocation types to a maximum of 255.
This constraint has become noticeable for AArch32 and RISC-V, and especially when platform-specific relocations are needed.
While the 24-bit symbol index field is less elegant, it hasn't posed significant issues in real-world use cases.

In contrast, the [WebAssembly object file format](https://github.com/WebAssembly/tool-conventions/blob/main/Linking.md#relocation-sections) uses LEB128 encoding for relocations and other constrol structures, offering a significant size advantage over ELF.

Inspired by WebAssembly, I will start discussion with a generic compression algorithm and then propose an alternative format (CREL) that addresses ELF's limitations.

## Compressed relocations

While the standard `SHF_COMPRESSED` feature is commonly used for debug sections, its application can easily extend to relocation sections.
I have developed a Clang/lld prototype that demonstrates this by compressing `SHT_RELA` sections.

The compressed `SHT_RELA` section occupies `sizeof(Elf64_Chdr) + size(compressed)` bytes. The implementation retains uncompressed content if compression would result in a larger size.

In scenarios with numerous smaller relocation sections (such as when using `-ffunction-sections -fdata-sections`), the 24-byte `Elf64_Chdr` header can introduce significant overhead.
This observation raises the question of whether encoding `Elf64_Chdr` fields using ULEB128 could further optimize file sizes.
With larger monolithic sections (`.text`, `.data`, `.eh_frame`), compression ratio would be higher as well.

```sh
# configure-llvm is my wrapper of cmake that specifies some useful options.
configure-llvm s2-custom0 -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS='clang;lld'
configure-llvm s2-custom1 -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS='clang;lld' -DCMAKE_{C,CXX}_FLAGS=-Xclang=--compress-relocations=zstd
ninja -C /tmp/out/s2-custom0 lld
ninja -C /tmp/out/s2-custom1 lld

ruby -e 'p Dir.glob("/tmp/out/s2-custom0/**/*.o").sum{|f| File.size(f)}'  # 135996752
ruby -e 'p Dir.glob("/tmp/out/s2-custom1/**/*.o").sum{|f| File.size(f)}'  # 116424688
```

Relocations consume a significant portion (approximately 20.9%) of the file size.
Despite the overhead of `-ffunction-sections -fdata-sections`, the compression technique yields a significant reduction of 14.5%!

However, dropping in-place relocation processing is a downside.

## CREL relocation format

The 1990 ELF paper _ELF: An Object File to Mitigate Mischievous Misoneism_ says "ELF allows extension and redefinition for other control structures."
Let's explore CREL, a new and more compact relocation format designed to replace REL and RELA.
Our emphasis is on simplicity over absolute minimal encoding.
This is achieved by using a byte-oriented encoding that avoids complex compression techniques (e.g., dictionary-based compression, entropy encoder).
As a byte-oriented format, CREL relocations can be further compressed by other codecs, if desired.
Using CREL as relocatable files can decrease memory usage.

See the end of the article for a detailed format description.

A `SHT_CREL` section (preferred name: `.crel<name>`) holds compact relocation entries that decode to `Elf32_Rela` or `Elf64_Rela` depending on the object file class (32-bit or 64-bit).
Its content begins with a ULEB128-encoded relocation count, followed by entries encoding `r_offset`, `r_type`, `r_symidx`, and `r_addend`.
The entries use ULEB128 and SLEB128 exclusively and there is no endianness difference.

Here are key design choices:

*Relocation count (ULEB128)*:

This allows for efficient retrieval of the relocation count without decoding the entire section.
While a `uint32_t` (like [`SHT_HASH`](https://www.sco.com/developers/gabi/latest/ch5.dynamic.html#hash)) could be used, ULEB128 aligns with subsequent entries, removes endianness differences, and offers a slight size advantage in most cases when the number of symbols can be encoded in one to three bytes.

*Shifted offset*:

64-bit data sections frequently have absolute relocations spaced 8 bytes apart.
Additionally, in RISC architectures, offsets are often multiples of 2 or 4.
A shift value of 2 allows delta offsets within the [0, 64) range to be encoded in a single byte, often avoiding the need for two-byte encoding.
In an AArch64 `-O3` build, the shifted offset technique reduces `size(.crel*)` by 12.8%.

Many C++ virtual tables have the first relocation at offset 0x10. In the absence of the shifted offset technique, the relocation at offset 0x10 cannot be encoded in one byte.

```
Relocation section '.crel.data.rel.ro._ZTVN12_GLOBAL__N_113InlineSpillerE' at offset 0x116fe contains 5 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000010  0000007e00000001 R_X86_64_64            0000000000000000 _ZN4llvm7Spiller6anchorEv + 0
0000000000000018  0000000c00000001 R_X86_64_64            0000000000000000 .text._ZN12_GLOBAL__N_113InlineSpillerD2Ev + 0
0000000000000020  0000000f00000001 R_X86_64_64            0000000000000000 .text._ZN12_GLOBAL__N_113InlineSpillerD0Ev + 0
0000000000000028  0000001100000001 R_X86_64_64            0000000000000000 .text._ZN12_GLOBAL__N_113InlineSpiller5spillERN4llvm13LiveRangeEditE + 0
0000000000000030  0000001a00000001 R_X86_64_64            0000000000000000 .text._ZN12_GLOBAL__N_113InlineSpiller16postOptimizationEv + 0
```

The shifted offset also works really well for dynamic relocations, whose offset differences are almost always a multiple of `sizeof(Elf_Addr)`.

*Addend bit*:

CREL was initially designed with explicit addends in each relocation entry.
However, this approach created redundancy when I extended CREL for dynamic relocations (discussed throroughly in another section).
While RELR remains an option, the goal is for CREL to be a viable alternative even without RELR.
In addition, when implicit addends are used, I feel sad if one bit in every relocation entry is wasted.

To address these concerns, a single bit flag has been introduced in the CREL header:

* `addend_bit==1`: The flags include a bit to signal the presence of the delta addend.
* `addend_bit==0`: The flags bit is stolen to encode one more bit for the offset. The addend is implicit and not encoded within the relocation.

This bit flag effectively balances efficiency by avoiding unnecessary storage for dynamic relocations while maintaining flexibility for cases requiring explicit addend values.

Assembler produced `SHT_CREL` sections are supposed to always set the `addend_bit` bit.

*Delta encoding for `r_offset` (ULEB128)*:

Section offsets can be large, and relocations are typically ordered. Storing the difference between consecutive offsets offers compression potential.
In most cases, a single byte will suffice.
While there are exceptions (general dynamic TLS model of s390/s390x uses a local "out-of-order" pair: `R_390_PLT32DBL(offset=o) R_390_TLS_GDCALL(offset=o-2)`), we are optimizing for the common case.

For ELFCLASS32, `r_offsets` members are calculated using modular arithmetic modulo 4294967296.

*Delta encoding for `r_symidx` (SLEB128)*:

This is more for consistency and less for the benefit.

Absolute symbol indexes allow one-byte encoding for symbols in the range [0,128) and offer minor size advantage for static relocations when the symbol table is sorted by usage frequency.
Delta encoding, on the other hand, might optimize for the scenario when the symbol table presents locality: neighbor symbols are frequently mutually called.

Delta symbol index enables one-byte encoding for GOT/PLT dynamic relocations when `.got`/`.got.plt` entries are ordered by symbol index.
For example, `R_*_GLOB_DAT` and `R_*_JUMP_SLOT` relocations can typically be encoded with repeated 0x05 0x01 (when `addend_bit==0 && shift==3`, offset++, symidx++).
Delta encoding has a disvantage. It can partial claim the optimization by arranging symbols in a "cold0 hot cold1" pattern.
In addition, delta symbol index enables one-byte encoding for GOT/PLT dynamic relocations when `.got`/`.got.plt` entries are ordered by symbol index.

In my experiments, absolute encoding with ULEB128 results in slightly larger .o file sizes for both x86-64 and AArch64 builds.

*Delta encoding for `r_type` (SLEB128)*:

Some psABIs utilize relocation types greater than 128.
AArch64's static relocation types begin at 257 and dynamic relocation types begin at 1024, necessitating two bytes with ULEB128/SLEB128 encoding in the absence of delta encoding.
Delta encoding allows all but the first relocation's type to be encoded in a single byte.
An alternative design is to define a base type in the header and encode types relative to the base type, which would introduce slight complexity.

If the AArch32 psABI could be redesigned, allocating `[0,64)` for Thumb relocation types and `[64,*)` for ARM relocation types would optimize delta encoding even further.

While sharing a single type code for multiple relocations would be efficient, it would require reordering relocations.
This conflicts with order requirements imposed by several psABIs and could complicate linker implementations.

*Delta encoding for addend (SLEB128)*:

Encoding the delta addend offers a slight size advantage and optimizes for cases like:

```asm
.quad .data + 0x78
.quad .data + 0x80
.quad .data + 0x88
...
```

*Symbol index/type/addend omission*

Relocations often exhibit patterns that can be exploited for size reduction:

* Symbol index/type remain constant with varying addends, common for `STT_SECTION` symbols, e.g. `.rodata`, `.eh_frame`, `.debug_str_offsets`, `.debug_names`, `.debug_line`, and `.debug_addr`.
* Type/addend remain constant with varying symbol indexes, common for non-section symbols, e.g. function calls, C++ virtual tables, and dynamic relocations.

```c
// Type/addend do not change.
// R_AARCH64_CALL(g0), ...
// R_X86_64_PLT32(g0-4), ...
void f() { g0(); g1(); g2(); }
```

We use the least significant bits of the offset member to signal the presence of symbol index, type, and addend information.
This allows us to omit delta fields when they match the previous entry.

CREL steals 3 bits from the offset member.
I have tried stealing just a bit and utilizing negative symbol index to signal type/addend omission, but offsets generally require fewer bits to encode and stealing bits from offsets is superior.

Omitting the symbol index information is especially beneficial for reducing debug build sizes.
For example, most `.debug_names` relocations can be encoded using only 4 bytes (offset and delta addend), instead of the 7 bytes required otherwise.

While RISC architectures often require multiple relocations with different types to access global data, making type omission slightly less beneficial than x86, the frequent use of call instructions offers large size reduction with type omission.

With a limited number of types and frequent zero addends (except `R_*_RELATIVE` and `R_*_IRELATIVE`), dynamic relocations also benefit from type/addend omission.

---

I have developed a prototype at <https://github.com/MaskRay/llvm-project/tree/demo-crel>.
CREL demonstrates superrior size reduction compared to the `SHF_COMPRESSED SHT_RELA` approach.

### LEB128 among variable-length integer encodings

LEB128 and UTF-8 stand out as the two most commonly used byte-oriented, variable-length integer encoding schemes.
Binary encodings often employ LEB128.
While alternatives like PrefixVarInt (or a suffix-based variant) might excel when encoding larger integers, LEB128 offers advantages when most integers fit within one or two bytes, as it avoids the need for shift operations in the common one-byte representation.

While we could utilize zigzag encoding `(i>>31) ^ (i<<1)` to convert SLEB128-encoded type/addend to use ULEB128 instead, the generate code is inferior to or on par with SLEB128 for one-byte encodings on x86, AArch64, and RISC-V.

```cpp
// One-byte case for SLEB128
int64_t from_signext(uint64_t v) {
  return v < 64 ? v - 128 : v;
}

// One-byte case for ULEB128 with zig-zag encoding
int64_t from_zigzag(uint64_t z) {
  return (z >> 1) ^ -(z & 1);
}
```

While some variale-length integer schemes allocate more integers within the one-byte bucket, I do not believe they would lead to noticeable improvement over LEB128 and their complexity is higher than LEB128.
For example, when I assign one extra bit to offsets (by clearing `addend_bit` in the header), `.crel.dyn` merely decreases by 2.5%.

Here is an extremely simple C decoder implementation for ULEB128 and SLEB128.
The clever use of 64/128 is from Stefan O'Rear.
The return type `uint64_t` can be changed to `size_t` when used in a dynamic loader.
```cpp
static uint64_t read_leb128(unsigned char **buf, uint64_t sleb_uleb) {
  uint64_t acc = 0, shift = 0, byte;
  do {
    byte = *(*buf)++;
    acc |= (byte - 128*(byte >= sleb_uleb)) << shift;
    shift += 7;
  } while (byte >= 128);
  return acc;
}

uint64_t read_uleb128(unsigned char **buf) { return read_leb128(buf, 128); }
int64_t read_sleb128(unsigned char **buf) { return read_leb128(buf, 64); }
```

I have used a modified lld analyze LEB128 length distribution in a x86-64 release build of `lld` that enables CREL.

```cpp
zo[std::min(getULEB128Size(offset-old_offset), 3u) - 1]++;
if (b & 1) {
  auto x = readSLEB128(p); symidx += x; zs[std::min(getSLEB128Size(x), 3u) - 1]++;
}
if (b & 2) {
  auto x = readSLEB128(p); type += x; zt[std::min(getSLEB128Size(x), 3u) - 1]++;
}
if (b & 4) {
  auto x = readSLEB128(p); addend += x; za[std::min(getSLEB128Size(x), 3u) - 1]++;
}
```

The distribution of ULEB128/SLEB128 lengths is:

```
        1       2       3+      any
offset  633056  48846   0       681902
type    187759  0       0       187759
symidx  360230  229610  879     590719
addend  191523  52293   2899    246715
```

80.5% LEB128 encodings are of 1 byte and 19.2% are of 2 bytes.
2-byte delta symidx members are quite common, but I do not plan to steal bits from other members to symidx.

VLQ (variable-length quantity) used by the MIDI file format is a big-endian variant of LEB128. While VLQ offers some advantages:

* Lexicographic ordering: This can be beneficial for database sorting but isn't a factor for object files.
* Potential for better sign extension.

git uses a bijective variant of VLQ. While bijectivity is a nice property, it is not useful.

The potential benefits are outweighed by a slightly more complex encoder and advantages of sticking with popular LEB128 in object files.

<https://sqlite.org/src4/doc/trunk/www/varint.wiki> describes a scheme that encodes 0~240 in one byte, but it is a bit complex.
A simplified version I explored led to larger relocatable files and dynamic relocations compared to LEB128.
Additionally, implementing 67-bit delta offsets and flags without 128-bit integer support would be cumbersome.

```cpp
uint64_t read_vu128(const uint8_t *&p) {
  uint8_t b = *p++;
  if (b < 0xf0) return b;
  b &= 0x0f;
  uint64_t value = 0xf0 + *p++;
  for (unsigned i = 1; i <= b; ++i)
    value += *p++ << i * 8;
  return value;
}
```

The following code implements 70-bit little-endian SuffixVarInt, which covers our 67-bit need.
The single-byte case does more work than ULEB128.

```c
#include <assert.h>
#include <stdbit.h>
#include <stdio.h>
#include <string.h>
#include <inttypes.h>

/*
little-endian suffix varint

xxxxxxx1:      7 value bits, 1 byte
xxxxxx10 ...: 14 value bits, 2 bytes
xxxxx100 ...: 21 value bits, 3 bytes
xxxx1000 ...: 28 value bits, 4 bytes
xxx10000 ...: 35 value bits, 5 bytes
xx100000 ...: 42 value bits, 6 bytes
x1000000 ...: 49 value bits, 7 bytes
10000000 ...: 56 value bits, 8 bytes
00000000 xxxxxxx1 ...: 63 value bits, 9 bytes
00000000 xxxxxx10 ...: 70 value bits, 10 bytes

(We could also take bit 9 and use 00000000 xxxxxxx0 ... (71 value bits))
*/

typedef __uint128_t u128;
typedef uint64_t uu64 [[gnu::aligned(1)]];

// Read variable-length unsigned 70-bit integer
u128 read_suffixvarint(unsigned char **buf) {
  uint64_t x = *(uu64 *)*buf;
  if ((*buf)[0] == 0x00) {
    unsigned char b1 = (*buf)[1];
    if ((b1 & 0x01) == 0x01) {
      // 9 bytes: 00000000 xxxxxxx1 ... (63 value bits)
      *buf += 9;
      return *(uu64 *)(*buf - 8) >> 1;
    }

    // 10 bytes: 00000000 xxxxxx10 ... (70 value bits)
    *buf += 10;
    uint64_t low = *(uu64 *)(*buf - 9) >> 2;
    return low | ((u128)((*buf)[-1]) << 62);
  }

  unsigned n = stdc_trailing_zeros(x) + 1;
  *buf += n;
  return x << (64 - 8 * n) >> (64 - 7 * n);
}

// Write variable-length unsigned 70-bit integer
unsigned write_suffixvarint(unsigned char *buf, u128 x) {
  assert(x < ((u128)1 << 70) && "needs modification to handle wider integers");
  unsigned sig_bits = x ? 128 - stdc_leading_zeros(x) : 1;
  unsigned n = sig_bits >= 64 ? 10 : (sig_bits + 6) / 7;
  if (n > 8) {
    if (n == 9) {
      // 9 bytes: 00000000 xxxxxxx1 ...
      buf[0] = 0x00;
      *(uu64 *)(buf + 1) = (x << 1) | 1;
      return 9;
    }

    // 10 bytes: 00000000 xxxxxx10 ...
    buf[0] = 0x00;
    *(uu64 *)(buf + 1) = (x << 2) | 2;
    buf[9] = x >> 62;
    return 10;
  }

  *(uu64 *)buf = (x << n) | (1ull << (n - 1));
  return n;
}

static void print_u128(u128 x) {
  if (x > UINT64_MAX)
    printf("%4llx%016llx", (unsigned long long)(x >> 64), (unsigned long long)x);
  else
    printf("%20llx", (unsigned long long)x);
}

int main() {
  unsigned char buf[16];
  const u128 tests[] = {
      0,
      1,
      127,
      128,
      255,
      1000,
      16383,
      16384,
      0x7fffffffffffffull,
      0xffffffffffffffull,
      0x7fffffffffffffffull,
      0x8000000000000000ull,
      0xffffffffffffffffull,
      (u128)0xffffffffffffffffull << 1,
      (u128)0xffffffffffffffffull << 6,
  };

  for (int i = 0; i < sizeof(tests)/sizeof(tests[0]); i++) {
    u128 original = tests[i];
    memset(buf, 0, sizeof(buf));
    unsigned bytes = write_suffixvarint(buf, original);
    unsigned char *ptr = buf;
    u128 decoded = read_suffixvarint(&ptr);
    print_u128(original);
    printf(" | Bytes: %2u | Hex: ", bytes);
    for (unsigned j = 0; j < bytes; j++)
      printf("%02x ", buf[j]);
    printf("| Decoded: ");
    print_u128(decoded);
    printf(" | %s\n", (original == decoded) ? "✓" : "✗");
  }
  return 0;
}
```

### Experiments

build         | format | `.o size`   | `size(.rel*)` | .o size<br>decrease
--------------|--------|-------------+---------------|----------
-O3           | RELA   |   136012504 |      28235448 |
-O3           | CREL   |   111583312 |       3806234 | 18.0%
aarch64 -O3   | RELA   |   124965808 |      25855800 |
aarch64 -O3   | CREL   |   102529784 |       3388307 | 18.0%
ppc64le -O3   | RELA   |   129017272 |      26589192 |
ppc64le -O3   | CREL   |   105860576 |       3432419 | 17.9%
riscv64 -O3   | RELA   |   227189744 |      91396344 |
riscv64 -O3   | CREL   |   149343352 |      13549699 | 34.3%
-O1 -g        | RELA   |  1506173760 |     340965576 |
-O1 -g        | CREL   |  1202445768 |      37237274 | 20.2%
-O3 -g $SPLIT | RELA   |   549003848 |     104227128 |
-O3 -g $SPLIT | CREL   |   459768736 |      14992114 | 16.3%

`SPLIT="-gpubnames -gsplit-dwarf"`

Let's compare x86_64 -O3 builds of lld.
`size(.crel*)/size(.rel*) = 3806234 / 28235448`, 13.5%.
The total .o file size has decreased by 18.0%.
In addition, the maximum resident set size of the linker (also lld) using mimalloc has decreased by 4.2%.

It would be interesting to explore the potential gains of combining zstd compression with CREL.

```sh
configure-llvm s2-custom3 -DLLVM_TARGETS_TO_BUILD=host -DLLVM_ENABLE_PROJECTS='clang;lld' -DCMAKE_{C,CXX}_FLAGS='-Wa,--crel -Xclang --compress-relocations=zstd'
ninja -C /tmp/out/s2-custom3 lld

ruby -e 'p Dir.glob("/tmp/out/s2-custom3/**/*.o").sum{|f| File.size(f)}'  # 111383192
```

(Here is a Ruby script to count relocations.

```ruby
agg=Hash.new 0
Dir.glob("**/*.o").each{|f|
  `readelf -Wr #{f}`.lines {|line|
    r = line.scan(/R_([^ ]+)/)
    next if r.empty?
    agg[r[0]] += 1
  }
}
puts agg
```
)

I debated whether to name the new section `SHT_RELOC` (`.reloc<name>`) or `SHT_RELLEB` (`.relleb<name>`).
Ultimately, I chose `SHT_CREL` because its unique name minimizes potential confusion, whereas `SHT_RELOC` could be confused with `SHT_REL` and `SHT_RELA` and the LEB128 part is not that strong.

## Case study

Let's explore some real-world scenarios where relocation size is critical.

### Marker relocations

Marker relocations are utilized to indicate certain linker optimization/relaxation is applicable.
While many marker relocations are used scarcely, RISC-V relocatable files are typically filled up with `R_RISCV_RELAX` relocations.
Their size contribution is quite substantial.

### `.llvm_addrsig`

On many Linux targets, Clang emits a special section called `.llvm_addrsig` (type `SHT_LLVM_ADDRSIG`, LLVM address-significance table) by default to allow `ld.lld --icf=safe`.
The `.llvm_addrsig` section stores symbol indexes in ULEB128 format, independent of relocations. Consequently, tools like `ld -r` and objcopy risk invalidate the section due to symbol table modifications.

Ideally, using relocations would allow certain operations. However, the size concern of REL/RELA in ELF hinders this approach.
In contrast, lld's Mach-O port [chose](https://discourse.llvm.org/t/problems-with-mach-o-address-significance-table-generation/63392) a relocation-based representation for `__DATA,__llvm_addrsig`.

If CREL is adopted, we can consider switching to the relocation representation.

### `.llvm.call-graph-profile`

LLVM leverages a special section called `.llvm.call-graph-profile` (type `SHT_LLVM_CALL_GRAPH_PROFILE`) for both instrumentation- and sample-based profile-guided optimization (PGO).
lld [utilizes this information](https://maskray.me/blog/2020-11-15-explain-gnu-linker-options#call-graph-profile-sort) ((from_symbol, to_symbol, weight) tuples) to optimize section ordering within an input section description, enhancing cache utilization and minimizing TLB thrashing.

Similar to `.llvm_addrsig`, the `.llvm.call-graph-profile` section initially faced the symbol index invalidation problem, which was solved by switching to relocations.
I opted for REL over RELA to reduce code size.

### DWARF sections

In a non-split-DWARF build, `.rela.debug_str_offsets` and `.rela.debug_addr` consume a significant portion of the file size.

DWARF v5 accelerated name-based access with the introduction of the `.debug_names` section.
However, in a `clang -g -gsplit-dwarf -gpubnames` generated relocatable file, the `.rela.debug_names` section can consume a significant portion (approximately 10%) of the file size.

```
Relocation section '.crel.debug_names' at offset 0x65c0 contains 200 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
000000000000002c  0000002f0000000a R_X86_64_32            0000000000000000 .debug_info + 0
00000000000004d8  000000320000000a R_X86_64_32            0000000000000000 .debug_str + 1ab
00000000000004dc  000000320000000a R_X86_64_32            0000000000000000 .debug_str + f61
00000000000004e0  000000320000000a R_X86_64_32            0000000000000000 .debug_str + f7f
...
```

This size increase has sparked discussions within the LLVM community about potentially [altering the file format for linking purposes](https://discourse.llvm.org/t/smaller-object-file-cost-for-debug-names-intended-only-for-linking/76445).

`.debug_line` and `.debug_addr` also contribute a lot of relocations.

```
Relocation section '.crel.debug_addr' at offset 0x64f1 contains 51 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000008  0000004300000001 R_X86_64_64            0000000000000000 _ZN4llvm30VerifyDisableABIBreakingChecksE + 0
0000000000000010  0000002d00000001 R_X86_64_64            0000000000000000 .rodata.str1.1 + 0
0000000000000018  0000002d00000001 R_X86_64_64            0000000000000000 .rodata.str1.1 + b
0000000000000020  0000002d00000001 R_X86_64_64            0000000000000000 .rodata.str1.1 + 13
...

Relocation section '.crel.debug_line' at offset 0x69a5 contains 81 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000022  000000350000000a R_X86_64_32            0000000000000000 .debug_line_str + 0
0000000000000026  000000350000000a R_X86_64_32            0000000000000000 .debug_line_str + 18
000000000000002a  000000350000000a R_X86_64_32            0000000000000000 .debug_line_str + 2c
...
```

Many adjacent relocations share the same section symbol.
CREL can compress most relocations into a few bytes depending on addend sizes (`offset+=O, addend+=A`).

By teaching the assembler to use implicit addends, we achieve even greater size reduction by compressing most relocations into a single byte (mostly 0x04, `offset+=1<<3`).
However, this might harm compression in the presence of `--compress-debug-sections=zstd`.
Personally I recommend that we don't use implicit addends.

### CREL for dynamic relocations

CREL excels with static relocations, but what about the dynamic case?

A substantial part of position-independent executables (PIEs) and dynamic shared objects (DSOs) is occupied by dynamic relocations.
While [RELR](/blog/2021-10-31-relative-relocations-and-relr) (a compact relative relocation format) offers size-saving benefits for relative relocations, other dynamic relocations can benefit from a compact relocation format.
There are a few properties:

* There are much fewer relocation types.
* The offsets are often adjacent by `Elf_Addr`. No two dynamic relocations can share the same offset.
* Each symbol is associated with very few dynamic relocations, typically 1 or 2 (`R_*_JUMP_SLOT` and `R_*_GLOB_DAT`). When a symbol is associated with more dynamic relocations, it is typically a base class function residing in multiple C++ virtual tables, e.g. `__cxa_pure_virtual`. `-fexperimental-relative-c++-abi-vtables` would eliminate such dynamic relocations.

Android's packed relocation format (linker implementation: `ld.lld --pack-dyn-relocs=android`) was an earlier design that applies to all dynamic relocations at the cost of complexity.
It replaces `.rel.dyn`/`.rela.dyn` but does not change the section name.

Additionally, Apple linkers and dyld use LEB128 encoding for bind opcodes.

Here is a one-liner to dump the relative relocation and non-relative relocation sizes for each shared object in a system library directory:
```sh
ruby -e 'Dir.glob("/usr/lib/x86_64-linux-gnu/*.so.*").each{|f| next if File.symlink?(f) || `file #{f}`!~/shared object/; s=`readelf -Wr #{f}`.lines; nr=s.count{|x|x=~/R_/&&x !~/_RELATIVE/}*24; r=s.count{|x|x=~/_RELATIVE/}*24; s=File.size(f); puts "#{f}\t#{s}\t#{r} (#{(r*100.0/s).round(2)}%)\t#{nr} (#{(nr*100.0/s).round(2)}%)" }'
```

I believe CREL compresses dynamic relocations well, but it is far from an optimal dynamic relocation format.
A generalized RELR format would leverage the dyamic relocation [properties](#dynamic-relocations) well. Here's a possible encoding:

```
// R_*_RELATIVE group
Encode(length of the group)
Encode(R_*_RELATIVE)
RELR

// R_*_GLOB_DAT/absolute address relocation group
Encode(length of the group)
Encode(R_*_GLOB_DAT)
Use RELR to encode offsets
Encode symbol indexes separately

// R_*_JUMP_SLOT group
Encode(length of the group)
Encode(R_*_JUMP_SLOT)
Use RELR to encode offsets
Encode symbol indexes separately

...
```

We need to enumerate all dynamic relocation types including `R_*_IRELATIVE`, `R_*_TLSDESC` used by some ports.
Some `R_*_TLSDESC` relocations have a symbol index of zero, but the straightforward encoding does not utilize this property.

Traditionally, we have two dynamic relocation ranges for executables and shared objects (except static position-dependent executables):

* `.rela.dyn` (`[DT_RELA, DT_RELA + DT_RELASZ)`) or `.rel.dyn` (`[DT_REL, DT_REL + DT_RELSZ)`)
* `.rela.plt` (`[DT_JMPREL, DT_JMPREL + DT_PLTRELSZ)`): Stored JUMP_SLOT relocations. `DT_PLTREL` specifies `DT_REL` or `DT_RELA`.

IRELATIVE relocations can be placed in either range, but preferrably in `.rel[a].dyn`.

Some GNU ld ports (e.g. SPARC) treat `.rela.plt` as a subset of `.rela.dyn`, introducing complexity for dynamic loaders.

**CREL adoption considerations**

* New dynamic tag (`DT_CREL`): To identify CREL relocations, separate from existing `DT_REL`/`DT_RELA`.
* No `DT_CRELSZ`: Relocation count can be derived from the CREL header.
* Output section description `.rela.dyn : { *(.rela.dyn) *(.rela.plt) }` is incompatible with CREL.

**Challenges with lazy binding**

glibc's lazy binding scheme relies on [random access to relocation entries within the `DT_JMPREL` table](/blog/2021-09-19-all-about-procedure-linkage-table#:~:text=_dl_fixup).
CREL's sequential nature prevents this. However, eager binding doesn't require random access.
Therefore, when `-z now` (eager binding) is enabled, we can:

* Set `DT_PLTREL` to `DT_CREL`.
* Replace `.rel[a].plt` with `.crel.plt`.

**Challenges with statically linked position-dependent executables**

glibc introduces additional complexity for IRELATIVE relocations in statically linked position-dependent executables.
They should only contain IRELATIVE relocations and no other dynamic relocations.

glibc's `csu/libc-start.c` processes IRELATIVE relocations in the range [`[__rela_iplt_start, __rela_iplt_end)`](/blog/2021-01-18-gnu-indirect-function#non-preemptible-ifunc#rela_iplt_start-and-__rela_iplt_end) (or `[__rel_iplt_start, __rel_iplt_end)`, determined at build time through `ELF_MACHINE_IREL`).
While CREL relocations cannot be decoded in the middle of the section, we can still place IRELATIVE relocations in `.crel.dyn` because there wouldn't be any other relocation types (position-dependent executables don't have RELATIVE relocations).
When CREL is enabled, we can define `__crel_iplt_start` and `__crel_iplt_end` for statically linked position-dependent executables.

If glibc only intends to support `addend_bit==0`, the code can simply be:
```c
  extern const uint8_t __crel_iplt_start[] __attribute__ ((weak));
  extern const uint8_t __crel_iplt_end[] __attribute__ ((weak));
  if (&__crel_iplt_start != &__crel_iplt_end) {
    const uint8_t *p = __crel_iplt_start;
    size_t offset = 0, count = read_uleb128 (&p), shift = count & 3;
    for (count >>= 3; count; count--) {
      uint8_t rel_head = *p++;
      offset += rel_head >> 2;
      if (rel_head & 128)
        offset += (read_uleb128 (&p) << 5) - 32;
      if (rel_head & 2)
        read_sleb128 (&p);
      elf_crel_irel ((ElfW (Addr) *) (offset << shift));
    }
  }
```

**Considering implicit addends for CREL**

Many dynamic relocations have zero addends:

* COPY/GLOB_DAT/JUMP_SLOT relocations only use zero addends.
* Absolute relocations could use non-zero addends with `STT_SECTION` symbol, but linkers convert them to relative relocations.

Usually only RELATIVE/IRELATIVE and potentially TPREL/TPOFF might require non-zero addends.
Switching from `DT_RELA` to `DT_REL` offers a minor size advantage.

I considered defining two separate dynamic tags (`DT_CREL` and `DT_CRELA`) to distinguish between implicit and explicit addends.
However, this would have introduced complexity:

* Should `llvm-readelf -r` dump the zero addends for `DT_CRELA`?
* Should dynamic loaders support both dynamic tags?

I placed the delta addend bit next to offset bits so that it can be reused for offsets.
Thanks to Stefan O'Rear's for making me believe that my original thought of reserving a single bit flag (`addend_bit`) within the CREL header is elegant.
Dynamic loaders prioritizing simplicity can hardcode the desired `addend_bit` value.

`ld.lld -z crel` defaults to implicit addends (`addend_bit==0`), but the option of using in-relocation addends is available with `-z crel -z rela`.

**DT_AARCH64_AUTH_RELR vs CREL**

The AArch64 PAuth ABI introduces `DT_AARCH64_AUTH_RELR` as a variant of RELR for signed relocations.
However, its benefit seems limited.

In a release build of Clang 16, using `-z crel -z rela` resulted in a `.crel.dyn` section size of only 1.0% of the file size.
Notably, enabling implicit addends with `-z crel -z rel` further reduced the size to just 0.3%.
While `DT_AARCH64_AUTH_RELR` will achieve a noticeable smaller relocation size if most relative relocations are encoded with it, the advantage seems less significant considering CREL's already compact size.

Furthermore, `DT_AARCH64_AUTH_RLEL` introduces additional complexity to the linker due to its 32-bit addend limitation: the in-place 64 value encodes a 32-bit schema, giving just 32 bits to the implicit addend.
If the addend does not fit into 32 bits, `DT_AARCH64_AUTH_RELR` cannot be used.
CREL with addends would avoid this complexity.

I have filed [Quantifying the benefits of DT_AARCH64_AUTH_RELR](https://github.com/ARM-software/abi-aa/issues/252).

---

I've implemented `ld.lld -z crel` to replace `.rel[a].dyn` and `.rel[a].plt` with `.crel.dyn` and `.crel.plt`.
Dynamic relocations are sorted by `(r_type, r_offset)` to better utilize CREL.

Let's link clang-16-release using RELA, `-z pack-relative-relocs`, `--pack-dyn-relocs=android+relr`, and `-z pack-relative-relocs -z crel` and analyze the results.

```
% fld.lld @response.txt -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .rela.dyn         RELA            00000000005df318 5df318 c3a980 18   A  3   0  8
  [ 9] .rela.plt         RELA            0000000001219c98 1219c98 001f38 18  AI  3  26  8
% fld.lld @response.txt -z pack-relative-relocs -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .rela.dyn         RELA            00000000005df340 5df340 011088 18   A  3   0  8
  [ 9] .relr.dyn         RELR            00000000005f03c8 5f03c8 0259d0 08   A  0   0  8
  [10] .rela.plt         RELA            0000000000615d98 615d98 001f38 18  AI  3  27  8
% fld.lld @response.txt --pack-dyn-relocs=android+relr -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .rela.dyn         ANDROID_RELA    00000000005df318 5df318 0011fc 01   A  3   0  8
  [ 9] .relr.dyn         RELR            00000000005e0518 5e0518 0259d0 08   A  0   0  8
  [10] .rela.plt         RELA            0000000000605ee8 605ee8 001f38 18  AI  3  27  8
% fld.lld @response.txt -z pack-relative-relocs -z crel -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .crel.dyn         CREL            00000000005df340 5df340 000fbc 00   A  3   0  8
  [ 9] .relr.dyn         RELR            00000000005e0300 5e0300 0259d0 08   A  0   0  8
  [10] .rel.plt          REL             0000000000605cd0 605cd0 0014d0 10  AI  3  27  8
% fld.lld @response.txt -z crel -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .crel.dyn         CREL            00000000005df318 5df318 082c29 00   A  3   0  8
  [ 9] .rel.plt          REL             0000000000661f48 661f48 0014d0 10  AI  3  26  8
% fld.lld @response.txt -z crel -z rela -o - | fllvm-readelf -S - | grep -E ' \.c?rel.?\.'
  [ 8] .crel.dyn         CREL            00000000005df318 5df318 1b8c69 00   A  3   0  8
  [ 9] .rela.plt         RELA            0000000000797f88 797f88 001f38 18  AI  3  26  8
```

Analysis

* Relative relocations usually outnumber non-relative relocations.
* RELR significantly optimizes relative relocations, offering the largest size reduction.
* CREL further improves the non-relative portion, compressing that portion to 5.77%, even better than Android packed relocations (6.60%)! Android's `r_info` sharing with `RELOCATION_GROUPED_BY_INFO_FLAG` has been overshadowed by our shifted offset technique.
* The non-relative relocation advantage is less pronounced since `.relr.dyn` still accounts for a significant portion of the size.
* `.crel.dyn` using `DT_CREL` (implicit addends) without RELR is more than 3x as large as RELR `.relr.dyn+.rela.dyn`.

Decoding ULEB128/SLEB128 would necessitate more work in the dynamic loader.

I have a much patch for the `DT_CREL` support.
In an x86-64 `-O2` build, CREL support linked with `-z pack-relative-relocs -z crel` increases the size of `libc.so` by just 200 bytes (0.0256%) compared to a non-CREL build with `-z pack-relative-relocs`.

* <https://github.com/MaskRay/musl/tree/crel>: musl patch
* <https://github.com/MaskRay/llvm-project/tree/demo-crel>: `ld.lld -z crel`

However, DT_CREL currently uses the non-standard value 0x40000026 (see this comment: 
<https://github.com/llvm/llvm-project/pull/91280#discussion_r1659384454>).
I hope that we can switch to 0x26 before lld gets `-z crel` support.

### Linker notes

`--emit-relocs` and `-r` necessitate combining relocation sections. The output size may differ from the sum of input sections.
The total relocation count must be determined, a new header written, and section content regenerated, as symbol indexes and addends may have changed.
If the linker does not attempt to determine the offset shift in another relocation scan, the offset shift in the header can be set to 0.
Debug sections, `.eh_frame`, and `.gcc_except_table` require special handling to rewrite relocations referencing a dead symbol to `R_*_NONE`.
This also necessitates updating the relocation type.

`--emit-relocs` and `-r` copy CREL relocation sections (e.g. `.crel.text`) to the output.
When `.rela.text` is also present, linkers are required to merge `.rela.text` into `.crel.text`.

GNU ld allows certain unknown section types:

* `[SHT_LOUSER,SHT_HIUSER]` and non-`SHF_ALLOC`
* `[SHT_LOOS,SHT_HIOS]` and non-`SHF_OS_NONCONFORMING`

but reports errors and stops linking for others (unless `--no-warn-mismatch` is specified).
When linking a relocatable file using `SHT_CREL`, you might encounter errors like the following:

```
% clang -Wa,--crel -fuse-ld=bfd a.c b.c
/usr/bin/ld.bfd: unknown architecture of input file `/tmp/a-1e0778.o' is incompatible with i386:x86-64 output
/usr/bin/ld.bfd: unknown architecture of input file `/tmp/b-9963f0.o' is incompatible with i386:x86-64 output
/usr/bin/ld.bfd: error in /tmp/a-1e0778.o(.eh_frame); no .eh_frame_hdr table will be created
/usr/bin/ld.bfd: error in /tmp/b-9963f0.o(.eh_frame); no .eh_frame_hdr table will be created
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Older lld and mold do not report errors. I have filed:

* <https://github.com/llvm/llvm-project/issues/84812> (milestone: 19.1)
* <https://github.com/rui314/mold/issues/1215> (milestone: 2.4.2)

In addition, when there is one `.eh_frame` section with CIE pieces but no relocation, `_bfd_elf_parse_eh_frame` will report an error.

### mips64el

mips64el has an incorrect `r_info`: a 32-bit little-endian symbol index followed by a 32-bit big-endian type.
If mips64el decides to adopt CREL, they can utilize this opportunity to fix `r_info`.

## Data compression

I analyzed relocatable files in lld 18 (x86_64, `-O3` builds) and extracted RELA and CREL relocations.
I investigated the potential benefits of compressing these combined sections within the file.

It's important to consider that data compression, while beneficial for size reduction, can introduce challenges for random access and increase memory usage especially for the linker.
Therefore, it might not be a suitable solution.

```python
with open(f'{f}.bin', 'wb') as out, \
        open(f'{f}.bin.lz4', 'wb') as out_lz4, \
        open(f'{f}.bin.zst', 'wb') as out_zstd, \
        open(f'{f}.bin.xz', 'wb') as out_xz:
    for ofile in Path(where).glob('**/*.o'):
        content = bytes()
        elf = lief.parse(str(ofile))
        for sec in elf.sections:
            if sec.name.startswith(f'.{f}'):
                out.write(sec.content)
                content += sec.content
        sub = subprocess.run(['lz4'], input=content, capture_output=True, check=True)
        out_lz4.write(sub.stdout if len(sub.stdout) < len(content) else content)
        sub = subprocess.run(['zstd'], input=content, capture_output=True, check=True)
        out_zstd.write(sub.stdout if len(sub.stdout) < len(content) else content)
        sub = subprocess.run(['xz'], input=content, capture_output=True, check=True)
        out_xz.write(sub.stdout if len(sub.stdout) < len(content) else content)
```

CREL, being a byte-oriented format, allows for further compression.
It can be regarded as a very efficient filter before a Lempel-Ziv compressor.

Even more surprising, CREL outperforms RELA compressed with lz4 (level 9) and zstd (level 6).

```
% stat -c '%s %n' *.bin*
3808775 crel.bin
2855535 crel.bin.lz4
2184912 crel.bin.xz
2319041 crel.bin.zst
28238016 rela.bin
9679414 rela.bin.lz4
2835180 rela.bin.xz
4281547 rela.bin.zst
```

Dynamic relocations are also worth investigating.
```sh
ld.lld @response.txt -z now --pack-dyn-relocs=relr -o clang.relr
ld.lld @response.txt -z now --pack-dyn-relocs=android+relr -o clang.relr+android
ld.lld @response.txt -z now --pack-dyn-relocs=relr -z crel -o clang.relr+crel
llvm-objcopy --dump-section .rela.dyn=reladyn clang.relr /dev/null
llvm-objcopy --dump-section .crel.dyn=creldyn clang.relr+crel /dev/null
llvm-objcopy --dump-section .rela.dyn=androiddyn clang.relr+android /dev/null
xz -fk reladyn androiddyn creldyn
zstd -fk reladyn androiddyn creldyn
% stat -c '%s %n' reladyn* androiddyn* creldyn*
69768 reladyn
4236 reladyn.xz
4449 reladyn.zst
4604 androiddyn
1484 androiddyn.xz
1481 androiddyn.zst
3980 creldyn
1344 creldyn.xz
1367 creldyn.zst
```

Interestingly, both zstd and LZMA2's default levels on RELA outperform Android's packed relocation format.
Even better, CREL outperforms them both!

The results do not suggest that we add a Lempel-Ziv compressor to CREL. It would significantly increase complexity and decoder memory usage.
The filesystem's transparent compression can handle this for us conveniently.

## CREL proposal for the generic ABI

The latest revision has been proposed at <https://groups.google.com/g/generic-abi/c/ppkaxtLb0P0>.
I have also created:

* [LLVM proposal](https://discourse.llvm.org/t/rfc-crel-a-compact-relocation-format-for-elf/77600)
* [binutils feature request](https://sourceware.org/PR31475)
* [glibc feature request](https://sourceware.org/PR31541)

In <https://www.sco.com/developers/gabi/latest/ch4.sheader.html>, make the following changes.

_In Figure 4-9: Section Types,sh_type, append a row_

`SHT_CREL` | 20

_Add text:_

SHT_CREL - The section holds compact relocation entries with explicit addends. An object file may have multiple relocation sections. See ''Relocation'' below for details.

_In Figure 4-16: Special Sections, append_

`.crelname` | `SHT_CREL` | see below

_Change the text below:_

.relname, .relaname, and .crelname

These sections hold relocation information, as described in ''Relocation''. If the file has a loadable segment that includes relocation, the sections' attributes will include the SHF_ALLOC bit; otherwise, that bit will be off. Conventionally, name is supplied by the section to which the relocations apply. Thus a relocation section for .text normally would have the name .rel.text, .rela.text, or .crel.text.

_In Figure 4-23: Relocation Entries, add:_

```c
typedef struct {
  Elf32_Addr r_offset;
  Elf32_Word r_symidx;
  Elf32_Word r_type;
  Elf32_Sword r_addend;
} Elf32_Crel;

typedef struct {
  Elf64_Addr r_offset;
  Elf64_Word r_symidx;
  Elf64_Word r_type;
  Elf64_Sxword r_addend;
} Elf64_Crel;
```

_Add text above "A relocation section references two other sections":_

A `SHT_CREL` section holds compact relocation entries that decode to `Elf32_Crel` or `Elf64_Crel` depending on the object file class (32-bit or 64-bit).
Its content begins with a ULEB128-encoded value `count * 8 + addend_bit * 4 + shift` (35-bit or 67-bit unsigned), where:

* `count`: Relocation count (32-bit or 64-bit unsigned).
* `addend_bit`: 1 indicates that relocation entries encode addends. 0 indicates implicit addends (stored in the location to be modified).
* `shift`: The shift value (0 to 3) applies to `delta_offset` in relocation entries.

Relocation entries (which encode `r_offset`, `r_symidx`, `r_type`, and `r_addend`) follow the header.
Note: `r_info` in traditional REL/RELA formats has been split into `r_symidx` and `r_type`, allowing `uint32_t` relocation types for ELFCLASS32 as well.

* Delta offset and flags (ULEB128): Holds `delta_offset * (addend_bit ? 8 : 4) + flags` (35-bit or 67-bit unsigned), where:
  + `delta_offset`: Difference in `r_offset` from the previous entry (truncated to `Elf32_Addr` or `Elf64_Addr`), right shifted by `shift`.
  + `flags`: 0 to 7 if `addend_bit` is 1; otherwise 0 to 3.
  + `flags & 1`: Indicate if delta symbol index is present.
  + `flags & 2`: Indicate if delta type is present.
  + `flags & 4`: Indicate if delta addend is present.
* Delta symbol index (SLEB128, if present): The difference in symbol index from the previous entry, truncated to a 32-bit signed integer.
* Delta type (SLEB128, if present): The difference in relocation type from the previous entry, truncated to a 32-bit signed integer.
* Delta addend (SLEB128, if present): The difference in addend from the previous entry, truncated to a 32-bit or 64-bit signed integer depending on the object file class.

ULEB128 or SLEB128 encoded values use the canonical representation (i.e., the shortest byte sequence).
For the first relocation entry, the previous offset, symbol index, type, and addend members are treated as zero.

Encoding/decoding delta offset and flags does not need multi-precision arithmetic. We can just unroll and special case the first iteration.
The header can be encoded/decoded in a similar way. An implementation can assume that the relocation count cannot be larger than 2**61 and simplify the code.

Example C++ encoder:

```cpp
// encodeULEB128(uint64_t, raw_ostream &os);
// encodeSLEB128(int64_t, raw_ostream &os);

const uint8_t addendBit = config->isRela ? 4 : 0, flagBits = config->isRela ? 3 : 2;
Elf_Addr offsetMask = 8, offset = 0, addend = 0;
uint32_t symidx = 0, type = 0;
for (const Elf_Crel &rel : relocs)
  offsetMask |= rel.r_offset;
int shift = std::countr_zero(offsetMask)
encodeULEB128(relocs.size() * 8 + addendBit + shift, os);
for (const Elf_Crel &rel : relocs) {
  Elf_Addr deltaOffset = (rel.r_offset - offset) >> shift;
  uint8_t b = (deltaOffset << flagBits) + (symidx != rel.r_symidx) +
              (type != rel.r_type ? 2 : 0) + (addend != rel.r_addend ? 4 : 0);
  if (deltaOffset < (0x80 >> flagBits)) {
    os << char(b);
  } else {
    os << char(b | 0x80);
    encodeULEB128(deltaOffset >> (7 - flagBits), os);
  }
  if (b & 1) {
    encodeSLEB128(static_cast<int32_t>(rel.r_symidx - symidx), os);
    symidx = rel.r_symidx;
  }
  if (b & 2) {
    encodeSLEB128(static_cast<int32_t>(rel.r_type - type), os);
    type = rel.r_type;
  }
  if (b & 4 & addendBit) {
    encodeSLEB128(std::make_signed_t<Elf_Addr>(rel.r_addend - addend), os);
    addend = rel.r_addend;
  }
}
```

Example C++ decoder:

```cpp
// uint64_t decodeULEB128(uint8_t *&p);
// int64_t decodeSLEB128(uint8_t *&p);

const auto hdr = decodeULEB128(p);
const size_t count = hdr / 8, flagBits = hdr & 4 ? 3 : 2, shift = hdr % 4;
Elf_Addr offset = 0, addend = 0;
uint32_t symidx = 0, type = 0;
for (size_t i = 0; i != count; ++i) {
  const uint8_t b = *p++;
  offset += b >> flagBits;
  if (b >= 0x80)
    offset += (decodeULEB128(p) << (7 - flagBits)) - (0x80 >> flagBits);
  if (b & 1)
    symidx += decodeSLEB128(p);
  if (b & 2)
    type += decodeSLEB128(p);
  if (b & 4 & hdr)
    addend += decodeSLEB128(p);
  rels[i] = {offset << shift, symidx, type, addend};
}
```

Both encoder and decoder can be simplified if the desired `addend_bit` is hardcoded, making `flagBits` an integer literal.

_In Figure 5-10: Dynamic Array Tags, d_tag, add:_

`DT_CREL` | 38 | `d_ptr` | optional | optional

_Add text below:_

* `DT_CREL` - This element is similar to `DT_REL`, except its table uses the CREL format. The relocation count can be inferred from the header.

_Update `DT_PLTREL` and `DT_PLTRELSZ`:_

* `DT_PLTRELSZ`: This element holds the total size, in bytes, of the relocation entries associated with the procedure linkage table. If an entry of type `DT_JMPREL` is present and the `DT_PLTREL` entry value is `DT_REL` or `DT_RELA`, a `DT_PLTRELSZ` must accompany it.
* `DT_PLTREL`: This member specifies the type of relocation entry to which the procedure linkage table refers. The `d_val` member holds `DT_REL`, `DT_RELA`, or `DT_CREL`, as appropriate. All relocations in a procedure linkage table must use the same relocation type.

## Abandonded proposal RELLEB (last revision)

A `SHT_CREL` section holds compact relocation entries that decode to `Elf32_Crel` or `Elf64_Crel` depending on the object file class (32-bit or 64-bit).
Its content begins with a ULEB128-encoded relocation count, followed by entries encoding `r_offset`, `r_symidx`, `r_type`, and `r_addend`.
Note that the `r_info` member in traditional REL/RELA formats has been split into separate `r_symidx` and `r_type` members, allowing `uint32_t` relocation types for ELFCLASS32 as well.

In the following description, `Elf_Addr`/`Elf_SAddr` denote `uint32_t`/`int32_t` for ELFCLASS32 or `uint64_t`/`int64_t` for ELFCLASS64.

* First member (ULEB128): Holds `2 * delta_offset + eq` (33-bit or 65-bit unsigned), where:
  + `delta_offset`: Difference in `r_offset` from the previous entry (`Elf_Addr`).
  + `eq`: Indicates if the symbol index/type match the previous entry (1 for match, 0 otherwise).
* Second Member (SLEB128) if `eq` is 1:
  + Difference in `r_addend` from the previous entry (`Elf_SAddr`).
* Second Member (SLEB128) if `eq` is 0:
  + If type and addend match the previous entry, the encoded value is the symbol index; type and addend are omitted.
  + Otherwise, the bitwise NOT of the encoded value (33-bit signed) is the symbol index; delta type and delta addend follow:
    - Delta type (SLEB128): The difference in relocation type from the previous entry (32-bit signed).
    - Delta addend (SLEB128): The difference in `r_addend` relative to the previous entry (signed `Elf_Addr`).

The bitwise NOT of symbol index 0xffffffff is -0x100000000 (33-bit) instead of 0 (32-bit).

Encoder in pseudo-code:
```c
Elf_Addr offset = 0, addend = 0;
uint32_t symidx = 0, type = 0;
encodeULEB128(relocs.size());
for (const Reloc &rel : relocs) {
  if (symidx == rel.r_symidx && type == rel.r_type) {
    // Symbol index/type match the previous entry. Encode the addend.
    encodeULEB128(2 * uint128_t(rel.r_offset - offset) + 1); // at most 65-bit
    encodeSLEB128(rel.r_addend - addend);
  } else {
    encodeULEB128(2 * uint128_t(rel.r_offset - offset)); // at most 65-bit
    if (type == rel.r_type && addend == rel.r_addend) {
      // Type/addend match the previous entry. Encode the symbol index.
      encodeSLEB128(rel.r_symidx);
    } else {
      // No optimization is applied. Encode symbol index, type, and addend.
      encodeSLEB128(~static_cast<int64_t>(symidx));
      encodeSLEB128(static_cast<int32_t>(rel.type - type));
      type = rel.r_type;
      encodeSLEB128(static_cast<Elf_SAddr>(rel.r_addend - addend));
      addend = rel.r_addend;
    }
    symidx = rel.r_symidx;
  }
}
```

Encoding/decoding a unsigned 65-bit does not need multi-precision arithmetic. We can just unroll and special case the first iteration.
Example C++ encoder:

```cpp
// encodeULEB128(uint64_t, raw_ostream &os);
// encodeSLEB128(int64_t, raw_ostream &os);

Elf_Addr offset = 0, addend = 0;
uint32_t symidx = 0, type = 0;
encodeULEB128(relocs.size(), os);
for (const Reloc &rel : relocs) {
  auto deltaOffset = static_cast<uint64_t>(rel.r_offset - offset);
  offset = rel.r_offset;
  uint8_t odd = outSymidx == symidx && outType == type, b = deltaOffset * 2 + odd;
  if (deltaOffset < 0x40) {
    os << char(b);
  } else {
    os << char(b | 0x80);
    encodeULEB128(deltaOffset >> 6, os);
  }
  symidx = rel.symidx;
  if (!odd && type == rel.type && addend == rel.addend) {
    encodeSLEB128(symidx, os);
  } else {
    if (!odd) {
      encodeSLEB128(~static_cast<int64_t>(symidx), os);
      encodeSLEB128(static_cast<int32_t>(rel.type - type), os);
      type = rel.type;
    }
    encodeSLEB128(std::make_signed_t<uint>(rel.addend - addend), os);
    addend = rel.addend;
  }
}
```

Example C++ decoder:

```cpp
// uint64_t decodeULEB128(uint8_t *&p);
// int64_t decodeSLEB128(uint8_t *&p);

size_t count = decodeULEB128(p);
Elf_Addr offset = 0, addend = 0;
uint32_t symidx = 0, type = 0;
for (size_t i = 0; i != count; ++i) {
  const uint8_t b = *p++;
  offset += b >> 1;
  if (b >= 0x80)
    offset += (decodeULEB128(p) << 6) - 0x40;
  int64_t x = decodeSLEB128(p);
  if (b & 1) {
    addend += x;
  } else {
    if (x < 0) {
      x = ~x;
      type += decodeSLEB128(p);
      addend += decodeSLEB128(p);
    }
    symidx = x;
  }
  rels[i] = {offset, symidx, type, addend};
}
```
