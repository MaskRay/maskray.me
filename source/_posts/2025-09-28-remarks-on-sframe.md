---
layout: post
title: Remarks on SFrame
author: MaskRay
tags: [linker,sframe]
---

The `.sframe` format is a lightweight alternative to `.eh_frame` and `.eh_frame_hdr` designed for efficient [stack unwinding](/blog/2020-11-08-stack-unwinding).
By trading some functionality and flexibility for compactness, SFrame achieves significantly smaller size while maintaining the essential unwinding capabilities needed by profilers.

SFrame focuses on three fundamental elements for each function:

- Canonical Frame Address (CFA): The base address for stack frame calculations
- Return address
- Frame pointer

An `.sframe` section follows a straightforward layout:

- Header: Contains metadata and offset information
- Auxiliary header (optional): Reserved for future extensions
- Function Descriptor Entries (FDEs): Array describing each function
- Frame Row Entries (FREs): Arrays of unwinding information per function

<!-- more -->

```cpp
struct [[gnu::packed]] sframe_header {
  struct {
    uint16_t sfp_magic;
    uint8_t sfp_version;
    uint8_t sfp_flags;
  } sfh_preamble;
  uint8_t sfh_abi_arch;
  int8_t sfh_cfa_fixed_fp_offset;
  // Used by x86-64 to define the return address slot relative to CFA
  int8_t sfh_cfa_fixed_ra_offset;
  // Size in bytes of the auxiliary header, allowing extensibility
  uint8_t sfh_auxhdr_len;
  // Numbers of FDEs and FREs
  uint32_t sfh_num_fdes;
  uint32_t sfh_num_fres;
  // Size in bytes of FREs
  uint32_t sfh_fre_len;
  // Offsets in bytes of FDEs and FREs
  uint32_t sfh_fdeoff;
  uint32_t sfh_freoff;
};
```

While magic is popular choices for file formats, they deviate from established ELF conventions, which simplifies utilizes the section type for distinction.

The version field resembles the similar uses within DWARF section headers.
SFrame will likely evolve over time, unlike ELF's more stable control structures.
This means we'll probably need to keep producers and consumers evolving in lockstep, which creates a stronger case for internal versioning.
An internal version field would allow linkers to upgrade or ignore unsupported low-version input pieces, providing more flexibility in handling version mismatches.

## Data structures

### Function Descriptor Entries (FDEs)

Function Descriptor Entries serve as the bridge between functions and their unwinding information.
Each FDE describes a function's location and provides a direct link to its corresponding Frame Row Entries (FREs), which contain the actual unwinding data.

```cpp
struct [[gnu::packed]] sframe_func_desc_entry {
  int32_t sfde_func_start_address;
  uint32_t sfde_func_size;
  uint32_t sfde_func_start_fre_off;
  uint32_t sfde_func_num_fres;
  // bits 0-3 fretype: sfre_start_address type
  // bit 4 fdetype: SFRAME_FDE_TYPE_PCINC or SFRAME_FDE_TYPE_PCMASK
  // bit 5 pauth_key: (AArch64 only) the signing key for the return address
  uint8_t sfde_func_info;
  // The size of the repetitive code block for SFRAME_FDE_TYPE_PCMASK; used by .plt
  uint8_t sfde_func_rep_size;
  uint16_t sfde_func_padding2;
};
```

The current design has room for optimization.
The `sfde_func_num_fres` field uses a full 32 bits, which is wasteful for most functions.
We could use `uint16_t` instead, requiring exceptionally large functions to be split across multiple FDEs.

It's important to note that SFrame's function concept represents code ranges rather than logical program functions.
This distinction becomes particularly relevant with compiler optimizations like hot-cold splitting, where a single logical function may span multiple non-contiguous code ranges, each requiring its own FDE.

The padding field `sfde_func_padding2` represents unnecessary overhead in modern architectures where unaligned memory access performs efficiently, making the alignment benefits negligible.

To enable binary search on `sfde_func_start_address`, FDEs must maintain a fixed size, which precludes the use of variable-length integer encodings like PrefixVarInt.

### Frame Row Entries (FREs)

Frame Row Entries contain the actual unwinding information for specific program counter ranges within a function.
The template design allows for different address sizes based on the function's characteristics.

```cpp
template <class AddrType>
struct [[gnu::packed]] sframe_frame_row_entry {
  // If the fdetype is SFRAME_FDE_TYPE_PCINC, this is an offset relative to sfde_func_start_address
  AddrType sfre_start_address;
  // bit 0 fre_cfa_base_reg_id: define BASE_REG as either FP or SP
  // bits 1-4 fre_offset_count: typically 1 to 3, describing CFA, FP, and RA
  // bits 5-6 fre_offset_size: byte size of offset entries (1, 2, or 4 bytes)
  sframe_fre_info sfre_info;
};
```

Each FRE contains variable-length stack offsets stored as trailing data.
The `fre_offset_size` field determines whether offsets use 1, 2, or 4 bytes (`uint8_t`, `uint16_t`, or `uint32_t`), allowing optimal space usage based on stack frame sizes.

## Architecture-specific stack offsets

SFrame adapts to different processor architectures by varying its offset encoding to match their respective calling conventions and architectural constraints.

### x86-64

The x86-64 implementation takes advantage of the architecture's predictable stack layout:

- First offset: Encodes CFA as `BASE_REG + offset`
- Second offset (if present): Encodes FP as `CFA + offset`
- Return address: Computed implicitly as `CFA + sfh_cfa_fixed_ra_offset` (using the header field)

### AArch64

AArch64's more flexible calling conventions require explicit return address tracking:

- First offset: Encodes CFA as `BASE_REG + offset`
- Second offset: Encodes return address as `CFA + offset`
- Third offset (if present): Encodes FP as `CFA + offset`

The explicit return address encoding accommodates AArch64's variable stack layouts and link register usage patterns.

### s390x

TODO

## `.eh_frame` and `.sframe`

SFrame reduces size compared to `.eh_frame` plus `.eh_frame_hdr` by:

- Eliminating `.eh_frame_hdr` through sorted `sfde_func_start_address` fields
- Replacing CIE pointers with direct FDE-to-FRE references
- Using variable-width `sfre_start_address` fields (1 or 2 bytes) for small functions
- Storing start addresses instead of address ranges. `.eh_frame` address ranges
- Start addresses in a small function use 1 or 2 byte fields, more efficient than `.eh_frame` initial\_location, which needs at least 4 bytes (`DW_EH_PE_sdata4`).
- Hard-coding stack offsets rather than using flexible register specifications

However, the bytecode design of `.eh_frame` can sometimes be more efficient than `.sframe`, as demonstrated on x86-64.

---

SFrame serves as a specialized complement to `.eh_frame` rather than a complement replacement.
The current version does not include personality routines, Language Specific Data Area (LSDA) information, or the ability to encode extra callee-saved registers.
While these constraints make SFrame ideal for profilers and debuggers, they prevent it from supporting C++ exception handling, where libstdc++/libc++abi requires the full `.eh_frame` feature set.

In practice, executables and shared objects will likely contain all three sections:

- `.eh_frame`: Complete unwinding information for exception handling
- `.eh_frame_hdr`: Fast lookup table for `.eh_frame`
- `.sframe`: Compact unwinding information for profilers

The auxiliary header, currently unused, provides a pathway for future enhancements.
It could potentially accommodate `.eh_frame` augmentation data such as personality routines, language-specific data areas (LSDAs), and signal frame handling, bridging some of the current functionality gaps.

## Large text section support

The `sfde_func_start_address` field uses a signed 32-bit offset to reference functions, providing a ±2GB addressing range from the field's location.
This signed encoding offers flexibility in section ordering-`.sframe` can be placed either before or after text sections.

However, this approach faces limitations with large binaries, particularly when LLVM generates `.ltext` sections for x86-64.
The typical section layout creates significant gaps between `.sframe` and `.ltext`:

```
.ltext          // Large text section
.lrodata        // Large read-only data
.rodata         // Regular read-only data
// .eh_frame and .sframe position
.text           // Regular text section
.data
.bss
.ldata          // Large data
.lbss           // Large BSS
```

## Linking and execution views

SFrame employs a unified indexed format across both relocatable files (linking view) and executable files (execution view).
While this design consistency appears elegant, it introduces significant complications in toolchain implementation.

Currently, Binutils enforces a single-element structure within each `.sframe` section, regardless of whether it resides in a relocatable object or final executable.
This approach differs from DWARF sections, which support multiple concatenated elements, each with its own header and body.

This design choice stems from Linux kernel requirements, where kernel modules are relocatable files created with `ld -r`.
The kernel's SFrame support expects each module to contain a single indexed format for efficient runtime processing.
Consequently, GNU ld merges all input `.sframe` sections into a single indexed element, even when producing relocatable files.
This behavior deviates from standard [relocatable linking](/blog/2022-11-21-relocatable-linking) conventions that suppress synthetic section finalization.

The fundamental design issue lies in making linker merging mandatory.
For optimal portability, unwinders should support multiple-element structures within a `.sframe` section.
When a linker builds an index for `.sframe`, it should be viewed as an optimization that relieves the unwinder from constructing its own index at runtime.
This index construction should remain optional rather than required.
While the `SFRAME_F_FDE_SORTED` flag can be cleared to permit unsorted FDEs, current unwinder implementations do not seem to support multiple elements in a single section.

A future version should distinguish between linking and execution views:

- Linking view: Assemblers produce a simpler format, omitting index-specific metadata fields
- Linkers concatenate `.sframe` input sections by default, consistent with DWARF and other metadata sections
- A new `--sframe-index` option enables linkers to synthesize a `.sframe_idx` section containing the indexed format, analogous to [`--gdb-index` and `--debug-names`](/blog/2022-10-30-distribution-of-debug-information).
  The linker builds `.sframe_idx` from input `.sframe` sections.
  To support the Linux kernel workflow (`ld -r` for kernel modules), `ld -r --sframe-index` must also generate the indexed format.
- Linker scripts control placement using: `.sframe_idx : { *(.sframe_idx) }`. From the linker perspective, `.sframe` input sections have been replaced by the linker-synthesized `.sframe_idx`.
  This output section description places the `.sframe_idx` into the `.sframe_idx` output section.

The linking view could omit index-specific metadata fields such as `sfh_num_fdes`, `sfh_num_fres`, `sfh_fdeoff`, and `sfh_freoff`.

The `.debug_pubnames`/`.gdb_index` design provides an excellent model for separate linking and execution views.
While DWARF v5's `.debug_names` unifies both views at the cost of larger linking formats, it represents a reasonable tradeoff since relocatable files contain only a single `.debug_names` section, and debuggers can efficiently load sections with concatenated name tables.

## Section group compliance issues

The current monolithic `.sframe` design creates ELF specification violations when dealing with [COMDAT section groups](/blog/2021-07-25-comdat-and-section-group).
GNU Assembler generates a single `.sframe` section containing relocations to `STB_LOCAL` symbols from multiple text sections, including those in different section groups.

This violates the ELF section group rule, which states:

> A symbol table entry with `STB_LOCAL` binding that is defined relative to one of a group's sections, and that is contained in a symbol table section that is not part of the group, must be discarded if the group members are discarded. References to this symbol table entry from outside the group are not allowed.

The problem manifests when inline functions are deduplicated:

```sh
cat > a.cc <<'eof'
[[gnu::noinline]] inline int inl() { return 0; }
auto *fa = inl;
eof
cat > b.cc <<'eof'
[[gnu::noinline]] inline int inl() { return 0; }
auto *fb = inl;
eof
~/opt/gcc-15/bin/g++ -Wa,--gsframe -c a.cc b.cc
```

Linkers correctly reject this violation:

```
% ld.lld a.o b.o
ld.lld: error: relocation refers to a discarded section: .text._Z3inlv
>>> defined in b.o
>>> referenced by b.cc
>>>               b.o:(.sframe+0x1c)

% gold a.o b.o
b.o(.sframe+0x1c): error: relocation refers to local symbol ".text._Z3inlv" [2], which is defined in a discarded section
  section group signature: "inl()"
  prevailing definition is from a.o
```

(In 2020, I reported a [similar issue](https://gcc.gnu.org/PR93195) for GCC `-fpatchable-function-entry=`.)

Some linkers don't implement this error check. A separate issue arises with garbage collection: by default, an unreferenced `.sframe` section will be discarded.
If the linker implements a workaround to force-retain `.sframe`, it might inadvertently retain all text sections referenced by `.sframe`, even those that would otherwise be garbage collected.

The solution requires restructuring the assembler's output strategy.
Instead of creating a monolithic `.sframe` section, the assembler should generate individual SFrame sections corresponding to each text section.
When a text section belongs to a COMDAT group, its associated SFrame section must join the same group.
For standalone text sections, the `SHF_LINK_ORDER` flag should establish the proper association.

This approach would create multiple SFrame sections within relocatable files, making the size optimization benefits of a simplified linking view format even more compelling.
While this comes with the overhead of additional section headers (where each `Elf64_Shdr` consumes 64 bytes), it's a cost we should pay to be a good ELF citizen.
This reinforces the value of my [section header reduction proposal](/blog/2024-04-01-light-elf-exploring-potential-size-reduction).

## Linker relaxation considerations

Since `.sframe` carries the `SHF_ALLOC` flag, it affects text section addresses and consequently influences [linker relaxation](/blog/2022-07-10-riscv-linker-relaxation-in-lld) on architectures like RISC-V and LoongArch.

If variable-length encoding is introduced to the format, `.sframe` would behave as an address-dependent section similar to `.relr.dyn`.
However, this dependency should not pose significant implementation challenges.

## Linker complexity

## Endianness considerations

The SFrame format currently supports endianness variants, which complicates toolchain implementation.
While runtime consumers typically target a single endianness, development tools must handle both variants to support cross-compilation workflows.

The endianness discussion in [The future of 32-bit support in the kernel](https://lwn.net/Articles/1035727/) reinforces my belief in preferring universal little-endian for new formats.
A universal little-endian approach would reduce implementation complexity by eliminating the need for:

- Endianness-aware function calls like `read32le(config, p)` where `config->endian` specifies the object file's byte order
- Template-based abstractions such as `template <class Endian>` that must wrap every data access function

Instead, toolchain code could use straightforward calls like `read32le(p)`, streamlining both implementation and maintenance.

This approach remains efficient even on big-endian architectures like IBM z/Architecture and POWER.
z/Architecture's LOAD REVERSED instructions, for instance, handle byte swapping with minimal overhead, often requiring no additional instructions beyond normal loads.
While slight performance differences may exist compared to native endian operations, the toolchain simplification benefits generally outweigh these concerns.

```c
#define WIDTH(x) \
typedef __UINT##x##_TYPE__ [[gnu::aligned(1)]] uint##x; \
uint##x load_inc##x(uint##x *p) { return *p+1; } \
uint##x load_bswap_inc##x(uint##x *p) { return __builtin_bswap##x(*p)+1; }; \
uint##x load_eq##x(uint##x *p) { return *p==3; } \
uint##x load_bswap_eq##x(uint##x *p) { return __builtin_bswap##x(*p)==3; }; \

WIDTH(16);
WIDTH(32);
WIDTH(64);
```

However, I understand that my opinion is probably not popular within the object file format community and faces resistance from stakeholders with significant big-endian investments.

## Questioned benefits

SFrame's primary value proposition centers on enabling frame pointer omission while preserving unwinding capabilities.
In scenarios where users already omit leaf frame pointers, SFrame could theoretically allow switching from `-fno-omit-frame-pointer -momit-leaf-frame-pointer` to `-fomit-frame-pointer -momit-leaf-frame-pointer`.
This benefit appears most significant on x86-64, which has limited general-purpose registers (without APX).
Performance analyses show mixed results: some studies claim frame pointers degrade performance by less than 1%, while others suggest 1-2%.
However, this argument overlooks a critical tradeoff—SFrame unwinding itself performs worse than frame pointer unwinding, potentially negating any performance gains from register availability.

Another claimed advantage is SFrame's ability to provide coverage in function prologues and epilogues, where frame-pointer-based unwinding may miss frames.
Yet this overlooks a straightforward alternative: frame pointer unwinding can be enhanced to detect prologue and epilogue patterns by disassembling instructions at the program counter.
No comparative analysis exists between this enhancement approach and SFrame's solution.

SFrame also faces a practical consideration: the `.sframe` section likely requires kernel page-in during unwinding, while the process stack is more likely already resident in physical memory.

Looking ahead, hardware-assisted unwinding through features like x86 Shadow Stack and AArch64 Guarded Control Stack may reshape the entire landscape, potentially reducing the relevance of metadata-based unwinding formats.

## Summary

SFrame represents a pragmatic approach to stack unwinding that achieves significant size reductions by trading flexibility for compactness.
Its design presents several implementation challenges that merit consideration for future versions.

- The unified linking/execution view complicates toolchain implementation without clear benefits
- Section group compliance issues create significant concerns for linker developers
- Limited large text section support restricts deployment in modern binaries
- Uncertainty remains about SFrame's viability as a complete `.eh_frame` replacement

Beyond these implementation concerns, SFrame faces broader ecosystem challenges.
As Ian Rogers noted in [LWN](https://lwn.net/Articles/1030223/), system-wide profiling encounters limitations when system calls haven't transitioned to user code, BPF helpers may return placeholder values, and JIT compilers require additional SFrame support.

The format's future also depends on evolving unwinding strategies. Frame pointer unwinding could potentially be enhanced to detect prologue and epilogue patterns, though comprehensive comparisons with SFrame remain absent from current literature.
