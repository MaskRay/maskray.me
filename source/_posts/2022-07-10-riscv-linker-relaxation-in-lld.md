---
layout: post
title: RISC-V linker relaxation in lld
author: MaskRay
tags: [binutils,llvm,linker]
---

On 2022-07-07, I added a RISC-V linker relaxation framework in ld.lld and implemented `R_RISCV_ALIGN/R_RISCV_CALL/R_RISCV_CALL_PLT` relaxation.
The changes will be included in the next llvm-project release 15.0.0.
This post describes the implementation.

See [The dark side of RISC-V linker relaxation](/blog/2021-03-14-the-dark-side-of-riscv-linker-relaxation) for more information about RISC-V linker relaxation.

<!-- more -->

## Problems

ld.lld performs these steps (simplified):

* Parse command line options
* Find and scan input files (.o, .so, .a), interleaved with symbol resolution
* Call LLVM LTO to get ELF object files
* Global transforms (section based garbage collection, identical code folding, etc)
* Create synthetic sections
* Map input sections and synthetic (linker-generated) sections into output sections
* Scan relocations
* Finalize synthetic sections
* Layout (addresses, thunks, `SHT_RELR`, symbol assignments)
* Assign file offsets
* Write header and sections

We need to find a place to insert the relaxation pass.

### Relocation scanning

Relocation scanning makes dynamic relocation decisions and determines the sizes of `.got`, `.got.plt`, `.plt`, `.rela.dyn`, and `.relr.dyn` sections.
Their address and size changes will affect subsequent sections and sections using certain linker script features.
The one-pass relocation scanning scheme is tied to the whole ld.lld design and is difficult to change.
Relocation scanning takes time and we want to perform it only when necessary.

Linker relaxation may make input sections smaller and nullify the current section layout.
For a call code sequence, if the size decrease makes the destination closer enough to the relocated location, we need to rewrite the code sequence into a shorter form.
This change may have a cascading effect and trigger further relaxation.
E.g. in the following diagram consisting of three input sections, if `call x` (a pseudo instruction which expands to 8 bytes) in section B is shortened, B's size will decrease and `call c` in A may become a new candidate for relaxation.

```text
A[... call c; ...] -- B[... call x; ...] -- C[c: ...]
```

### Symbol values

The changed section layout may change symbol values. It is rare but an output section address can use a symbol value.
In the following linker script example, the size change of `.mid` will change the address of `.high`.

```text
SECTIONS {
  .mid 0x10800 : { mid_start = .; *(.mid); mid_end = .; }
  .high 0x110000+(mid_end-mid_start) : { *(.high) }
  .high2 0x210000+SIZEOF(.mid) : { *(.high2) }
}
```

## Design

Linker relaxation has to an iterative process. Since it interacts with address-dependent sections and symbol assignments, the main idea is to add linker relaxation to the layout phase. So we get:

* Scan relocations
* Finalize synthetic sections
* Layout (_relaxation,_ addresses, thunks, `SHT_RELR`, symbol assignments)
* Assign file offsets

"Scan relocations" is kept. We add another relocation scanning pass to process all relaxable relocations. This pass computes multiple results.

* for each relocated location, the replacement relocation type, the rewritten code sequence, and the number of bytes to delete
* the size of each input code section
* `st_value` and `st_size` for each symbol defined relative to the section

The results are used by `script->assignAddresses()` to compute the next layout: section addresses and symbol values.
We repeat the process until the results converge.

In some uncommon cases an input section may expand in a later iteration.
If we choose to shrink sections at the end of one iteration, the expansion will be difficult to handle.
My idea is that the section size shrink and code sequence rewrites need to be postponed after the iteration fixed point is reached.

```cpp
template <class ELFT> void Writer<ELFT>::finalizeAddressDependentContent() {
  ...
  uint32_t pass = 0;
  for (;;) {
    Create thunks or call relaxOnce;
    ++pass;

    Report "not converged" if pass is too large;

    Update address-dependent sections;
    Assign addresses to sections and symbols;
  }
  if (!config->relocatable && config->emachine == EM_RISCV)
    riscvFinalizeRelax(pass);
  ...
}
```

Two function calls are added to `finalizeAddressDependentContent`: `relaxOnce` and `riscvFinalizeRelax`.
The RISC-V port implements `relaxOnce` which calls `relax` on all input code sections.

```cpp
bool RISCV::relaxOnce(int pass) const {
  ...
  bool changed = false;
  for (OutputSection *osec : outputSections) {
    if (!(osec->flags & SHF_EXECINSTR))
      continue;
    for (InputSection *sec : getInputSections(*osec, storage))
      changed |= relax(*sec);
  }
  return changed;
}
```

```cpp
static bool relax(InputSection &sec) {
  Restore original st_value for symbols relative to this section.

  std::fill_n(aux.relocTypes.get(), sec.relocations.size(), R_RISCV_NONE);
  aux.writes.clear();
  for (auto [i, r] : llvm::enumerate(sec.relocations)) {
    const uint64_t loc = secAddr + r.offset - delta;
    uint32_t &cur = aux.relocDeltas[i], remove = 0;
    switch (r.type) {
    case R_RISCV_ALIGN: {
      remove = the number of bytes to delete;
      break;
    }
    case R_RISCV_CALL:
    case R_RISCV_CALL_PLT:
      if (i + 1 != sec.relocations.size() &&
          sec.relocations[i + 1].type == R_RISCV_RELAX)
        relaxCall(sec, i, loc, r, remove);
      break;
    // Other relaxable relocation types
    }

    Update symbol st_value/st_size according to symbol anchors;

    delta += remove;
    if (delta != cur) {
      cur = delta;
      changed = true;
    }
  }

  Update trailing symbol anchors;

  sec.bytesDropped = delta;
  return changed;
}
```

`relax` iterates over non-resolved relocations for this input section and sets `remove` to the number of bytes to delete.
`delta` is the accumulated number of bytes to delete. It is stored in `aux.relocDeltas[i]` for processing in `riscvFinalizeRelax`.

### Symbol anchors

Updating `st_value` and `st_size` for each symbol defined relative to the section uses a neat technique.

```asm
  ...
a:
  .balign 16  # R_RISCV_ALIGN(r_addend=12)
b:
```

In this example, there is an `R_RISCV_ALIGN` relocation at the `.balign` location. Its offset equals of symbol `a`'s `st_value`.
If some bytes preceding `a` are deleted, `a`'s `st_value` needs to be decreased by that number of bytes.
`b` has a larger `st_value` and its `st_value` needs to additionally take into account the `R_RISCV_ALIGN` relaxation.

To compute all `st_value` of symbols relative to the current input section, we maintain two sorted lists: (a) relaxable relocations (b) `st_value`.
For each symbol, find the relocation with the largest `r_offset` which is smaller than the symbol's `st_value`, then decrease `st_value` by `r_offset`.
The interleave of `st_value` values and `r_offset` values is like the merge function of merge sort.

`st_size` can be computed similarly. Instead we interleave `st_value+st_size` values with `r_offset` values.
After the final `st_value+st_size` is determined, decrease the sum by the final `st_value` to compute the final `st_size`.
In the implementation, I just place all initial `st_value` and `st_value+st_size` values in one sorted list. Both are indicated by a `SymbolAnchor` object.

```cpp
struct SymbolAnchor {
  uint64_t offset;
  Defined *d;
  bool end; // true for the anchor of st_value+st_size
};

    if (remove) {
      for (; sa.size() && sa[0].offset <= r.offset; sa = sa.slice(1)) {
        if (sa[0].end)
          sa[0].d->size = sa[0].offset - delta - sa[0].d->value;
        else
          sa[0].d->value -= delta;
      }
    }
```

Since we use the decrement amount (`sa[0].d->value -= delta;`), when starting the next iteration, we need to restore the original `st_value`.

### Finalize relaxation

```cpp
void elf::riscvFinalizeRelax(int passes) {
  ...
  for (OutputSection *osec : outputSections) {
    if (!(osec->flags & SHF_EXECINSTR))
      continue;
    for (InputSection *sec : getInputSections(*osec, storage)) {
      RISCVRelaxAux &aux = *sec->relaxAux;
      if (!aux.relocDeltas)
        continue;

      Allocate space for the new section content to `p`;
      sec->rawData = makeArrayRef(p, newSize);

      // Update section content: remove NOPs for R_RISCV_ALIGN and rewrite
      // instructions for relaxed relocations.
      for (size_t i = 0, e = rels.size(); i != e; ++i) {
        uint32_t remove = aux.relocDeltas[i] - delta;
        delta = aux.relocDeltas[i];
        if (remove == 0)
          continue;

        // Copy from last location to the current relocated location.
        const Relocation &r = rels[i];
        uint64_t size = r.offset - offset;
        memcpy(p, old.data() + offset, size);
        p += size;
```

```cpp
        // For R_RISCV_ALIGN, we will place `offset` in a location (among NOPs)
        // to satisfy the alignment requirement. If `remove` is a multiple of 4,
        // it is as if we have skipped some NOPs. Otherwise we are in the middle
        // of a 4-byte NOP, and we need to rewrite the NOP sequence.
        int64_t skip = 0;
        if (r.type == R_RISCV_ALIGN) {
          if (remove % 4 != 0) {
            skip = r.addend - remove;
            Rewrite `skip` bytes with nop and an optional trailing c.nop;
          }
        } else if (RelType newType = aux.relocTypes[i]) {
          Rewrite code sequence;
        }

        p += skip;
        offset = r.offset + skip + remove;
      }
      memcpy(p, old.data() + offset, old.size() - offset);

      Subtract the previous relocDeltas value from the relocation offset.
      For a pair of R_RISCV_CALL/R_RISCV_RELAX with the same offset, decrease
      their r_offset by the same delta.
    }
  }
}
```

For each input code section, We iterate over its non-resolved relocations.
For an `R_RISCV_ALIGN` associated with some bytes to delete, we copy all content from the previous location to `r_offset`, then skip some bytes for the next copy.
```asm
  ...         # Copy all content from the previous location to r_offset
  .balign 8  # R_RISCV_ALIGN(r_addend=6)
  # A prefix of the NOPs may be skipped for the next memcpy
  addi a0, a0, 1
```

Say we need to delete 2 bytes. If we use `[]` to indicate the copied bytes, the current and the next copy patterns will look like:
```text
old: ...]  NOP  NOP [NOP  NOP  NOP  NOP  ADDI ADDI ADDI ADDI ...]
old:                 next copy

new: ...] [NOP  NOP  NOP  NOP  ADDI ADDI ADDI ADDI ...]
new:
```

Let's check a call relaxation case. The `call` pseudo instruction expands to a pair of auipc and jalr.
```asm
  call dest@plt  # R_RISCV_CALL_PLT, R_RISCV_RELAX
```

If auipc+jalr can be relaxed to a 4-byte jal, we ignore auipc, replace jalr with jal, and increment `p` and `offset` so that next memcpy will start copying from the first byte after jalr.
The rewritten instruction starts at the first byte indicated by `skip=4`.
```text
old: ...] AUIPC AUIPC AUIPC AUIPC JALR JALR JALR JALR [.........]
          remove=4                skip=4               next copy

new: ...] JAL   JAL   JAL   JAL  [.........]
```

Here is a demonstration for a `tail` pseudo instruction which is relaxed to `c.j`.
```asm
  tail dest@plt  # R_RISCV_CALL_PLT, R_RISCV_RELAX
```
```text
old: ...] AUIPC AUIPC AUIPC AUIPC JALR JALR JALR JALR [.........]
          remove=6                          skip=2     next copy

new: ...] C.J   C.J  [.........]
```

## Relaxable code sequences

### Alignment relaxation

With 3 values we can compute the address of the relocated location: `secAddr + r.offset - delta`.
`delta` is the asscumulated number of bytes to delete. It is subtracted from the original `r_offset` value.

The alignment is `PowerOf2Ceil(r.addend + 2)`. The expected location after alignment is `(loc + align - 1) & -align` and therefore `loc + r.addend  - ((loc + align - 1) & -align)` is the number of bytes to delete.

### Call relaxation

These following pseudo instructions are available to call subroutines.
```asm
call a@plt          # @plt can be omitted. In ld.lld R_RISCV_CALL/R_RISCV_CALL_PLT are indistinguishable
# auipc ra, 0       # R_RISCV_CALL_PLT(a), R_RISCV_RELAX
# jalr ra, 0(ra)

tail a@plt
# auipc t1, 0       # R_RISCV_CALL_PLT(a), R_RISCV_RELAX
# jalr zero, 0(t1)

jump a, t0
# auipc t0, 0       # R_RISCV_CALL(a), R_RISCV_RELAX
# jalr zero, 0(t0)
```

Each expands to a pair of `auipc` and `jalr`.

* `call`: ra is both a scratch register and the destination register
* `tail`: t1 is a scratch register. x0 is the destination register
* `jump`: the scratch register is specified. x0 is the destination register

The two instructions can be relaxed to one alternative instruction. There are 3 choices:

* `c.j`: RVC, the destination register is x0, and the displacement is representable as an int12
* `c.jal`: RV32C, the destination register is ra, and the displacement is representable as an int12
* `jal`: the displacement is representable as an int21

The first two need to delete 6 bytes and rewrite 2 bytes while the third needs to delete 4 bytes and rewrite 4 bytes.

### Local-exec TLS relaxation

See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage) for more information about TLS.

Computing the address or storing a value into a TLS variable takes 3 instructions.
If `st_value(x) < 2048` (i.e. `hi20(x) == 0`), one instruction suffices.

```text
lui rd, %tprel_hi(x)           # R_RISCV_TPREL_HI20, R_RISCV_RELAX
add rd, rd, tp, %tprel_add(x)  # R_RISCV_TPREL_ADD, R_RISCV_RELAX
addi rd, rd, %tprel_lo(x)      # R_RISCV_TPREL_LO12_I, R_RISCV_RELAX

=>

addi rd, tp, st_value(x)
```

```text
lui rd, %tprel_hi(x)           # R_RISCV_TPREL_HI20, R_RISCV_RELAX
add rd, rd, tp, %tprel_add(x)  # R_RISCV_TPREL_ADD, R_RISCV_RELAX
sw rs, st_value(x)(rd)         # R_RISCV_TPREL_LO12_S, R_RISCV_RELAX

=>

sw rs, st_value(x)(rd)
```

Pending patch: <https://reviews.llvm.org/D129425>

### lui relaxation

If ld.lld implements this, most absolute and PC-relative relocations need bookkeeping that they can candidates for relaxation.
This may add quite a bit of overhead.

### Relaxation against the Global Pointer

See "Relaxing Against the Global Pointer" on <https://www.sifive.com/blog/all-aboard-part-3-linker-relaxation-in-riscv-toolchain>.

I am of the opinion that this choice is short-sighted, so I created <https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/298> which was soon closed.
However, I don't receive strong arguments supporting this scheme.
I wish that interested users help me by making some measurement.
