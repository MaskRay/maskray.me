---
layout: post
title: Remarks on SFrame
author: MaskRay
tags: [linker,sframe]
---

SFrame is a new [stack walking format](/blog/2020-11-08-stack-unwinding) for userspace profiling, inspired by Linux's in-kernel [ORC unwind format](https://docs.kernel.org/arch/x86/orc-unwinder.html).
While SFrame eliminates some `.eh_frame` CIE/FDE overhead, it sacrifices functionality (e.g., personality, LSDA, callee-saved registers) and flexibility, and its stack offsets are less compact than `.eh_frame`'s bytecode-style CFI instructions.
In llvm-project executables I've tested on x86-64, `.sframe` section is 20% larger than `.eh_frame`.
It also remains significantly larger than highly compact schemes like [Windows ARM64 unwind codes](https://www.corsix.org/content/windows-arm64-unwind-codes).

SFrame describes three elements for each function:

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

FP and return address may not be saved at the same time.
In leaf functions GCC might save the return address and FP to floating-point registers.

- First offset: Encodes CFA as `BASE_REG + offset`
- Second offset (if preset): Encodes the return address as one of
  + stack slot: `CFA + offset2, if (offset2 & 1 == 0)`
  + register number: `offset2 >> 1, if (offset2 & 1 == 1)`
  + not saved: `if (offset2 == SFRAME_FRE_RA_OFFSET_INVALID)`
- Third offset (if present)
  + FP stack slot = CFA + offset3, if (offset3 & 1 == 0)
  + FP register number = offset3 >> 1, if (offset3 & 1 == 1)

The format uses 0 (an invalid SFrame RA offset from CFA value) to indicate that the return address is not saved, while FP is saved.

## Toolchain implementation

In the GNU toolchain, the assembler in GNU Binutils reinterprets CFI directives and generates the `.sframe` section, while GCC itself has no knowledge of SFrame.

Some scenarios that cannot be described by `.eh_frame` in the absence of the frame pointer are equally inexpressible in SFrame.
Additionally, SFrame has extra limitations, as certain CFI directives cannot be re-encoded into the SFrame format.
You can take a look at `as_warn` code in binutils-gdb `gas/gen-sframe.c` to learn some cases.

On the other hand, the assembler approach allows SFrame to work with hand-written assembly files with CFI directives.

## ORC and `.sframe`

TODO

## `.eh_frame` and `.sframe`

SFrame reduces header size compared to `.eh_frame` plus `.eh_frame_hdr` by:

- Eliminating `.eh_frame_hdr` through sorted `sfde_func_start_address` fields
- Replacing CIE pointers with direct FDE-to-FRE references
- Using variable-width `sfre_start_address` fields (1 or 2 bytes) for small functions
- Storing start addresses instead of address ranges. `.eh_frame` address ranges
- Start addresses in a small function use 1 or 2 byte fields, more efficient than `.eh_frame` initial\_location, which needs at least 4 bytes (`DW_EH_PE_sdata4`).
- Hard-coding stack offsets rather than using flexible register specifications

However, the bytecode design of `.eh_frame` can sometimes be more efficient than `.sframe`, as demonstrated on x86-64.

---

SFrame serves as a specialized complement to `.eh_frame` rather than a complete replacement.
The current version does not include personality routines, Language Specific Data Area (LSDA) information, or the ability to encode extra callee-saved registers.
While these constraints make SFrame ideal for profilers, they prevent it from supporting C++ exception handling, where libstdc++/libc++abi requires the full `.eh_frame` feature set.

In practice, executables and shared objects will likely contain all three sections:

- `.eh_frame`: Complete unwinding information for exception handling
- `.eh_frame_hdr` (encompassed by the `PT_GNU_EH_FRAME` program header): Fast lookup table for `.eh_frame`
- `.sframe` (encompassed by the `PT_GNU_SFRAME` program header)

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

## Object file format design issues

### Mandatory index building problems

Currently, Binutils enforces a single-element structure within each `.sframe` section, regardless of whether it resides in a relocatable object or final executable.
While the `SFRAME_F_FDE_SORTED` flag can be cleared to permit unsorted FDEs, proposed unwinder implementations for the Linux kernel do not seem to support multiple elements in a single section.
The design choice makes linker merging mandatory rather than optional.

This design choice stems from Linux kernel requirements, where kernel modules are relocatable files created with `ld -r`.
The pending SFrame support for linux-perf expects each module to contain a single indexed format for efficient runtime processing.
Consequently, GNU ld merges all input `.sframe` sections into a single indexed element, even when producing relocatable files.
This behavior deviates from standard [relocatable linking](/blog/2022-11-21-relocatable-linking) conventions that suppress synthetic section finalization.

This approach differs from almost every metadata section, which support multiple concatenated elements, each with its own header and body.
LLVM supports numerous well-behaved metadata sections (`__asan_globals`, `.stack_sizes`, `__patchable_function_entries`, `__llvm_prf_cnts`, `__sancov_bools`, `__llvm_covmap`, `__llvm_gcov_ctr_section`, `.llvmcmd`, and `llvm_offload_entries`) that concatenate without issues.
SFrame stands apart as the only metadata section demanding version-specific merging as default linker behavior, creating unprecedented maintenance burden.
For optimal portability, unwinders should support multiple-element structures within a `.sframe` section.

For optimal portability, we must support object files from diverse origins—not just those built from a single toolchain.
In environments where almost everything is built from source with a single toolchain offering strong SFrame support, forcing default-on index building may be acceptable.
However, we must also accommodate environments with prebuilt object files using older SFrame versions, or toolchains that don't support old formats.
I believe unwinders should support multiple-element structures within a `.sframe` section.
When a linker builds an index for `.sframe`, it should be viewed as an optimization that relieves the unwinder from constructing its own index at runtime.
This index construction should remain optional rather than required.

### Section group compliance and garbage collection issues

GNU Assembler generates a single `.sframe` section containing relocations to `STB_LOCAL` symbols from multiple text sections, including those in different section groups.

This creates ELF specification violations when a referenced text section is discarded by the [COMDAT section group rule](/blog/2021-07-25-comdat-and-section-group).
The ELF specification states:

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

### Version compatibility challenges

The current design creates significant version compatibility problems.
When a linker only supports v3 but encounters object files with v2 `.sframe` sections, it faces impossible choices:

- Discard v2 sections: Silently losing functionality
- Report errors: Breaking builds with mixed-version object files
- Concatenate sections: Currently unsupported by unwinders
- Upgrade v2 to v3: Requires maintaining version-specific merge logic for every version

This differs fundamentally from reading a format—each version needs version-specific *merging* logic in every linker.
Consider the scenario where v2 uses layout A, v3 uses layout B, and v4 uses layout C.
A linker receiving objects with all three versions must produce coherent output with proper indexing while maintaining version-specific merge logic for each.

Real-world mixing scenarios include:

- Third-party vendor libraries built with older toolchains
- Users linking against prebuilt libraries from different sources
- Users who don't need SFrame but must handle prebuilt libraries with older versions
- Users updating their linker to a newer version that drops legacy SFrame support

Most users will not need stack tracing features—this may change eventually, but that will take many years.
In the meantime, they must accept unneeded information while handling the resulting compatibility issues.

Requiring version-specific merging as default behavior would create maintenance burden unmatched by any other loadable metadata section.

### Proposed format separation

A future version should distinguish between linking and execution views to resolve the compatibility and maintenance challenges outlined above.
This separation has precedent in existing debug formats: `.debug_pubnames`/`.gdb_index` provides an excellent model for separate linking and execution views.
DWARF v5's `.debug_names` takes a different approach, unifying both views at the cost of larger linking formats—a reasonable tradeoff since relocatable files contain only a single `.debug_names` section, and debuggers can efficiently load sections with concatenated name tables.

For SFrame, the separation would work as follows:

**Separate linking format.** Assemblers produce a simpler format, omitting index-specific metadata fields such as `sfh_num_fdes`, `sfh_num_fres`, `sfh_fdeoff`, and `sfh_freoff`.

**Default concatenation behavior.** Linkers concatenate `.sframe` input sections by default, consistent with DWARF and other metadata sections.
Linkers can handle mixed-version scenarios gracefully without requiring version-specific merge logic, eliminating the impossible maintenance burden of keeping version-specific merge logic for every SFrame version in every linker implementation.
Distributions can roll out SFrame support incrementally without requiring all linkers to support index building immediately.

The unwinder implementation cost is manageable.
Stack unwinders already need to support `.sframe` sections across the main executable and all shared objects.
Supporting multiple concatenated elements within a single `.sframe` section presents no fundamental technical barrier—this is a one-time implementation cost that provides forward and backward compatibility.

**Optional index construction.** When the opt-in option `--sframe-index` is requested, the linker builds an index from recognized versions while reporting warnings for unrecognized ones.
This is analogous to [`--gdb-index` and `--debug-names`](/blog/2022-10-30-distribution-of-debug-information).

With this approach, the linker builds `.sframe_idx` from input `.sframe` sections.
To support the Linux kernel workflow (`ld -r` for kernel modules), `ld -r --sframe-index` must also generate the indexed format.

The index construction happens before section matching in linke scripts.
The output section description `.sframe_idx : { *(.sframe_idx) }` places the synthesized `.sframe_idx` into the `.sframe_idx` output section.
`.sframe` input sections have been replaced by the linker-synthesized `.sframe_idx`, so we don't write `*(.sframe)`.

## Alternative: Deriving SFrame from .eh_frame

An alternative approach could eliminate the need for assemblers to generate `.sframe` sections directly.
Instead, the linker would merge and optimize `.eh_frame` as usual (which requires CIE and FDE boundary information), then derive `.sframe` (or `.sframe_idx`) from the optimized `.eh_frame`.

This approach offers a significant advantage: since the linker only reads the stable `.eh_frame` format and produces `.sframe` or `.sframe_idx` as output, version compatibility concerns disappear entirely.

While CFI instruction decoding introduces additional complexity (a step previously unneeded), this is balanced by the architectural advantage of centralizing the conversion logic.
Rather than scattering format-specific processing code throughout the linker (similar to how `SHF_MERGE` and `.eh_frame` require special internal representations), the transformation logic remains localized.

The counterargument centers on maintenance burden.
This fine-grained knowledge of the SFrame format may expose the linker to more frequent updates as the format evolves—a serious risk, given that the linker's foundational role in the build process demands exceptional stability and robustness.

### Post-processing alternative

A more cautious intermediate strategy could leverage existing Linux distribution post-processing tools, modifying them to append `.sframe` sections to executable and shared object files after linking completes.
While this introduces more friction than native linker support and requires integration into package build systems, it offers several compelling advantages:

- Allows `.sframe` format experimentation without imposing linker complexity
- Provides time for the format to mature and prove its value before committing to linker integration
- Enables testing across diverse userspace packages in real-world scenarios
- Post-link tools can optimize and even overwrite sections in-place without linker constraints
- For cases where optimization significantly shrinks the section, `.sframe` can be placed at the end of the file (similar to BOLT moving `.rodata`)

However, this approach faces practical challenges.
Post-processing adds build complexity, particularly with features like build-ids and read-only file systems.
The success of `.gdb_index`, where linker support (`--gdb-index`) proved more popular than post-link tools, suggests that native linker support eventually becomes necessary for widespread adoption.

The key question is timing: should linker integration be the starting point or the outcome of proven stability?

## SHF_ALLOC considerations

The `.sframe` section carries the `SHF_ALLOC` flag, meaning it's loaded as part of the program's read-only data segment.
This design choice creates tradeoffs:

**With SHF_ALLOC:**
- `.sframe` contributes to initial read-only data segment consumption
- Can be accessed directly as part of the memory-mapped area, relying on kernel's page fault on demand mechanism.

**Without SHF_ALLOC:**
- No upfront memory cost
- Tracers must open the file and initiate IO to mmap the section on demand
- Runtime cost may not amortize well for frequent tracing

Analysis of 337 files in /usr/bin and /usr/lib/x86_64-linux-gnu/ shows `.eh_frame` typically consumes 5.2% (median: 5.1%) of file size:

```
EH_Frame size distribution:
  Min: 0.3%    Max: 11.5%    Mean: 5.2%    Median: 5.1%

  0%-1%: 7 files      5%-6%: 62 files
  1%-2%: 17 files     6%-7%: 33 files
  2%-3%: 37 files     7%-8%: 36 files
  3%-4%: 49 files     8%-9%: 20 files
  4%-5%: 50 files     9%-10%: 20 files
                      10%-12%: 6 files
```

If `.sframe` size is comparable to `.eh_frame`, this represents significant overhead for applications that never use stack tracing—likely the majority of users.
Most users will not need stack trace features, raising the question of whether having `.sframe` always loaded is an acceptable overhead for distributions shipping it by default.

perf supports `.debug_frame` (tools/perf/util/unwind-libunwind-local.c), which does not have `SHF_ALLOC`.
While there's a difference between status quo and what's optimal, the non-`SHF_ALLOC` approach deserves consideration for scenarios where runtime tracing overhead can be amortized or where memory footprint matters more than immediate access.

## Kernel challenges

The `.sframe` section may not be resident in the physical memory. SFrame proposers are attempting to defer user stack traces until syscall boundaries.

Ian Rogers points out that BPF programs can no longer simply stack trace user code. This change breaks stack trace deduplication, a commonly used BPF primitive.

## Miscellaneous minor considerations

**Linker relaxation considerations:**

Since `.sframe` carries the `SHF_ALLOC` flag, it affects text section addresses and consequently influences [linker relaxation](/blog/2022-07-10-riscv-linker-relaxation-in-lld) on architectures like RISC-V and LoongArch.

If variable-length encoding is introduced to the format, `.sframe` would behave as an address-dependent section similar to `.relr.dyn`.
However, this dependency should not pose significant implementation challenges.

**Endianness considerations:**

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

SFrame's primary benefit centers on enabling frame pointer omission while preserving unwinding capabilities.
In scenarios where users already omit leaf frame pointers, SFrame could theoretically allow switching from `-fno-omit-frame-pointer -momit-leaf-frame-pointer` to `-fomit-frame-pointer -momit-leaf-frame-pointer`.
This benefit appears most significant on x86-64, which has limited general-purpose registers (without APX).
Performance analyses show mixed results: some studies claim frame pointers degrade performance by less than 1%, while others suggest 1-2%.
However, this argument overlooks a critical tradeoff—SFrame unwinding itself performs worse than frame pointer unwinding, potentially negating any performance gains from register availability.

Another claimed advantage is SFrame's ability to provide coverage in function prologues and epilogues, where frame-pointer-based unwinding may miss frames.
Yet this overlooks a straightforward alternative: frame pointer unwinding can be enhanced to detect prologue and epilogue patterns by disassembling instructions at the program counter.

SFrame also faces a practical consideration: the `.sframe` section likely requires kernel page-in during unwinding, while the process stack is more likely already resident in physical memory.
As Ian Rogers noted in [LWN](https://lwn.net/Articles/1030223/), system-wide profiling encounters limitations when system calls haven't transitioned to user code, BPF helpers may return placeholder values, and JIT compilers require additional SFrame support.

Looking ahead, hardware-assisted unwinding through features like x86 Shadow Stack and AArch64 Guarded Control Stack may reshape the entire landscape, potentially reducing the relevance of metadata-based unwinding formats.
Meanwhile, compact unwinding schemes like [Windows ARM64](https://www.corsix.org/content/windows-arm64-unwind-codes) demonstrate that significantly smaller metadata formats remain viable alternatives to both SFrame and `.eh_frame`.
Proposals like Asynchronous Compact Unwind Descriptors have demonstrated that compact unwind formats can work with shrink-wrapping optimizations.
There is a feature request for a compact information for AArch64 <https://github.com/ARM-software/abi-aa/issues/344>

## Summary

Beyond these fundamental questions about SFrame's value proposition, the format presents a size improvement to Linux kernel's ORC unwinder.
Its design presents several implementation challenges that merit consideration for future versions:

- Object file format design issues (mandatory index building, section group compliance, version compatibility)
- Limited large text section support restricts deployment in modern binaries
- Size issue

These technical concerns, combined with the fundamental value questions raised above, suggest that careful consideration is warranted before widespread adoption.

## If we proceed, here is how to do it right

According to [this comment on llvm-project #64449](https://github.com/llvm/llvm-project/issues/64449#issuecomment-3433777733), "v3 is the version that will be submitted upstream when the time is right."
Please share feedback on the format before it's finalized, even if you may not be impressed with the design.

To ensure rapid SFrame evolution without compatibility concerns, a better approach is to build a library that parses `.eh_frame` and generates SFrame.
The Linux kernel can then use this library (in objtool?) to generate SFrame for vmlinux and modules.
Relying on assembler/linker output for this critical metadata format requires a level of stability that is currently concerning.

The ongoing maintenance implications warrant particular attention.
Observing the binutils mailing list reveals a significant volume of SFrame commits.
Most linker features stabilize quickly after initial implementation, but SFrame appears to require continued evolution.
Given the linker's foundational role in the build process, which demands exceptional stability and robustness, the long-term maintenance burden deserves careful consideration.

Early integration into GNU toolchain has provided valuable feedback for format evolution, but this comes at the cost of coupling the format's maturity to linker stability.
The SFrame GNU toolchain developers exhibit a [concerning tendency to disregard ELF and linker conventions](https://sourceware.org/pipermail/binutils/2025-October/144974.html)—a serious problem for all linker maintainers.

### Learning from existing compact unwind implementations

LLVM has had a battle-tested compact unwind format in production use since 2009 with OS X 10.6. The efficiency gains are dramatic even if it might only cover synchronous unwinding needs.
OpenVMS's x86-64 port, which is ELF-based, also adopted this format as documented in their "VSI OpenVMS Calling Standard" and their [2018 post on LLVM Discourse](https://discourse.llvm.org/t/rfc-asynchronous-unwind-tables-attribute/59282).
This isn't to suggest we should simply adopt the existing compact unwind format wholesale.
The x86-64 design dates back to 2009 or earlier, and there are likely improvements we can make.
However, we should aim for similar or better efficiency gains.

On AArch64, there are at least two formats the ELF one can learn from: LLVM's compact unwind format (aarch64) and Windows ARM64 Frame Unwind Code.
