layout: post
title: The dark side of RISC-V linker relaxation
author: MaskRay
tags: [binutils,llvm,linker]
---

Updated in 2023-09.

This article introduces RISC-V linker relaxation and describes the downside.

## Linker optimization/relaxation

Because the linker has a global view and layout information, it can perform some peephole optimizations which are difficult/impossible to do on the compiler side.
Generic link-time code sequence transformation is risky, because semantic information is lost and what the linker sees are byte streams.
However, if every instruction in the candidate code sequence is associated with one ore more relocations, the ABI and the implementation can assign (additional) semantics to the relocation types and make such transformation safe.
This technique is usually called linker optimization or linker relaxation.
It seems that the term "linker optimization" is often used when the number of bytes does not change while "linker relaxation" is used when the number of bytes decreases.

<!-- more -->

In GNU ld, gold and ld.lld, their i386, x86-64 and ppc64 ports have implemented various linker optimizations.
Here is an example of x86-64 GOTPCRELX optimization (available in GNU ld since 2015).

```asm
# Load the address via GOT indirection.
movq x@GOTPCREL(%rip), %rax  # R_X86_64_REX_GOTPCRELX
# =>
# Add an offset to PC.
leaq x(%rip), %rax
```

See [All about Global Offset Table#GOT optimization](/blog/2021-08-29-all-about-global-offset-table#got-optimization) for detail.
More ports have implemented TLS code sequence optimization.
See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage) for details.

Because the term "link-time optimization" is similar to linker relaxation but is usually used in a narrow sense which is very different (communicate symbol resolution to the compiler, combine the information of multiple translation units, and perform IR level optimization), I (and some other folks) have used "linker relaxation" to refer to the transformations without changing the code sequence length.

Now let's move our focus to linker relaxation.

RISC architectures usually need more than one instructions to materialize a symbol address.
There are usually two instructions, the first materializing the high bits and the second materializing the low bits.
In many cases, one single instruction can be used if the symbol is sufficiently near to the program counter.
This requires the linker to be able to delete an instruction from the section.

Another case is the design of branch instructions.
On some RISC architectures (e.g. AVR), a long instruction can be replaced with a short instruction.

```asm
jmp dest    # 4 bytes, R_AVR_CALL
# =>
rjmp dest   # 2 bytes
```

A branch instruction typically has a much shorter range than the 32-bit address space.
In rare cases the jump target may be out of range.
Most RISC architectures use range extension thunks: let the linker redirect the branch instruction to a thunk which materializes the target address and jumps there.

## RISC-V linker relaxation

Instead of giving relocation types such as `R_RISCV_HI20, R_RISCV_LO12, R_RISCV_PCREL_LO12_I, R_RISCV_PCREL_LO12_S, R_RISCV_CALL` additional semantics, the designer introduced a new relocation type `R_RISCV_RELAX`.
You will see that at the same location there are two relocations.
The ELF specification says:

> If multiple _consecutive_ relocation records are applied to the same relocation location (`r_offset`), they are composed instead of being applied independently, as described above. By _consecutive_, we mean that the relocation records are contiguous within a single relocation section. By _composed_, we mean that the standard application described above is modified as follows:
>
> * In all but the last relocation operation of a composed sequence, the result of the relocation expression is retained, rather than having part extracted and placed in the relocated field. The result is retained at full pointer precision of the applicable ABI processor supplement.
> * In all but the first relocation operation of a composed sequence, the addend used is the retained result of the previous relocation operation, rather than that implied by the relocation type.
>
> Note that a consequence of the above rules is that the location specified by a relocation type is relevant for the first element of a composed sequence (and then only for relocation records that do not contain an explicit addend field) and for the last element, where the location determines where the relocated value will be placed. For all other relocation operands in a composed sequence, the location specified is ignored.
>
> An ABI processor supplement may specify individual relocation types that always stop a composition sequence, or always start a new one.

In the composed sequence, `R_RISCV_RELAX` asks the linker to possibly delete bytes.

For branches which can potentially be long ranged, RISC-V uses 2 instructions.
This nicely avoids range extension thunks.
If the branch turns out to be short ranges, GNU ld can delete one instruction and even compress the remaining one.

```asm
call fun                       # auipc ra, ..; jalr ra, ..(ra)
# =>
jal fun  # or c.jal fun
```

### Example

```c
void ext(void);
void foo(void) {
  ext();
  ext();
  ext();
  ext();
}
```

```asm
0000000000000000 <.text>:
# sh_addralign=4, insert NOP of sh_addrline-2 bytes
       0: 01 00         nop
                0000000000000000:  R_RISCV_ALIGN        *ABS*+0x2

0000000000000002 <foo>:
       2: 41 11         addi    sp, sp, -16
       4: 06 e4         sd      ra, 8(sp)
       6: 97 00 00 00   auipc   ra, 0
                0000000000000006:  R_RISCV_CALL ext
                0000000000000006:  R_RISCV_RELAX        *ABS*
       a: e7 80 00 00   jalr    ra
       e: 97 00 00 00   auipc   ra, 0
                000000000000000e:  R_RISCV_CALL ext
                000000000000000e:  R_RISCV_RELAX        *ABS*
      12: e7 80 00 00   jalr    ra
      16: 97 00 00 00   auipc   ra, 0
                0000000000000016:  R_RISCV_CALL ext
                0000000000000016:  R_RISCV_RELAX        *ABS*
      1a: e7 80 00 00   jalr    ra
      ...
```

Two instructions are used to materialize a call.
If the call turns out to be short ranged, the first instruction can be dropped.

```asm
000000000000244 <foo>:
     244: 41 11         addi    sp, sp, -16
     246: 06 e4         sd      ra, 8(sp)
     248: ef 00 80 01   jal     24 <ext>
     24c: ef 00 40 01   jal     20 <ext>
     250: ef 00 00 01   jal     16 <ext>
     254: ef 00 c0 00   jal     12 <ext>
     258: a2 60         ld      ra, 8(sp)
     25a: 41 01         addi    sp, sp, 16
     25c: 11 a0         j       4 <ext>
     25e: 00 00         unimp
```

### Global pointer relaxation

In `-no-pie` mode, GNU ld performs global pointer relaxation.

When producing an executable, linkers define `__global_pointer$` at the start of `.sdata` plus 0x800.
The runtime initializes gp (x3) to the value of `__global_pointer$`. When materializing an address +-2KiB of `__global_pointer$`, we can replace `lui+addi` with one `addi`.

```asm
lui a0, %hi(sym)               # R_RISCV_HI20, R_RISCV_RELAX
addi a0, a0, %lo(sym)          # R_RISCV_LO12, R_RISCV_RELAX
# =>
addi a0, offset(gp)

.L0: auipc a0, %pcrel_hi(sym)  # R_RISCV_PCREL_HI20, R_RISCV_RELAX
addi a0, a0, %pcrel_lo(.L0)    # R_RISCV_PCREL_LO12_I, R_RISCV_RELAX
# =>
addi a0, offset(gp)
```

From binutils 2.41 onwards, GNU ld supports [`--no-relax-gp`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=50980ba351856dff75bb0743bfca62f4c3ab19ff) to disable global pointer relaxation.

As of April 2023, the psABI has [mentioned](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/371) that gp (x3) can be used for a platform-specific purpose.
Such uses are incompatible with global pointer relaxation.

ld.lld [got](https://reviews.llvm.org/D143673) global pointer relaxation in April 2023. We made a deliberate choice to make `--no-relax-gp` the default.

[Haiku](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/298#issuecomment-1344724796), Android, and Fuchsia have mentioned that they'd like to use gp for non-relaxation purposes.

## Assembler implications

### Alignment directives

For an alignment directive, the assembler emits padding bytes (NOP instructions) and creates an `R_RISCV_ALIGN` relocation pointing to the beginning of the padding bytes.
The instruction that needs to be aligned is located at the offset of `R_RISCV_ALIGN` plus its addend.
The linker is responsible for deleting some bytes to ensure the location aligned after relaxing preceding instructions.
```asm
call fun      # 8 bytes
.balign 16    # NOPs with an R_RISCV_ALIGN relocation
mv a1, a0
```

The number of padding bytes is dependent on whether RVC (compressed instructions) is effective.
A `.balign $align` directive results in `$align-2` bytes when RVC is enabled and otherwise `$align-4` bytes.
The number is carefully chosen to the maximum bytes that may be kept after linking. Conceptually the linker only needs to delete bytes, not add new bytes.

However, the design choise is not friendly to the linker when linker relaxation is disabled (`ld --no-relax`).
`$align-2` or `$align-4` bytes do not necessarily make the following instruction aligned.
Therefore, a linker still needs to process `R_RISCV_ALIGN` relocations and delete some bytes, even if `R_RISCV_RELAX` relocations are ignored.

ld.lld prior to 15.0 does not implement linker relaxation, and conservatively bails out with an error like `error: relocation R_RISCV_ALIGN requires unimplemented linker relaxation`.

### Linker friendly `R_RISCV_ALIGN`

In ELF, many relocation types capable of linker optimization are pure optional features: `R_AVR_CALL, R_PPC64_PCREL_OPT, R_X86_64_GOTPCRELX, R_X86_64_REX_GOTPCRELX`, TLS relocation types supporting GD->LE/GD->IE/LD->LE/etc optimizations.

Currently `R_RISCV_ALIGN` just encodes the expected alignment. Can it encode the actual bytes of NOPs beside the expected alignment?
Then linkers not supporting linker relaxation can happily accept `-mrelax` object files.

There are several schemes

* Encode the actual bytes of NOPs in the symbol part of the `r_info` field. The associated symbol can be absolute (`st_shndx==SHN_ABS`).
* Encode the actual bytes of NOPs in the high bits of the `r_addend` field.
* Introduce a new relocation type used together with `R_RISCV_ALIGN`.

The third approach is probably least favored. The first one should be compatible with existing implementations.

I filed <https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/183> for this issue, but closed it after I implemented linker relaxation for ld.lld.

### `.align [abs-expr[, abs-expr[, abs-expr]]]`

In GNU assembler, the third argument of `.align` specifies the maximum number of bytes that should be skipped by this alignment directive.
This is unfortunately not representable with the `R_RISCV_ALIGN` scheme.
The second argument is not either.

### `R_RISCV_RELAX` relaxation does not use the local RVC state

`R_RISCV_RELAX` relaxation uses `e_eflags & EF_RISCV_RVC` in the ELF header to decide whether compressed instructions can be used.
This does not take into account of `.option norvc` regions.
In the following example (adapted from <https://github.com/riscv-collab/riscv-gnu-toolchain/issues/445>), `tail foo` will be relaxed to a compressed instruction `c.j foo`, even if the region is marked with `.option norvc`.

```asm
.option rvc
  add a0, a0, a1
  add a1, a1, a2
  .balign 4        # 2 bytes padding, R_RISCV_ALIGN

.option norvc
  add a0, a1, a2
  tail foo         # R_RISCV_CALL_PLT+R_RISCV_RELAX
  .balign 8        # 4 bytes padding, R_RISCV_ALIGN
foo:
```

Worse, the 4 padding bytes cannot satisfy the requirement of `.bliang 8`.
GNU ld will report an error `6 bytes required for alignment to 8-byte boundary, but only 4 present`.
```
0:  add a0, a0, a1
2:  add a1, a1, a2
4:  add a0, a1, a2
8:  c.j foo
10: // needs 6 bytes to satisfy .balign 8, but there are only 4 padding bytes
```

### Fixup against a local symbol

For a reference to a local symbol in the same section (e.g., a branch target), traditionally no relocation is needed as the fixup is resolved at assembly time.
With linker relaxation, we need to preserve the relocation as the offset can be changed at link time.

```asm
.globl fun
fun:
  jmp .L0     # No relocation
.L0:
  call local0 # No relocation

local0:
```

### Label differences related to text section symbols

For a label difference `A-B` in assembly, if A and B are separated by a linker-relaxable instruction, we should emit a pair of ADD/SUB relocations (e.g. `R_RISCV_ADD32`/`R_RISCV_SUB32`, `R_RISCV_ADD64`/`R_RISCV_SUB64`).
(`R_RISCV_32_PCREL` can be used for certain 32-bit relocations, but GNU assembler only emits `R_RISCV_32_PCREL` for `.eh_frame`.)

The assembler behavior is not well documented (<https://github.com/riscv-non-isa/riscv-asm-manual/issues/80>).
Anyhow, let's see an example.

```asm
.text
w:
.long extern - w   # extern remains undefined
.long w1 - w       # w1 is defined after parsing this expression
.long .L.str - w   # .L.str will be defined (in LLVM MC, .L.str is considered temporary while it isn't in GNU assembler)

w1:
```

`w1 - w` can be folded as a constant.
`extern - w` requires ADD/SUB relocations while `w1 - w` can be folded to a constant (LLVM integrated assembler behavior).
However, GNU assembler emits ADD/SUB relocations for `w1 - w` as `gas/config/tc-riscv.c` conservatively defines `TC_FORCE_RELOCATION_SUB_SAME(seg)` to disable folding for a code section.

Prior to LLVM 17.0, the decision to emit ADD/SUB is made upfront at parsing time with inadequate heuristics (`requiresFixup`).
As a result, older LLVM integrated assembler versions incorrectly suppress `R_RISCV_ADD32/R_RISCV_SUB32` for the following code:

```asm
# Both end and begin are not defined yet. We decide ADD/SUB relocations upfront and don't know they will be needed.
.4byte end-begin

begin:
  call foo
end:
```

Android [riscv64 mterp: Fix "oat" code size calculation.](https://android-review.googlesource.com/c/platform/art/+/2619609) is an instance to work around the issue by making one symbol defined before the `.4byte` directive.

This has been fixed by my [[RISCV] Make linker-relaxable instructions terminate MCDataFragment](https://reviews.llvm.org/D153097) and [[RISCV] Allow delayed decision for ADD/SUB relocations](https://reviews.llvm.org/D155357).
For a label difference `A-B` in assembly, if `A` and `B` are in the same section and not separated by alignment directives, assembler-relaxable instructions, or linker-relaxable instructions, `A-B` can be folded as a constant.

Let's see an example where an assembler-relaxable instruction makes `A-B` not foldable.
```asm
.L1:
  .dword .L2-.L1   # R_RISCV_ADD32/R_RISCV_SUB32
  beq s1, s1, .L1
.L2:
```

Therefore, for the following 32-bit label difference in DWARF, there are generally a pair of `R_RISCV_ADD32`/`R_RISCV_SUB32` in the `-mrelax` mode.
```asm
.section        .debug_info,"",@progbits
...
.word   .Lfunc_end0-.Lfunc_begin0
```

GNU assembler generally emits `R_RISCV_ADD32`/`R_RISCV_SUB32` pairs in more cases. It even emits a pair of `R_RISCV_ADD32`/`R_RISCV_SUB32` for `.word .Lfunc_end0-.Lfunc_begin0` using `-mno-relax`.

TODO gas notes
```
read.c:emit_expr_with_reloc
write.c:2322 resolve_symbol_value
write.c:2339 adjust_reloc_syms
write.c:2348 bfd_map_over_sections (stdoutput, fix_segment, (char *) 0);
  config/tc-riscv.c:4156 If we are deleting this reloc entry, we must fill in the
write.c:2505 bfd_map_over_sections (stdoutput, write_relocs, (char *) 0);
```

## DWARF

DWARF is a widely used debugging information format.

### Code addresses

In DWARF, a debugging information entry (DIE) describing an entity that has a range of machine code adddresses may have `DW_AT_low_pc/DW_AT_high_pc/DW_AT_ranges` attributes to describe the addresses.

On other architectures, the typical implementation uses assembly directives this way and needs one relocation referencing the start of the function.

```asm
.quad   .Lfunc_begin0              # DW_AT_low_pc
.long   .Lfunc_end0-.Lfunc_begin0  # DW_AT_high_pc
```

Due to linker relaxation, the length is not a constant, so the label difference will actually produce two relocations, a pair of `R_RISCV_ADD32` and `R_RISCV_SUB32`.
So RISC-V gives us two relocations which take some space in the object file.

```text
0x0000002a:   DW_TAG_subprogram [2]
                DW_AT_low_pc [DW_FORM_addr]     (0x0000000000000002 ".text")
                  # R_RISCV_64
                DW_AT_high_pc [DW_FORM_data4]   (0x00000040)
                  # Constant on other architectures
                  # Two relocations: R_RISCV_ADD32 and R_RISCV_SUB32
                  # Neither GCC nor Clang uses DW_FORM_addr (DWARF v3)
                DW_AT_frame_base [DW_FORM_exprloc]      (DW_OP_reg8 X8)
                DW_AT_name [DW_FORM_strp]       ( .debug_str[0x00000020] = "foo")
                DW_AT_decl_file [DW_FORM_data1] ("/tmp/c/a.c")
                DW_AT_decl_line [DW_FORM_data1] (2)
                DW_AT_external [DW_FORM_flag_present]   (true)
```

An alternative design is to let the value of the `DW_AT_high_pc` be of class address, specifically, `DW_FORM_addr`. It takes 8 bytes on ELFCLASS64 but can remove one relocation.

```
.quad   .Lfunc_begin0              # DW_AT_low_pc
.quad   .Lfunc_end0                # DW_AT_high_pc
```

### Line number information

Line number information gives association from source file locations to machine instruction addresses.
It is conceptually a matrix with one row for each instruction. The matrix has columns for:

* the source file name
* the source line number
* the source column number
* and so on

DWARF uses a byte-coded language to encode the matrix. The specification says:

> Most of the instructions in a line number program are special opcodes.`

For the above example (consecutive `ext()` calls), on most architectures, a call takes one special opcode of one byte.
`llvm-dwarfdump --debug-line` can dump the matrix:

```asm
# x86-64
            Address            Line   Column File   ISA Flags
            ------------------ ------ ------ ------ --- -------------
0x00000025: 00 DW_LNE_set_address (0x0000000000000000)
0x00000030: 13 address += 0,  line += 1
            0x0000000000000000      2      0      1   0  is_stmt
0x00000031: 05 DW_LNS_set_column (3)
0x00000033: 0a DW_LNS_set_prologue_end
0x00000034: 4b address += 4,  line += 1
            0x0000000000000004      3      3      1   0  is_stmt prologue_end
0x00000035: 75 address += 7,  line += 1
            0x000000000000000b      4      3      1   0  is_stmt
0x00000036: 75 address += 7,  line += 1
            0x0000000000000012      5      3      1   0  is_stmt
0x00000037: 75 address += 7,  line += 1
            0x0000000000000019      6      3      1   0  is_stmt
0x00000038: 75 address += 7,  line += 1
            0x0000000000000020      7      3      1   0  is_stmt
0x00000039: 75 address += 7,  line += 1
            0x0000000000000027      8      3      1   0  is_stmt
```

However, one RISC-V, it is bloated with two relocations associated with one row!
`DW_LNS_advance_line+DW_LNS_fixed_advance_pc+DW_LNS_copy` take 6 bytes. Two `Elf64_Rela` relocations take 48 bytes.

```asm
# RISC-V
            Address            Line   Column File   ISA Flags
            ------------------ ------ ------ ------ --- -------------
0x00000025: 00 DW_LNE_set_address (0x0000000000000002)
0x00000030: 13 address += 0,  line += 1
            0x0000000000000002      2      0      1   0  is_stmt
0x00000031: 05 DW_LNS_set_column (3)
0x00000033: 0a DW_LNS_set_prologue_end
0x00000034: 03 DW_LNS_advance_line (3)
0x00000036: 09 DW_LNS_fixed_advance_pc (0x0002)
0x00000039: 01 DW_LNS_copy
            0x0000000000000004      3      3      1   0  is_stmt prologue_end
0x0000003a: 03 DW_LNS_advance_line (4)
0x0000003c: 09 DW_LNS_fixed_advance_pc (0x000e)
0x0000003f: 01 DW_LNS_copy
            0x0000000000000012      4      3      1   0  is_stmt
0x00000040: 03 DW_LNS_advance_line (5)
0x00000042: 09 DW_LNS_fixed_advance_pc (0x0008)
0x00000045: 01 DW_LNS_copy
            0x000000000000001a      5      3      1   0  is_stmt
...

Relocation section ''.rela.debug_line' at offset 0x7e0 contains 17 entries:
    Offset             Info             Type      Symbol's Value  Symbol's Name + Addend
0000000000000028  0000000400000002 R_RISCV_64    0000000000000002 <null> + 0
0000000000000037  0000000600000022 R_RISCV_ADD16 0000000000000004 <null> + 0
0000000000000037  0000000400000026 R_RISCV_SUB16 0000000000000002 <null> + 0
000000000000003d  0000000a00000022 R_RISCV_ADD16 0000000000000012 <null> + 0
000000000000003d  0000000600000026 R_RISCV_SUB16 0000000000000004 <null> + 0
0000000000000043  0000000b00000022 R_RISCV_ADD16 000000000000001a <null> + 0
0000000000000043  0000000a00000026 R_RISCV_SUB16 0000000000000012 <null> + 0
...
```

54x waste! So, what is wrong?

Well, due to linker relaxation, the address increment between two calls is not a compile-time constant.
A special opecode is encoded with the following formula:

```text
# DWARF v4 introduced maximum_operations_per_instruction for VLIW architectures.
# maximum_operations_per_instruction is 1 and operation_increment is address_advance on non-VLIW architectures.
opcode = line_increment - line_base + (line_range * operation_increment) + opcode_base
```

The variables except operation_increment are compile-time constants, but we don't have a relocation type representing multiplication.
If such a relocation type exists and the compiler can ensure that the maximum `address_advance` does not make the ubyte `opcode` overflow (this is not simple), we can make the linked output much smaller.
That said, relocations are still the primary overhead of object files.

Among major binary formats (Mach-O (8 bytes), PE-COFF (10 bytes)), `Elf64_Rela` (24 bytes) on ELF has a very noticeable overhead.
I don't know whether RISC-V people may want to switch to ELFCLASS32 for 64-bit RISCV in the future.
Currently the Linux kernel associates ELFCLASS32 to ILP32 ABI variants, but nothing prevents ELFCLASS32 from being used for small code model object files.

#### Generated line number information for assembly files

The GNU assembler and LLVM integrated assembler both create a `.debug_line` section for assembly files that have `.file` and `.loc` directives.
When `-g` is specified for assembly files without these directives, debug information is synthesized, including line number information.

However, the LLVM integrated assembler has a bug that creates issues for `-mrelax` with `.debug_line` for both explicitly specified and synthesized `.loc` directives.
This is due to `MCDwarfLineTable::emitOne` calling `MCObjectStreamer::emitDwarfAdvanceLineAddr` to emit line table opcodes for a section.

For an assembly file, `createRISCVELFStreamer` is used to create a `MCObjectStreamer` instance with a `MCAssembler` object.
For a C/C++ file, `createRISCVObjectTargetStreamer` is used to create a `MCObjectStreamer` instance without a `MCAssembler` object.

When a `MCAssembler` object is used, label differences in the line number information are incorrectly foldable.
The LLVM integrated assembler will emit special opcodes that hard code instruction offsets, which can become incorrect when the instruction offset changes due to linker relaxation.

```sh
#!/bin/sh -e
cat > x.c <<eof
void f();
void _start() {
  f();
  f();
  f();
}
eof
# C to object file: correct DW_LNS_fixed_advance_pc
clang --target=riscv64 -g -c x.c
llvm-dwarfdump --debug-line -v x.o | grep \ DW_LNS_fixed_advance_pc

# Assembly to object file with synthesized line number information: incorrect special opcodes
clang --target=riscv64 -S x.c && clang --target=riscv64 -g -c x.s
llvm-dwarfdump --debug-line -v x.o | grep \ DW_LNS_fixed_advance_pc; test $? -eq 1

# Assembly with .loc to object file: incorrect special opcodes
clang --target=riscv64 -S -g x.c && clang --target=riscv64 -c x.s
llvm-dwarfdump --debug-line -v x.o | grep \ DW_LNS_fixed_advance_pc; test $? -eq 1
```

Using `-S` and `-c` on one source file is not commonly used to build projects.
However, `clang --target=riscv64 -g -c x.s` is somewhat common. My suggestion is to remove `-g` for now.
Synthesized debug information is not very useful anyway.

I have [fixed this](https://reviews.llvm.org/D150004) for Clang 17.0.

### Call frame information

A frame description entry (FDE) in call frame information (`.debug_frame/.eh_frame`) encodes the length of the entity. A pair of relocations are needed.

Similar to line number information, the call frame instructions are encoded in a byte-coded language.
The `DW_CFA_advance_loc` instruction has a 6-bit operand (encoded wit hthe opcode) representing that the location delta is `operand * code_alignment_factor`.
The instruction can be relocated by a pair of `R_RISCV_SET6` and `R_RISCV_SUB6`.

### Split DWARF

`-gsplit-dwarf` produces a `.dwo` file which will not be processed by the linker.
If `.dwo` files contain relocations, they will not be resolved. Therefore the practice is that `.dwo` files do not contain relocations.

Currently Clang and GCC use `DW_AT_high_pc` or `DW_AT_ranges` to describe address ranges where the range sizes/endpoints are described by relocations.
Such implementations are incompatible with linker relaxation.
To work with linker relaxation, `DW_AT_high_pc` and `DW_AT_ranges` need to use indexes into `.debug_addr` (e.g. `DW_RLE_startx_endx`).

<https://github.com/llvm/llvm-project/issues/56642>

### Range lists and location lists

For a long time, due to lack of support for `.uleb128` label differences, the compact `DW_LLE_offset_pair` description could not be used.
GCC uses the `DW_LLE_startx_endx` description ([PR99090](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=99090)).
The operands are indices into the `.debug_addr` section. The two values in `.debug_addr` are relocated by `R_RISCV_64` relocations.

In 2023, two relocation types, `R_RISCV_SET_ULEB128` and `R_RISCV_SUB_ULEB128`, are defined.
GNU assembler [emits a pair of `R_RISCV_SET_ULEB128` and `R_RISCV_SUB_ULEB128`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=f1cd8b94e7c941c2a9107c1112ab2339916b8efd) for a `.uleb128` label difference involving a relaxable text section.
There was a bug that a `R_RISCV_SUB_ULEB128` relocation may get [an incorrect non-zero addend](https://sourceware.org/bugzilla/show_bug.cgi?id=31179).
The fix added a `--check-uleb128` option to GNU ld to check such bugs.

I implemented [assembler support for `R_RISCV_SET_ULEB128`/`R_RISCV_SUB_ULEB128` in LLVM integrated assembler](https://reviews.llvm.org/D157657) and 
[lld support for sections without the `SHF_ALLOC` flag](https://github.com/llvm/llvm-project/pull/72610).

## Language-specific data area

In the Itanium C++ ABI, the information needed to process exceptions is called language-specific data area (LSDA).
On ELF targets, this is usually stored in the `.gcc_except_table` section.
See [C++ exception handling ABI](/blog/2020-12-12-c++-exception-handling-abi) for details.

```cpp
int comdat() {
  try { throw 1; }
  catch (int) { return 1; }
  return 0;
}
```

Call site records describe the landing pad offset/length.
Without linker relaxation, the values are assembly time constant and actually `.gcc_except_table` has no relocations referencing text sections.

```asm
  .section .gcc_except_table,"a",@progbits
  .p2align 2
GCC_except_table0:
.Lexception0:
  .byte    255                         # @LPStart Encoding = omit
  .byte    3                           # @TType Encoding = udata4
  .uleb128 .Lttbase0-.Lttbaseref0
.Lttbaseref0:
  .byte    1                           # Call site Encoding = uleb128
  .uleb128 .Lcst_end0-.Lcst_begin0
.Lcst_begin0:
  .uleb128 .Lfunc_begin0-.Lfunc_begin0 # >> Call Site 1 <<
  .uleb128 .Ltmp0-.Lfunc_begin0        #   Call between .Lfunc_begin0 and .Ltmp0
  .byte    0                           #     has no landing pad
  .byte    0                           #   On action: cleanup
  .uleb128 .Ltmp0-.Lfunc_begin0        # >> Call Site 2 <<
  .uleb128 .Ltmp1-.Ltmp0               #   Call between .Ltmp0 and .Ltmp1
  .uleb128 .Ltmp2-.Lfunc_begin0        #     jumps to .Ltmp2
  .byte    1                           #   On action: 1
  .uleb128 .Ltmp1-.Lfunc_begin0        # >> Call Site 3 <<
  .uleb128 .Lfunc_end0-.Ltmp1          #   Call between .Ltmp1 and .Lfunc_end0
  .byte    0                           #     has no landing pad
  .byte    0                           #   On action: cleanup
.Lcst_end0:
  .byte    1                           # >> Action Record 1 <<
                                       #   Catch TypeInfo 1
  .byte   0                            #   No further actions
  .p2align 2
                                       # >> Catch TypeInfos <<
  .long _ZTIi                          # TypeInfo 1
.Lttbase0:
```

With linker relaxation, in the generic case we need a pair of relocations ([R_RISCV_SET_ULEB128 and R_RISCV_SUB_ULEB128](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/commit/96d6e190e9fc04a8517f9ff7fb9aed3e9876cbd6)) to represent label differences.

Older GCC and Clang used `DW_EH_PE_udata4` (`.word`) to encode call site records.
```asm
  .section        .gcc_except_table,"a",@progbits
  .p2align        2
GCC_except_table1:
.Lexception0:
  .byte   255                          # @LPStart Encoding = omit
  .byte   155                          # @TType Encoding = indirect pcrel sdata4
  .uleb128 .Lttbase0-.Lttbaseref0
.Lttbaseref0:
  .byte   3                            # Call site Encoding = udata4
  .uleb128 .Lcst_end0-.Lcst_begin0
.Lcst_begin0:
  .word   .Lfunc_begin1-.Lfunc_begin1  # >> Call Site 1 <<
  .word   .Ltmp2-.Lfunc_begin1         #   Call between .Lfunc_begin1 and .Ltmp2
  .word   0                            #     has no landing pad
  .byte   0                            #   On action: cleanup
  .word   .Ltmp2-.Lfunc_begin1         # >> Call Site 2 <<
  .word   .Ltmp3-.Ltmp2                #   Call between .Ltmp2 and .Ltmp3
  .word   .Ltmp4-.Lfunc_begin1         #     jumps to .Ltmp4
  .byte   1                            #   On action: 1
  .word   .Ltmp3-.Lfunc_begin1         # >> Call Site 3 <<
  .word   .Lfunc_end1-.Ltmp3           #   Call between .Ltmp3 and .Lfunc_end1
  .word   0                            #     has no landing pad
  .byte   0                            #   On action: cleanup
.Lcst_end0:
  .byte   1                            # >> Action Record 1 <<
  ...
```

The other problem is that relocations from `.gcc_except_table` to the text section can cause some linker garbage collection difficulty.
It requires fragmented `.gcc_except_table` sections.
See [C++ exception handling ABI](/blog/2020-12-12-c++-exception-handling-abi) for details.

The call site table length field in the header is encoded in uleb128.
For more fun, the value and other uleb128 offsets/lengths can cause oscillation and require iterations in the assembler.
See GNU as ([PR4029](https://sourceware.org/bugzilla/show_bug.cgi?id=4029)).

## `ld --emit-relocs`

A fair amount of work is needed to keep `--emit-relocs` output updated in the presence of linker relaxation.
Relocations associated with shrunk instructions may look strange and some relocations may feel out of the place.

For GNU ld's AArch64 and x86-64 ports, the `--emit-relocs` code retains the original relocation type even if a linker optimization is applied.
This is partly to communicate more information to the user and partly because the transformation may not be described with any existing relocation type.
The ppc64 port converts certain relocation types.

```sh
cat > aarch64.s <<'eof'
.global _start; _start:
  adrp    x1, :got:x
  ldr     x1, [x1, #:got_lo12:x]
.data; .globl x; .hidden x; x: .word 0
eof
cat > ppc64.s <<'eof'
.globl _start; _start:
  addis 3, 2, .Lhidden@toc@ha // R_PPC64_TOC16_HA(.toc) => R_PPC64_TOC16_HA(hidden)
  ld    3, .Lhidden@toc@l(3)  // R_PPC64_TOC16_LO_DS(.toc) => R_PPC64_TOC16_LO(hidden)
  lwa   3, 0(3)
.data; .globl hidden; .hidden hidden; hidden: .long 0
.section .toc,"aw",@progbits
.Lhidden: .tc hidden[TC], hidden
eof
cat > x86-64.s <<'eof'
.globl _start; _start:
  movq foo@gotpcrel(%rip), %rax
foo: nop
eof

aarch64-linux-gnu-gcc -fuse-ld=lld -B/tmp/Rel/bin -nostdlib aarch64.s -Wl,--emit-relocs -o aarch64 && aarch64-linux-gnu-objdump -dr aarch64
powerpc64le-linux-gnu-gcc -nostdlib ppc64.s -Wl,--emit-relocs -o ppc64 && powerpc64le-linux-gnu-objdump -dr ppc64
gcc -nostdlib x86-64.s -Wl,--emit-relocs -o x86-64 && objdump -dr x86-64
```

```sh
cat > riscv64.s <<'eof'
.global _start; _start:
  call f@plt
  call f@plt
  .balign 8
f: ret
eof

riscv64-linux-gnu-gcc -nostdlib riscv64.s -Wl,--emit-relocs -o riscv64
```

However, the RISC-V port changes the relocation type, including relocation types for relaxable instructions (e.g. `R_RISCV_CALL_PLT`), and markers (`R_RISCV_ALIGN` and `R_RISCV_RELAX`).
`R_RISCV_CALL_PLT` is changed to the `R_RISCV_JAL`, which can describe the relaxed instruction.
`R_RISCV_ALIGN` and `R_RISCV_RELAX` are somehow changed to `R_RISCV_NONE`, losing the original information.

I believe the behavior was not a deliberate decision as there is no `--emit-relocs` test at all in `ld/testsuite/ld-riscv-elf/`.
So far, the unintentional change seems fine but certain relaxations (such as TLSDESC) may not have relocation types associated with the relaxed instructions.

```

Disassembly of section .text:

00000000000002a0 <_start>:
 2a0:   008000ef                jal     2a8 <f>
                        2a0: R_RISCV_JAL        f
 2a4:   004000ef                jal     2a8 <f>
                        2a4: R_RISCV_NONE       *ABS*+0x4
                        2a4: R_RISCV_JAL        f

00000000000002a8 <f>:
 2a8:   8082                    ret
                        2a8: R_RISCV_NONE       *ABS*+0x4
                        2a8: R_RISCV_NONE       *ABS*+0x6
```

I will be quite concerned if RISC-V `--emit-relocs` gets more uses.
I think the expected behavior should be sorted out first.

<https://sourceware.org/bugzilla/show_bug.cgi?id=30844>

## Linker implementation

GNU ld does the following steps:
```c
lang_check_relocs ();

ldemul_after_allocation ();
  ldelf_map_segments
    lang_relax_sections
      lang_size_sections (&relax_again, false);
        bfd_relax_section
          _bfd_riscv_relax_delete
          _bfd_riscv_relax_align

ldwrite ();
  bfd_final_link
    riscv_elf_relocate_section
```

See [RISC-V linker relaxation in lld](/blog/2022-07-10-riscv-linker-relaxation-in-lld) for the ld.lld implementation.

## Does link-time optimization or post link-time optimization help?

Nearly no. This is a phase ordering issue. IR level (and future machine IR level) link-time optimization is performed very early, just after symbol resolution.
It has to be done early because the later steps need access to its emitted input sections.
The LTO library has no layout information to guide its choice of branch instructions.
As an example, for a reference from .data to .text, the LTO library does not know the distance.

## Epilog

I sometimes call linker relaxation poor man's link-time optimization with nice ergonomics.
I agree it is useful, probably more so for embedded systems, but there appears to be great relocatable object file size cost.
Certain toolchain components have high complexity but they are mostly settled down now.

LoongArch folks appear zealous and want to add linker relaxation (https://github.com/loongson/LoongArch-Documentation/pull/77).
I have expressed my concern on the GitHub PR and <https://sourceware.org/pipermail/binutils/2022-December/125322.html> regarding adding support without sorting out all the issues.
