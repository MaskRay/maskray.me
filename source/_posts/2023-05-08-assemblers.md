layout: post
title: Assemblers
author: MaskRay
tags: [clang,gcc]
---

This article provides a description of popular assemblers and their architecture-specific differences.

## Assemblers

GCC generates assembly code and invokes GNU Assembler (also known as "gas"), which is part of GNU Binutils, to convert the assembly code into machine code.
The GCC driver is also capable of accepting assembly input files.
Due to GCC's widespread use, GNU Assembler is arguably the most popular assembler.

Within the LLVM project, the LLVM integrated assembler is a library that is linked by Clang, llvm-mc, and lld (for LTO purposes) to generate machine code.
It supports a wide range of GNU Assembler syntax and can be used as a drop-in replacement for GNU Assembler.

On the Windows platform, the Microsoft Macro Assembler (MASM) is widely used.

On the IBM AIX platform, the AIX assembler is used. In 2019, IBM developers started to modify LLVM integrated assembler to support the AIX syntax. 

On the IBM z/OS platform, the IBM High Level Assembler (HLASM) is used. In 2021, IBM developers started to modify LLVM integrated assembler to support the HLASM syntax.

<!-- more -->

## Architectures

### x86

There are two main syntax dialects: the Microsoft assembler (MASM) syntax and the GNU assembler (GAS) syntax.
The MASM syntax is also known as the Intel syntax, widely utilized in the Windows environment and within the reverse engineering community.
The GNU syntax, also known as the AT&T syntax, derived from PDP-11, is default and dominant in the *NIX world.
The GNU syntax exhibits several key differences:

* The operand list is reversed compared to Intel syntax.
* The four-part generic addressing mode is written as `displacement(base,index,scale)` instead of `[base+index*scale+disp]` in Intel syntax.
* Immediate values are prefixed with `$`, while registers are prefixed with `%`.
* The mnemonics have a suffix indicating the operand size, e.g. `b` for 1 byte, `w` for 2 bytes (Word), `d` for 4 bytes (Dword), and `q` for 8 bytes (Qword).

```
% gcc -S a.c
% cat a.s
...
        movl    var(%rip), %eax
        addl    $3, %eax
% gcc -S -masm=intel a.c
% cat a.s
...
        .intel_syntax noprefix
...
        mov     eax, DWORD PTR var[rip]
        add     eax, 3
```

Using `as -msyntax=intel -mnaked-reg` allows parsing the input in the Intel syntax without a register prefix.
This is similar to including a `.intel_syntax noprefix` directive in the input.

With `llvm-mc -x86-asm-syntax=intel`, the input can be parsed in the Intel syntax. Using `-output-asm-variant=1` will print instructions in Intel syntax.

`llvm-mc -output-asm-variant=1` and [`clang -Xclang --output-asm-variant=1`](https://github.com/llvm/llvm-project/pull/109360) can be used to select the syntax for output.

However, the Intel syntax has not been formally defined, and causes ambiguity for symbol references.
Many x86 instructions take an operand that can be a register or a memory location.
Due to the ambiguity, variable names that collide with registers cannot be used with GCC (<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=53929>).

```
% cat ambiguous.c
int *rip, rax;
int foo() { return rip[rax]; }
% gcc -S -masm=intel ambiguous.c -o -
...
        mov     rax, QWORD PTR rip[rip]
        mov     edx, DWORD PTR rax[rip]
% gcc -c -masm=intel ambiguous.c
/tmp/ccEOMwm6.s: Assembler messages:
/tmp/ccEOMwm6.s:28: Error: invalid use of register
/tmp/ccEOMwm6.s:29: Error: invalid use of register
```

The `_` or `@` prefix in 32-bit Windows's symbol names may be used for disambiguation.

I wish that `-masm=intel` will become the default. ([Please, really, make `-masm=intel` the default for x86](https://gcc.gnu.org/pipermail/gcc/2022-November/240103.html).

For x86 architecture, NASM and YASM are also popular assemblers.
They uses a variant of Intel syntax that omits `PTR` in constructs like `DWORD PTR`.

---

gas's x86 port supports `-O/-O2/-Os` options to [encode instructions using alternative, shorter or more efficient forms](https://sourceware.org/PR22871).
For example, most instructions need a REX prefix to encode REX.W if the operand size is 64-bit.
Sometimes the 32-bit operand size is sufficient and the conversion is beneficial due to shorter sizes without affecting CPU's store forwarding, e.g. `andq $imm31, %reg; xorq %reg, %reg -> andl $imm31, %reg; xorl %reg, %reg`.
AND/OR instructions using the same register can be converted to TEST, e.g. `andq %reg, %reg -> testq %reg, %reg`.

For x86 architecture, NASM is another popular assembler.

## MIPS

Modifiers are utilized to describe different access types of a symbol.
This serves as a bonus as it prevents symbol references from being mistaken as register names.
However, the function call-like syntax can appear verbose.

```asm
lui     a0, %tprel_hi(tls)
add     a0, a0, tp, %tprel_add(tls)
lw      a0, %tprel_lo(tls)(a0)
lui     a1, %hi(var)
lw      a2, %lo(var)(a1)
```

### Power ISA

Power ISA assembly may seem unusual, as general-purpose registers are not prefixed with the `r` prefix.
Whether an integer denotes a register or an immediate value depends on its position as an operand in an instruction.
I find that this difference slightly affects readability.

Similar to x86, postfix modifiers are used to describe different access kinds of a symbol.

```asm
addis 5, 13, tls@tprel@ha
lwz 5, tls@tprel@l(5)
addis 3, 2, var@toc@ha
addi 2, 2, var@toc@l
```

### AArch64

Prefix modifiers are used to describe various access types of a symbol. Personally, this is the modifier syntax that I prefer the most.

```asm
add     x8, x8, :tprel_hi12:tls
add     x8, x8, :tprel_lo12_nc:tls
adrp    x8, fp
ldr     x8, [x8, :lo12:fp]
```

### RISC-V

The modifier syntax is copied from MIPS.

The documentation is available on <https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md>.

## Interaction with compiler drivers

By default, Clang employs the LLVM integrated assembler for most targets.
However, we can opt for using an external assembler by specifying `-fno-integrated-as`.

We can instruct GCC to pass assembler options to gas using `-Wa,`. For example, `gcc -c -Wa,-L a.c` passes the `-L` option to gas.

However, it is important to note that Clang has limited support for `-Wa,` options when the LLVM integrated assembler is used.
This turns out to be fine as many `-Wa,` options aren't really needed.

## Inline assembly

Certain compilers allow the inclusion of assembly code within a high-level language.

The most widely used implementation is GCC [Basic Asm](https://gcc.gnu.org/onlinedocs/gcc/Basic-Asm.html) and [Extended Asm](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html).
On Windows, MSVC supports [inline assembly](https://learn.microsoft.com/en-us/cpp/assembler/inline/inline-assembler) for x86-32 but not for x86-64 and Arm.

Clang supports both GCC and MSVC inline assembly. Clang's MSVC inline assembly can be utilized with x86-64.

Some compilers provide additional variants of inline assembly. Here are some relevant links:

* Free Pascal <https://wiki.freepascal.org/Asm>
* D <https://dlang.org/spec/iasm.html>
* Jai <https://jai.community/t/inline-assembly/139>
* Nim <https://nim-lang.github.io/Nim/manual.html#statements-and-expressions-assembler-statement>

My most wanted feature for GCC inline assembly is [x86: Support a machine constraint to get raw symbol name](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=105576).

Clang concatenates module-level inline assembly and emits it before functions, while GCC retains the order among functions.
```
% cat x.c
void f0() {}
void f1() {}
asm(".if f1-f0 > 2\n.globl x; x: \n.endif");
% gcc -c x.c
% gcc -c x.c -ffunction-sections
/tmp/cckqh92A.s: Assembler messages:
/tmp/cckqh92A.s:40: Error: non-constant expression in ".if" statement
% clang -c x.c
<inline asm>:1:5: error: expected absolute expression
    1 | .if f1-f0 > 2
      |     ^
1 error generated.
```

`MCContext::diagnose` passes the diagnostic message to `MCContext::DiagHandler` (set up by `MachineModuleInfoWrapperPass::doInitialization`), which forwards the message to `LLVMContext::diagnose`, which in turn calls `ClangDiagnosticHandler::handleDiagnostics` (set by `BackendConsumer::HandleTranslationUnit`).

## Notes on GNU Assembler

In the binutils-gdb repository, `opcodes/` contains information on how to assemble and disassemble instructions, but a lot of instruction code also lives on `gas/config/tc-$arch.c`.

`//` is the universal comment marker that is available on all ports. Ports may support additional comment markers, e.g. `#`.

`gas/read.c:potable` defines directives available to all targets. `gcc/config/obj-elf.c:elf_pseudo_table` defines ELF-specific directives. An architecture can define its own directives.

`.set sym, expr` (synonymous with `.equ sym, expr` and `sym = expr`) equates a symbol to an expression. This can either define a symbol or redirect an undefined symbol to another symbol.
The expression can use forward reference, namely symbols that are not defined yet.
The defined symbols have the `STB_LOCAL` binding by default, and can be changed with `.globl` or `.weak` directive.

`.eqv sym, expr` calls `S_SET_FORWARD_REF` on the symbol. The evaluation (including DOT) is postponed to the use site.

For example, below we define two aliases for the symbol `_start`, where `backward` has the `STB_GLOBAL` binding.
```asm
.set forward, _start
.globl _start; _start:
.set backward, _start
.globl backward
```

`.set` can redirect an undefined symbol to another symbol.
```asm
call memcpy@plt

copymem = memcpy
```

Defining a local symbol alias for a global symbol can avoid certain relocations.

`.file` and `.loc` directives are used to create the `.debug_line` section.
If `-g` is specified and none of `.file` and `.loc` is used, gas synthesizes debug information:

* `.debug_info`: `DW_TAG_compile_unit` tags
* `.debug_line`
* `.debug_ranges` or `.debug_rnglists`

`.cfi*` directives are used to create `.eh_frame` or `.debug_frame`.

GNU Assembler implements "INDEFINITE REPEAT BLOCK DIRECTIVES: .IRP AND .IRPC" from MACRO-11.
```asm
.rept 3
  ret
.endr
# => ret; ret; ret

.irpc i,012
  movq $\i, %rax
.endr
# => movq $0, %rax; movq $1, %rax; movq $2, %rax

.irp i,%rax,%rbx,%rcx
  movq \i, %rax
.endr
# => movq %rax, %rax; movq %rbx, %rax; movq %rcx, %rax
```

Unfortunately none of the directives can implement `for (int i = 0; i < 20; i++)`.
`.irpc i,0123456789` gives just 10 iterations while `.irp i,0,1,2,3,4,5,...` is tedious and error-prone.

That said, we can define a macro that expands to a sequence of strings: `0, (0+1), ((0+1)+1), (((0+1)+1)+1), ...`

```asm
% cat a.s
.data
.macro iota n, i=0
.if \n-\i
.byte \i
iota \n, "(\i+1)"
.endif
.endm

iota 16
% as a.s -o a.o
% readelf -x .data a.o

Hex dump of section '.data':
  0x00000000 00010203 04050607 08090a0b 0c0d0e0f ................
```

`"(\i+1)"` creates a new expression but doesn't evaluate it. To evaluate the new expression, we can use the `%` operator in the [alternate macro mode](https://sourceware.org/binutils/docs/as/Altmacro.html) available since [2004](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=caa32fe507720347f37e2d060d64a89d96928db1).
```asm
% cat a.s
.data
.altmacro
.macro iota n, i=0
.if \n-\i
.byte \i
iota \n, %(\i+1)
.endif
.endm

iota 16
% as a.s -o a.o
% readelf -x .data a.o

Hex dump of section '.data':
  0x00000000 00010203 04050607 08090a0b 0c0d0e0f ................
```

gas has supported `\+` for `.macro` directives since May 2024.
I ported it to LLVM integrated assembler and made `\+` available for `.rept` and `.irp`/`.irpc`.
We finally have a nice for loop syntax.

`.if`, `.ifdef`, and `.ifndef` directives allow us to write conditional code in assembly tests without using a C preprocessor.
I often use `.ifdef` to combine positive tests and negative tests in one file.

```asm
# RUN: llvm-mc %s | FileCheck %s
# RUN: not llvm-mc --defsym ERR=1 %s -o /dev/null 2>&1 | FileCheck %s --check-prefix=ERR

# CHECK: ...
## positive tests

.ifdef ERR
# ERR: ...
## negatives tests
.endif
```

ffmpeg uses a nice technique to append `.size` directive after the function body.
```asm
.macro function name align=2
  .macro endfunc
    .size \name, . - \name
    .purgem endfunc
  .endm
  .text
  .balign \align
  \name:
.endm

function foo align=2
  ret
endfunc

function bar align=2
  ret
endfunc
```

`.purgem` ensures that `endfunc` is undefined after instantiated.

The `.reloc` directive creates a relocation with the specified type, location, and addend.

GNU Assembler has [supported `.incbin`](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=7e005732aa09aad97c790cab88d294aeed06eada) since 2001-07 (hey, C/C++ `#embed`).
The review thread mentioned that `.incbin` had been supported by some other assemblers.

To support RISC-V linker relaxation, `gcc/config/tc-riscv.c` starts a new fragment after a relaxable instruction.

A `struct frag` represents a code fragment that contains some known number of bytes, followed by some unknown number of bytes.

A fixup (`struct fix`) describes a location of size 1/2/4/8 within a fragment that will be changed at write time.
The location will be changed to `value(fx_addsy)-value(fx_subsy)+fx_offset`.

A symbol has a name and may be attached to a fragment (`frag != NULL`).
For a local symbol (created by `local_symbol_make`, used by some `STB_LOCAL` symbols), the pointer actually refers to `struct local_symbol`, which stores the section and the value.
Fr a non-local symbol, the symbol is attached to a BFD symbol (`asymbol *bsym`) and contains extra fields (`struct xsymbol *x`).

`frag_offset_fixed_p` computes the offset between two fragments.

### `debug_memory`

Setting `debug_memory` to 1 causes gas to use a smaller obstack chunk size (64 instead of 4096).

### `do_scrub_chars`

The function is called to preprocess the input. A line like `# 123 filename` will be converted to a `.linefile` directive, which will be handled by `new_logical_line_flags` to affect the logical line number.

### `write_object_file`

`relax_seg` iterates over fragments and tries to relax them.

`size_seg` finalizes section sizes.

`adjust_reloc_syms` iterates over fixups and tries changing the referenced symbol to a section symbol.
For an equated symbol (`symbol_equated_reloc_p`), the expression will be used.
`config/tc-riscv.h` defines `tc_fix_adjustable(fixup)` to 0 to prevent section symbol conversion.

`obj_frob_symbol` sets symbol information.

`fix_segment` iterates over fixups and decides whether a fixup is applied as a constant or will become a relocation.
The target-specific `md_apply_fix` is called.

`config/tc-arm.h` defines `TC_FORCE_RELOCATION_SUB_SAME` to force `R_ARM_REL32` relocations against Thumb functions.

`config/tc-riscv.c` defines `md_apply_fix` to split a fixup into two and add a `BFD_RELOC_RISCV_RELAX` fixup (`R_RISCV_RELAX` relocation) for linker-relaxable instructions.

#### Expression evaluation

There are [three expression evaluation strategies](https://sourceware.org/cgit/binutils-gdb/commit/?id=9497f5ac6bc10bdd65ea471787619bde1edca77d): `expr_normal`, `expr_evaluate`, and `expr_defer`.

For a binary operator, the `expr_defer` strategy skips evaluation if `S_IF_FORWARD_REF` is true for either operand.
Otherwise, basic constant folding schemes are applied:

* constant OP constant
* x+constant and x-constant
* y-x when x and y are in the same subsection. In particular, the result can be folded if x and y are only separated by fill fragments.
* ...

`symbol_clone_if_forward_ref`, when called (primarily by `operand` with the `expr_evaluate`/`expr_normal` strategy), clones the expression tree.

For the `expr_evaluate` strategy, `resolve_expression` is additionally performed. The function computes arithmetic operations.

Here are some examples:

`.size exp`: `resolve_expression` called by `elf_frob_symbol` determines the size.

`.if exp`: `expression_and_evaluate` (`expr_evaluate` mode) parses the expression and calls `resolve_expression`.
Regarding `x: call ext; y: .if y-x == 8` for RISC-V, `y-x` is folded to 8 even if `call ext` is linker relaxable.
`HANDLE_CONDITIONAL_ASSEMBLY` in the read loop ignores the input if `current_cframe->ignoring`.

`.long y-x`: `fixup_segment` folds the expression if `sub_symbol_segment == add_symbol_segment && !S_FORCE_RELOC(x, 0) && !S_FORCE_RELOC(y, 0) && !TC_FORCE_RELOCATION_SUB_SAME (fixP, add_symbol_segment)`.
`config/tc-riscv.c` defines `TC_FORCE_RELOCATION_SUB_SAME(seg)` as: seg is a special section (one of `absolute_section, undefined_section, reg_section, expr_section`) or seg contains code (possibly linker relaxable).
As a result, `.long y-x` results in ADD/SUB relocations if x and y are defined in the same text section, even if they are not separated by a linker-relaxable instruction.

## Notes on LLVM assembly and machine code emission

Clang has the capability to output either assembly code or an object file. Generating an object file directly without involving an assembler is referred to as "direct object emission."

To provide a unified interface, `MCStreamer` is created to handle the emission of both assembly code and object files.
The two primary subclasses of `MCStreamer` are `MCAsmStreamer` and `MCObjectStreamer`, responsible for emitting assembly code and machine code respectively.

Now, let's see who communicates with the `MCStreamer` API.

In the data flow pipeline for compilation `C/C++ code -> LLVM IR -> MachineInstr -> MCInst -> assembly code/machine code`, LLVMAsmPrinter calls `MCStreamer` API to emit assembly code or machine code.

In the case of an assembly input file, LLVM creates an `MCAsmParser` object (LLVMMCParser) and a target-specific `MCTargetAsmParser` object.
The `MCAsmParser` is responsible for tokenizing the input, parsing assembler directives, and invoking the `MCTargetAsmParser` to parse an instruction.
Both the `MCAsmParser` and  `MCTargetAsmParser` objects can call `MCStreamer` API to emit assembly code or machine code.

For an instruction parsed by the `MCTargetAsmParser`, if the streamer is an `MCAsmStreamer`, the `MCInst` will be pretty-printed by a target-specific `MCInstPrinter`.
If the streamer is an `MCELFStreamer` (other object file formats are similar), `MCELFStreamer::emitInstToData` will use `${Target}MCCodeEmitter` from LLVM${Target}Desc to encode the `MCInst`, emit its byte sequence, and records needed relocations.
An `ELFObjectWriter` object is used to write the relocatable object file.

`MCAssembler` manages `MCAsmBackend`, `MCCodeEmitter`, and `MCObjectWriter`.
Both `MCAsmStreamer` and `MCObjectStreamer` construct a `MCAssembler` instance.

* `MCObjectStreamer` requires all three instances to be available, so targets that implement machine code emission must implement `MCAsmBackend`.
* `MCAsmStreamer` allows all three to be nullptr. However, most targets implement `MCAsmBackend` and `MCObjectWriter` will be available as well. The instance holds some object file format specific states, even if we don't write an object file.

## Notes on LLVM integrated assembler

## Symbol name prefixes

`PrivateGlobalPrefix`: Temporary labels that are not saved in the symbol table unless `--save-temp-labels` is specified.
This corresponds to system-specific local label prefixes in GNU assembler.

`PrivateLocalPrefix` marks basic blocks labels.

`LinkerPrivateGlobalPrefix` (Mach-O specific: "l") marks a non-temporary symbol that can be used as an atom.
The symbol is not external and will be discarded during linking.
For ELF targets, the `MCContext::createLinkerPrivateTempSymbol` function creates a temporary symbol starting with ".L".

### Workflow

TODO

```
MCAssembler::layout
  MCAssembler::handleFixup
    ELFObjectWriter::recordRelocation
ELFWriter::writeSectionData
```

Some expressions cannot use relocations (e.g., `.size` and `.fill` directives). They set `InSet` as a parameter in various methods in `llvm/lib/MC/`.
This makes expression evaluation different based on the context and should be generally avoided if possible.
This issue is aggravated by the fact that LLVMMC seems to misuse `InSet` and makes `InSet` affect the behavior in situations where it doesn't need to.
Unfortunately, the uses are a bit messy. Additionally, `InSet` appears to be used as a workaround for Mach-O in some places, specifically related to its `.subsections_via_symbols` mechanism.

When assembling an assembly file into an object file, an expression can be evaluated in multiple steps. Two steps are particularly important:

* Parsing time. At this stage, We have a `MCAssembler` object but no `MCAsmLayout` object. Instruction operands and certain directives like `.if` require the ability to evaluate an expression early.
* Object file writing time. At this stage, we have both a `MCAssembler` object and a `MCAsmLayout` object. The `MCAsmLayout` object provides information about the offset of each fragment.

### LLVM-specific directives

`.lto_discard` is an LLVM-specific directive that ignores label and symbol attributes for specified symbols.
LLVMLTO uses the directive to ignore non-prevailing symbols in module-level inline assembly during regular LTO (e.g., `module asm ".weak foo"\nmodule asm ".equ foo,bar"`).

[`.lto_set_conditional sym1, sym2`](https://reviews.llvm.org/D113613) is an LLVM-specific directive that is a variant of assignment (`.set`, `.equ`, `.equiv`).
`sym1` is defined only if `sym2` is defined.
If we create an alias for a symbol defined by module-level inline assembly, we can avoid creating the symbol if the aliasee is dropped by LTO.

z/OS uses IBM High Level Assembler (HLASM). IBM added some support to LLVMMCParser in 2021.

### Inline assembly

There are 3 code paths:

* `-fno-integrated-as` (except XCOFF for AIX): emit raw assembly to `MCAsmStreamer` to without parsing/checking
* `-S -fintegrated-as` (also `-c --save-temps -fintegrated-as`): parse, check, and emit object code to `MCAsmStreamer`
* `-c -fintegrated-as`: parse, check, and emit object code to `MCObjectStreamer`

Parsing inline assembly with LLVMMCParser allows validation and formatting purposes.
In addition, knowing the number of instructions helps implement precise branch relaxation.

#### MSVC inline assembly

`__asm` blocks are parsed for Windows target triples. This extension is available on other targets by specifying `-fasm-blocks` or the broad `-fms-extensions`.
An `__asm` statement is represented as a `clang::MSAsmStmt` object.
`clang::Parser::ParseMicrosoftAsmStatement` parses the inline assembly string and calls `llvm::AsmParser::parseMSInlineAsm`.
It is worth noting that the string may be modified during this process.
For a `clang::MSAsmStmt` object, LLVM IR is generated through `clang::CodeGen::CodeGenFunction::EmitAsmStmt`.

### Span-dependent instruction optimization

LLVM MC calls this assembler relaxation.

```asm
foo: nop; jmp foo
```

With `clang --target=x86_64 -c -mrelax-all` (`llvm-mc -mc-relax-all`), LLVM integrated assembler uses the 5-byte encoding instead of the relaxable 2-byte encoding.

```asm
foo: c.beqz a0, foo
```

With `clang --target=riscv64 -c -mrelax-all`, LLVM integrated assembler splits the instruction into two: `bnez a0, .+6; j foo`.

### Generated debug information

If `-g` is specified and none of `.file` and `.loc` is used, LLVM integrated assembler synthesizes debug information.

### `.debug_line` creation

After parsing an instruction or directive, `MCObjectStreamer::emitBytes` is called to emit bytes.
If we have seen a `.loc` directive (either user-specified or synthesized), we create a temporary label at the current location and add a `MCDwarfLineEntry` entry to the current section.

After the file is parsed, `MCObjectStreamer::finishImpl` calls `MCDwarfLineTable::emit` to emit `.debug_line`.

For each line entry, `MCDwarfLineTable::emitOne` computes how to increment the location and the line number.
`MCObjectStreamer::emitDwarfAdvanceLineAddr` emits `DW_LNE_set_address` to set the address for the first entry.
For the other entries, `MCObjectStreamer::emitDwarfAdvanceLineAddr` selects one of the opcodes in decreasing precedence: `DW_LNS_copy`, standard opcode, `DW_LNS_const_add_pc` plus standard opcode, `DW_LNS_advance_pc`.

There was a "fast path" (`if (AddrDelta->evaluateAsAbsolute(Res, getAssemblerPtr()))`) to emit a byte sequence instead of a `MCDwarfLineAddrFragment`.

### Symbol reassignment

LLVM integrated assembler reports `error: invalid reassignment of non-absolute variable 'x'` for the test case on <https://sourceware.org/bugzilla/show_bug.cgi?id=288>.
```asm
_start:
.set x,0
.long x
.set x,.-_start
.long x
.balign 4
.set x,.-_start
.long x
```

In the Linux kernel, [powerpc/64/asm: Do not reassign labels](https://git.kernel.org/linus/d72c4a36d7ab560127885473a310ece28988b604) removed a reassignment pattern to allow LLVM integrated assembler.

GNU assembler uses `symP->flags.resolving` to detect `symbol definition loop encountered at` errors.
The variable is primarily used by `symbol_clone_if_forward_ref` and `resolve_symbol_value`.

### Expression evaluation

* `evaluateKnownAbsolute`: rarely used. One use case is `.size end-start` to force evaluating to a constant even if the two symbols are separated by a linker-relaxable instruction

TODO span-dependent instructions. _Assembling code for machines with span-dependent instructions_, 1978

Thomas G. Szymanski, _Assembling Code for Machines with Span-Dependent Instructions_, April 1978, coined the term "span-dependent instructions".
_AIX PS/2 and System/370 Programming Tools and Interfaces_ refers to the term and defines `.optim` and `.noopt`.
There is no more information. It's possible that AIX PS/2 and System/370 only use .optim/.noopt for branches, a restricted form of the general span-dependent expressions.

In GNU assembler, .optim/.noopt has been ignored for a long time (since 1992).

https://reviews.llvm.org/D109109 `getSymbolOffsetImpl` is recursively called due to https://github.com/llvm/llvm-project/issues/19577

### Mach-O

Apple cctools assembler was modified from binutils.
It contains code not in upstream binutils, e.g. `.subsections_via_symbols`.

### Mach-O ` a = b + 4; .long a` hack

Normally, relocations referencing a variable are redirected to the symbol in the resolved expression.
However, to [maintain compatibility with Apple cctools assembler](https://github.com/llvm/llvm-project/issues/19577), when the addend is non-zero, the relocation symbol uses the variable itself.

```asm
.data
a = b + 4
.long a   # ARM64_RELOC_UNSIGNED(a) instead of b; This might work around the linker bug(?) when the referenced symbol is b and the addend is 4.

c = d
.long c   # ARM64_RELOC_UNSIGNED(d)

y:
x = y + 4
.long x   # ARM64_RELOC_UNSIGNED(x) instead of y
```

This leads to another workaround in `MCFragment.cpp:getSymbolOffsetImpl`: [[MC] Recursively calculate symbol offset](https://reviews.llvm.org/D109109).
This workaround is to support the following assembly:

```asm
l_a:
l_b = l_a + 1
l_c = l_b
.long l_c
```

### Mach-O label differences

In the LLVM integrated assembler, at layout time, `MCAssembler::relaxDwarfCallFrameFragment` evaluates a label difference created by a CFI instruction.

For Mach-O, if a non-private label appears between a pair of `.cfi_startproc` and `.cfi_endproc`, the `.cfi_startproc` and `.cfi_offset` will be associated with different atoms.

```asm
.cfi_startproc
_foo:
  .cfi_offset 0, 0
.cfi_endproc
```

and you will see an error since LLVM 19 (<https://github.com/llvm/llvm-project/commit/75006466296ed4b0f845cbbec4bf77c21de43b40>).
```
a.s:3:3: error: invalid CFI advance_loc expression
  .cfi_offset 0, 0
  ^
```
