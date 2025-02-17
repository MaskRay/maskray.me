layout: post
title: "Clang's -O0 output: branch displacement and size increase"
author: MaskRay
tags: [llvm,clang,x86]
---

tl;dr Clang 19 will remove the `-mrelax-all` default at `-O0`, significantly decreasing the text section size for x86.

## Span-dependent instructions

In assembly languages, some instructions with an immediate operand can be encoded in two (or more) forms with different sizes.
On x86-64, a direct JMP/JCC can be encoded either in 2 bytes with a 8-bit relative offset or 6 bytes with a 32-bit relative offset.
A short jump is preferred because it takes less space. However, when the target of the jump is too far away (out of range for a 8-bit relative offset), a near jump must be used.

```asm
ja foo    # jump short if above, 77 <rel8>
ja foo    # jump near if above, 0f 87 <rel32>
.nops 126
foo: ret
```

A 1978 paper by Thomas G. Szymanski ("_Assembling Code for Machines with Span-Dependent Instructions_") used the term "span-dependent instructions" to refer to such instructions with short and long forms.
Assemblers grapple with the challenge of choosing the optimal size for these instructions, often referred to as the "branch displacement problem" since branches are the most common type.
A good resource for understanding Szymanski's work is [_Assembling Span-Dependent Instructions_](https://www.complang.tuwien.ac.at/anton/assembling-span-dependent.html).

<!-- more -->

## Start small and grow

Popular assemblers still used today tend to favor a "start small and grow" approach, typically requiring one more pass than Szymanski's "start big and shrink" method.
This approach often results in smaller code and can handle additional complexities like alignment directives.

In LLVM, the [MC library](https://blog.llvm.org/2010/04/intro-to-llvm-mc-project.html) (Machine Code) is reponsible for assembly, disassembly, and object file formats.
Within MC, "assembler relaxation" deals with span-dependent instructions. This is distinct from [linker relaxation](/blog/2021-03-14-the-dark-side-of-riscv-linker-relaxation).

Eli Bendersky provides a detailed explanation in a [2013 blog post](https://eli.thegreenplace.net/2013/01/03/assembler-relaxation) and highlights an interesting behavior:

> For example, when compiling with -O0, the LLVM assembler simply relaxes all jumps it encounters on first sight. This allows it to put all instructions immediately into data fragments, which ensures there's much fewer fragments overall, so the assembly process is faster and consumes less memory.

When `-O0` is enabled and the integrated assembler is used (common by default), [clangDriver](/blog/2021-03-28-compiler-driver-and-cross-compilation) passes the `-mrelax-all` flag to the LLVM MC library.
This sets the `MCRelaxAll` flag in `MCTargetOptions`, instructing the assembler to potentially start with the long form (near) for JMP and JCC instructions on the X86 target only. Other instructions like ADD/SUB/CMP and non-x86 architectures remain unaffected.

## `-mrelax-all` tradeoff

Here is an example:
```c
void foo(int a) {
  // -mrelax-all: near jump (6 bytes)
  // -mno-relax-all or -fno-integrated-as: short jump (2 bytes)
  if (a) bar();
}
```

The assembly (`clang -S`) looks like:
```asm
foo:                                    # @foo
# %bb.0:                                # %entry
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        cmpl    $0, -4(%rbp)
        je      .LBB0_2
# %bb.1:                                # %if.then
        movb    $0, %al
        callq   bar@PLT
.LBB0_2:                                # %if.end
        addq    $16, %rsp
        popq    %rbp
        retq
```

The JE instruction assembles to either a short jump (8-bit relative offset) or near jump (32-bit relative offset).

```
# -mrelax-all
MCSection
  MCDataFragment: empty
  MCAlignFragment: alignment=4
  MCDataFragment: instructions including JE (jump near if equal, 6 bytes)

# -mno-relax-all
MCSection
  MCDataFragment: empty
  MCAlignFragment: alignment=4
  MCDataFragment: instructions before JE (push; mov; sub; mov; cmp)
  MCRelaxableFragment: JE (jump short if equal, 2 bytes). This JE could be expanded, but not in this case.
  MCDataFragment: instructions after JE (mov; call; add; pop; ret)
```

The impact of `-mrelax-all` on text section size is significant, especially when there are many branch instructions.
In an x86-64 release build of lld, `-mrelax-all` increased the `.text` section size by 7.9%.
This translates to a 5.4% increase in VM size and a 4.6% increase in the overall file size.
In a RISC-V rv64gc release build of lld, `-mrelax-all` increased the `.text` section size by 13.7%.
This translates to a 9.0% increase in VM size and a 7.2% increase in the overall file size.

Dean Michael Berris proposed to [remove the `-mrelax-all` default for `-O0`](https://reviews.llvm.org/D21830) in 2016, but it stalled.
`-mrelax-all` caused undesired interaction issues with RISC-V's [conditional branch transforms](https://reviews.llvm.org/D108961), leading Craig Topper to [remove `-mrelax-all`](https://github.com/llvm/llvm-project/pull/88538) at `-O0` for RISC-V recently.

This actually indicated a size regression when the condition branch transform patch landed in 2023.
```asm
blt a1, a2, .Lfoo
beqz a1, .Lfoo
.Lfoo:

# llvm-mc -filetype=obj -triple=riscv64 -mattr=+relax,+c -mc-relax-all
blt a1, a2, .Lfoo       # R_RISCV_BRANCH(.Lfoo), range: +-4KiB
c.beqz a1, .Lfoo        # R_RISCV_BRANCH(.Lfoo)
.Lfoo:
# llvm-mc -filetype=obj -triple=riscv64 -mattr=+relax,+c
bge a1, a2, .+8
jal zero, .Lfoo         # R_RISCV_JAL(.Lfoo), range: +-2MiB
c.bneq a1, .+8
jal zero, .Lfoo         # R_RISCV_JAL(.Lfoo)
```

While `-mrelax-all` might have offered slight compile time benefits in the past, the gains are negligible today.
Benchmarking using stage 2 builds of Clang showed no measurable difference between `-mrelax-all` and `-mno-relax-all`.
On llvm-compile-time-tracker running the llvm-test-suite/CTMark benchmark, compile time actually [increased slightly](https://llvm-compile-time-tracker.com/compare.php?from=ef2ca97f48f1aee1483f0c29de5ba52979bec454&to=18376810f359dbd39d2a0aa0ddfc0f7f50eac199&stat=instructions%3Au) by 0.62% while the text section size [decreased](https://llvm-compile-time-tracker.com/compare.php?from=ef2ca97f48f1aee1483f0c29de5ba52979bec454&to=18376810f359dbd39d2a0aa0ddfc0f7f50eac199&stat=size-text) by 4.44%.

A difference for assembly at different optimisation levels would be quite surprising.
GCC/GNU assembler don't exhibit similar expansion of JMP/JCC instructions even at `-O0`.

These arguments strengthen the case for removing `-mrelax-all` as the default for `-O0`.
[My patch](https://github.com/llvm/llvm-project/pull/90013) has landed and will be included in the next major release, LLVM 19.1.
I have also changed Clang to respect `-mrelax-all` for assembly input: `clang -c --target=x86_64 -mrelax-all a.s`

## Understanding the compile time difference

I have studied a notorious huge file, `llvm/lib/Target/X86/X86ISelLowering.cpp`.

**Fragment count**: A significant difference exists in the number of assembler fragments generated:

* `-mrelax-all`:     89633
* `-mno-relax-all`: 143852

With `-mrelax-all`, the number of `MCRelaxableFragment`s is substantially reduced (to zero when building Clang).
This reduction likely contributes to the compile time difference.

**Fixed-point iteration**: `-mrelax-all` ensures the fixed-point iteration algorithm (almost always) converges in a single iteration.
In contrast, with `-mno-relax-all`, around 6% of sections require additional iterations.
However, this difference is likely not the primary factor affecting compile time.

```
// -mrelax-all
1: 13919
2: 1

// -mno-relax-all
1: 13103
2: 793
3: 23
4: 1
```

## Why didn't people complain about the code size increase?

Because people generally care less about `-O0` code size.

`-O0` is frequently used with `-g` to include debugging information.
This debug information can overshadow the size increase caused by `-mrelax-all`.
(`-O1` or above sacrifices some debuggability.)

In addition, not all projects can be successfully built with `-O0` optimization.
This is typically due to issues like very large programs or mandatory inlining behavior.

For a discussion on size reduction ideas in ELF relocatable files, please check out my blog post about [Light ELF](/blog/2024-04-01-light-elf-exploring-potential-size-reduction).

## PNaCl aligned bundling and `-mrelax-all`

PNaCl's Software Fault Isolation mechanism requires that certain instructions and groups of instructions do not cross a bundle boundary.
When `.bundle_align_mode` is in effect, each instruction is placed in its own fragment, allowing flexible NOP padding.
Enabling `-mrelax-all` enables an [optimization](http://reviews.llvm.org/D8072) that might cause some data fragments to combine, potentially decreasing the total number of fragments.

---

You might also be interested in [my notes](/blog/2023-05-08-assemblers) about GNU assembler and LLVM integrated assembler.
