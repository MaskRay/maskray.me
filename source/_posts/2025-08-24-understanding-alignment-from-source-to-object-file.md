---
layout: post
title: "Understanding alignment - from source to object file"
author: MaskRay
tags: [assembler,llvm,linker]
---

Alignment refers to the practice of placing data or code at memory addresses that are multiples of a specific value, typically a power of 2.
This is typically done to meet the requirements of the programming language, ABI, or the underlying hardware.
Misaligned memory accesses might be expensive or will cause traps on certain architectures.

This blog post explores how alignment is represented and managed as C++ code is transformed through the compilation pipeline: from source code to LLVM IR, assembly, and finally the object file. We'll focus on alignment for both variables and functions.

<!-- more -->

## Alignment in C++ source code

[C++ [basic.align]](https://eel.is/c++draft/basic.align) specifies

> Object types have alignment requirements ([basic.fundamental], [basic.compound]) which place restrictions on the addresses at which an object of that type may be allocated. An alignment is an implementation-defined integer value representing the number of bytes between successive addresses at which a given object can be allocated. An object type imposes an alignment requirement on every object of that type; stricter alignment can be requested using the alignment specifier ([dcl.align]). Attempting to create an object ([intro.object]) in storage that does not meet the alignment requirements of the object's type is undefined behavior.

`alignas` can be used to request a stricter alignment. [[decl.align]](https://eel.is/c++draft/dcl.align)

> An alignment-specifier may be applied to a variable or to a class data member, but it shall not be applied to a bit-field, a function parameter, or an exception-declaration ([except.handle]). An alignment-specifier may also be applied to the declaration of a class (in an elaborated-type-specifier ([dcl.type.elab]) or class-head ([class]), respectively). An alignment-specifier with an ellipsis is a pack expansion ([temp.variadic]).

Example:
```cpp
alignas(16) int i0;
struct alignas(8) S {};
```

If the strictest `alignas` on a declaration is weaker than the alignment it would have without any alignas specifiers, the program is ill-formed.

```
% echo 'alignas(2) int v;' | clang -fsyntax-only -xc++ -
<stdin>:1:1: error: requested alignment is less than minimum alignment of 4 for type 'int'
    1 | alignas(2) int v;
      | ^
1 error generated.
```

However, the GNU extension `__attribute__((aligned(1)))` can request a weaker alignment.

```c
typedef int32_t __attribute__((aligned(1))) unaligned_int32_t;
```

Further reading: [What is the Strict Aliasing Rule and Why do we care?](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8)

## LLVM IR representation

In the LLVM Intermediate Representation (IR), both global variables and functions can have an `align` attribute to specify their required alignment.

[Global variable alignment](https://llvm.org/docs/LangRef.html#global-variables):

> An explicit alignment may be specified for a global, which must be a power of 2. If not present, or if the alignment is set to zero, the alignment of the global is set by the target to whatever it feels convenient. If an explicit alignment is specified, the global is forced to have exactly that alignment. Targets and optimizers are not allowed to over-align the global if the global has an assigned section. In this case, the extra alignment could be observable: for example, code could assume that the globals are densely packed in their section and try to iterate over them as an array, alignment padding would break this iteration. For TLS variables, the module flag MaxTLSAlign, if present, limits the alignment to the given value. Optimizers are not allowed to impose a stronger alignment on these variables. The maximum alignment is 1 << 32.

Function alignment

> An explicit alignment may be specified for a function. If not present, or if the alignment is set to zero, the alignment of the function is set by the target to whatever it feels convenient. If an explicit alignment is specified, the function is forced to have at least that much alignment. All alignments must be a power of 2.

A backend can override this with a preferred function alignment (`STI->getTargetLowering()->getPrefFunctionAlignment()`), if that is larger than the specified align value.
(<https://discourse.llvm.org/t/rfc-enhancing-function-alignment-attributes/88019/3>)

---

In addition, `align` can be used in parameter attributes to decorate a pointer or [vector of pointers](https://reviews.llvm.org/D115161).

## LLVM back end representation

**Global variables**
`AsmPrinter::emitGlobalVariable` determines the alignment for global variables based on a set of nuanced rules:

* With an explicit alignment (`explicit`),
  - If the variable has a section attribute, return `explicit`.
  - Otherwise, compute a preferred alignment for the data layout (`getPrefTypeAlign`, referred to as `pref`). Return `pref < explicit ? explicit : max(E, getABITypeAlign)`.
* Without an explicit alignment: return `getPrefTypeAlign`.

`getPrefTypeAlign` employs a heuristic for global variable definitions: if the variable's size exceeds 16 bytes and the preferred alignment is less than 16 bytes, it sets the alignment to 16 bytes.
This heuristic balances performance and memory efficiency for common cases, though it may not be optimal for all scenarios.
(See [Preferred alignment of globals > 16bytes](https://discourse.llvm.org/t/preferred-alignment-of-globals-16bytes/24410) in 2012)

For assembly output, AsmPrinter emits `.p2align` (power of 2 alignment) directives with a zero fill value (i.e. the padding bytes are zeros).
```
% echo 'int v0;' | clang --target=x86_64 -S -xc - -o -
        .file   "-"
        .type   v0,@object                      # @v0
        .bss
        .globl  v0
        .p2align        2, 0x0
v0:
        .long   0                               # 0x0
        .size   v0, 4
...
```

**Functions**
For functions, `AsmPrinter::emitFunctionHeader` emits alignment directives based on the machine function's alignment settings.

```cpp
void MachineFunction::init() {
...
  Alignment = STI.getTargetLowering()->getMinFunctionAlignment();

  // FIXME: Shouldn't use pref alignment if explicit alignment is set on F.
  if (!F.hasOptSize())
    Alignment = std::max(Alignment,
                         STI.getTargetLowering()->getPrefFunctionAlignment());
```

* The subtarget's minimum function alignment
* If the function is not optimized for size (i.e. not compiled with `-Os` or `-Oz`), take the maximum of the minimum alignment and the preferred alignment.
  For example, `X86TargetLowering` sets the preferred function alignment to 16.

```
% echo 'void f(){} [[gnu::aligned(32)]] void g(){}' | clang --target=x86_64 -S -xc - -o -
        .file   "-"
        .text
        .globl  f                               # -- Begin function f
        .p2align        4
        .type   f,@function
f:                                      # @f
...
        .globl  g                               # -- Begin function g
        .p2align        5
        .type   g,@function
g:                                      # @g
```

The emitted `.p2align` directives omits the fill value argument: for code sections, this space is filled with no-op instructions.

## Assembly representation

GNU Assembler supports multiple alignment directives:

* `.p2align 3`: align to 2**3
* `.balign 8`: align to 8
* `.align 8`: this is identical to `.balign` on some targets and `.p2align` on the others.

Clang supports "direct object emission" (`clang -c` typically bypasses a separate assembler), the LLVMAsmPrinter directly uses the `MCObjectStreamer` API.
This allows Clang to emit the machine code directly into the object file, bypassing the need to parse and interpret alignment directives and instructions from a text-based assembly file.

These alignment directives has an optional third argument: the maximum number of bytes to skip.
If doing the alignment would require skipping more bytes than the specified maximum, the alignment is not done at all.
GCC's `-falign-functions=m:n` utilizes this feature.

## Object file format

In an object file, the section alignment is determined by the strictest alignment directive present in that section.
The assembler sets the section's overall alignment to the maximum of all these directives, as if an implicit directive were at the start.

```
.section .text.a,"ax"
# implicit alignment max(4, 8)

.long 0
.balign 4
.long 0
.balign 8
```

This alignment is stored in the `sh_addralign` field within the ELF section header table.
You can inspect this value using tools such as `readelf -WS` (`llvm-readelf -S`) or `objdump -h` (`llvm-objdump -h`).

## Linker considerations

The linker combines multiple object files into a single executable.
When it maps input sections from each object file into output sections in the final executable, it ensures that section alignments specified in the object files are preserved.

### How the linker handles section alignment

**Output section alignment**: This is the maximum `sh_addralign` value among all its contributing input sections.
This ensures the strictest alignment requirements are met.

**Section placement**:
The linker also uses input `sh_addralign` information to position each input section within the output section.
As illustrated in the following example, each input section (like `a.o:.text.f` or `b.o:.text`) is aligned according to its `sh_addralign` value before being placed sequentially.

```
output .text
  # align to sh_addralign(a.o:.text). No-op if this is the first section without any preceding DOT assignment or data command.
  a.o:.text
  # align to sh_addralign(a.o:.text.f)
  a.o:.text.f
  # align to sh_addralign(b.o:.text)
  b.o:.text
  # align to sh_addralign(b.o:.text.g)
  b.o:.text.g
```

**Link script control**
A linker script can override the default alignment behavior.
The `ALIGN` keyword enforces a stricter alignment. For example `.text : ALIGN(32) { ... }` aligns the section to at least a 32-byte boundary.
This is often done to optimize for specific hardware or for memory mapping requirements.

The `SUBALIGN` keyword on an output section overrides the input section alignments.

**Padding**:
To achieve the required alignment, the linker may insert padding between sections or before the first input section (if there is a gap after the output section start).
The fill value is determined by the following rules:

* If specified, use the [`=fillexp` output section attribute](https://sourceware.org/binutils/docs/ld/Output-Section-Attributes.html) (within an output section description).
* If a non-code section, use zero.
* Otherwise, use a trap or no-op instructin.

### Padding and section reordering

Linkers typically preserve the order of input sections from object files.
To minimize the padding required between sections, linker scripts can use a `SORT_BY_ALIGNMENT` keyword to arrange input sections in descending order of their alignment requirements.
Similarly, GNU ld supports [`--sort-common`](/blog/2022-02-06-all-about-common-symbols#sort-common) to sort COMMON symbols by decreasing alignment.

While this sorting can reduce wasted space, modern linking strategies often prioritize other factors, such as cache locality (for performance) and data similarity (for Lempel–Ziv compression ratio), which can conflict with sorting by alignment.
(Search `--bp-compression-sort=` on [Explain GNU style linker options](/blog/2020-11-15-explain-gnu-linker-options)).

### System page size

The alignment of a variable or function can be as large as the system page size. Some implementations allow a larger alignment.
([Over-aligned segment](/blog/2023-12-17-exploring-the-section-layout-in-linker-output#over-aligned-segment))

## ABI compliance

Some platforms have special rules. For example,

* On SystemZ, the `larl` (load address relative long) instruction cannot generate odd addresses. To prevent GOT indirection, compilers ensure that symbols are at least aligned by 2. ([Toolchain notes on z/Architecture](/blog/2024-02-11-toolchain-notes-on-z-architecture))
* On AIX, the default alignment mode is `power`: for double and long double, the first member of this data type is aligned according to its natural alignment value; subsequent members of the aggregate are aligned on 4-byte boundaries. (<https://reviews.llvm.org/D79719>)
* z/OS caps the maximum alignment of static storage variables to 16. (<https://reviews.llvm.org/D98864>)

The standard representation of the the Itanium C++ ABI requires member function pointers to be even, to distinguish between virtual and non-virtual functions.

> In the standard representation, a member function pointer for a virtual function is represented with ptr set to 1 plus the function's v-table entry offset (in bytes), converted to a function pointer as if by `reinterpret_cast<fnptr_t>(uintfnptr_t(1 + offset))`, where `uintfnptr_t` is an unsigned integer of the same size as `fnptr_t`.

Conceptually, a pointer to member function is a tuple:

* A function pointer or virtual table index, discriminated by the least significant bit
* A displacement to apply to the `this` pointer

Due to the least significant bit discriminator, members function need a stricter alignment even if `__attribute__((aligned(1)))` is specified:

```cpp
virtual void bar1() __attribute__((aligned(1)));
```

Side note: check out [MSVC C++ ABI Member Function Pointers](https://rants.vastheman.com/2021/09/21/msvc/) for a comparison with the MSVC C++ ABI.

## Architecture considerations

Contemporary architectures generally support unaligned memory access, likely with very small performance penalties.
However, some implementations might restrict or penalize unaligned accesses heavily, or require specific handling.
Even on architectures supporting unaligned access, atomic operations might still require alignment.

* On AArch64, a bit in the system control register `sctlr_el1` enables alignment check.
* On x86, if the AM bit is set in the CR0 register and the AC bit is set in the EFLAGS register, alignment checking of user-mode data accessing is enabled.

Linux's RISC-V port supports `prctl(PR_SET_UNALIGN, PR_UNALIGN_SIGBUS);` to enable strict alignment.

`clang -fsanitize=alignment` can detect misaligned memory access. Check out my [write-up](/blog/2023-01-29-all-about-undefined-behavior-sanitizer#fsanitizealignment).

In 1989, US Patent 4814976, which covers "RISC computer with unaligned reference handling and method for the same" (4 instructions: lwl, lwr, swl, and swr), was granted to MIPS Computer Systems Inc.
It caused a barrier for other RISC processors, see [The Lexra Story](https://www.probell.com/lexra/).

> Almost every microprocessor in the world can emulate the functionality of unaligned loads and stores in software. MIPS Technologies did not invent that. By any reasonable interpretation of the MIPS Technologies' patent, Lexra did not infringe. In mid-2001 Lexra received a ruling from the USPTO that all claims in the the lawsuit were invalid because of prior art in an IBM CISC patent. However, MIPS Technologies appealed the USPTO ruling in Federal court, adding to Lexra's legal costs and hurting its sales. That forced Lexra into an unfavorable settlement. The patent expired on December 23, 2006 at which point it became legal for anybody to implement the complete MIPS-I instruction set, including unaligned loads and stores.

## Aligning code for performance

GCC offers a family of performance-tuning options named `-falign-*`, that instruct the compiler to align certain code segments to specific memory boundaries.
These options might improve performance by preventing certain instructions from crossing cache line boundaries (or instruction fetch boundaries), which can otherwise cause an extra cache miss.

* `-falign-function=n`: Align functions.
* `-falign-labels=n`: Align branch targets.
* `-falign-jumps=n`: Align branch targets, for branch targets where the targets can only be reached by jumping.
* `-falign-loops=n`: Align the beginning of loops.

Important considerations

**Inefficiency with Small Functions**:
Aligning small functions can be inefficient and may not be worth the overhead. To address this, GCC introduced `-flimit-function-alignment` in 2016.
The option sets `.p2align` directive's max-skip operand to the estimated function size minus one.

```
% echo 'int add1(int a){return a+1;}' | gcc -O2 -S -fcf-protection=none -xc - -o - -falign-functions=16 | grep p2align
        .p2align 4
% echo 'int add1(int a){return a+1;}' | gcc -O2 -S -fcf-protection=none -xc - -o - -falign-functions=16 -flimit-function-alignment | p2align
        .p2align 4,,3
```

The max-skip operand, if present, is evaluated at parse time, so you cannot do:
```asm
.p2align 4, , b-a
a:
  nop
b:
```

In LLVM, the x86 backend does not implement `TargetInstrInfo::getInstSizeInBytes`, making it challenging to implement `-flimit-function-alignment`.

**Cold code**:
These options don't apply to cold functions. To ensure that cold functions are also aligned, use `-fmin-function-alignment=n` instead.

**Benchmarking**:
Aligning functions can make benchmarks more reliable.
For example, on x86-64, a hot function less than 32 bytes might be placed in a way that uses one or two cache lines (determined by `function_addr % cache_line_size`), making benchmark results noisy.
Using `-falign-functions=32` can ensure the function always occupies a single cache line, leading to more consistent performance measurements.

---

LLVM notes: In `clang/lib/CodeGen/CodeGenModule.cpp`, `-falign-function=N` sets the alignment if a function does not have the `gnu::aligned` attribute.

A hardware loop typically consistants of 3 parts:

A low-overhead loop (also called a zero-overhead loop) is a hardware-assisted looping mechanism found in many processor architectures, particularly digital signal processors (DSPs).
The processor includes dedicated registers that store the loop start address, loop end address, and loop count.
A hardware loop typically consists of three components:

* Loop setup instruction: Sets the loop end address and iteration count
* Loop body: Contains the actual instructions to be repeated
* Loop end instruction: Jumps back to the loop body if further iterations are required

Here is an example from Arm v8.1-M low-overhead branch extension.

```asm
1:
  dls lr, Rn    // Setup loop with count in Rn
  ...           // Loop body instructions
2:
  le lr, 1b     // Loop end - branch back to label 1 if needed
```

To minimize the number of cache lines used by the loop body, ideally the loop body (the instruction immediately following DLS) should be aligned to a 64-byte boundary.
However, GNU Assembler lacks a directive to specify alignment like "align DLS to a multiple of 64 plus 60 bytes."
Inserting an alignment after the DLS is counterproductive, as it would introduce unwanted NOP instructions at the beginning of the loop body, negating the performance benefits of the low-overhead loop mechanism.

It would be desirable to simulate the functionality with `.org ((.+4+63) & -64) - 4  // ensure that .+4 is aligned to 64-byte boundary`, but this complex expression involves bitwise AND and is not a relocatable expression.
LLVM integrated assembler would report `expected absolute expression` while GNU Assembler has a similar error.

A potential solution would be to extend the alignment directives with an optional offset parameter:

```asm
# Align to 64-byte boundary with 60-byte offset, using NOP padding in code sections
.balign 64, , , 60

# Same alignment with offset, but skip at most 16 bytes of padding
.balign 64, , 16, 60
```

Xtensa's `LOOP` instructions has similar alignment requirement, but I am not familiar with the detail. The GNU Assembler uses the special alignment as a special machine-dependent fragment.
(<https://sourceware.org/binutils/docs/as/Xtensa-Automatic-Alignment.html>)
