---
layout: post
title: Toolchain notes on MIPS
author: MaskRay
tags: [llvm,linker]
---

This article describes some notes about MIPS with a focus on the ELF object file format, GCC, binutils, and LLVM/Clang.

In the llvm-project project, I sometimes find myself assigned as a reviewer for MIPS patches.
I want to be transparent that I have no interest in MIPS, but my concern lies with the specific components that are impacted (Clang driver, ld.lld, MC, compiler-rt, etc.).
Therefore, regrettably, I have to spend some time studying MIPS.

> Using copper as a mirror, one can straighten their attire; using the past as a mirror, one can understand rise and fall; using people as a mirror, one can discern gains and losses.  -- _貞觀政要_

<!-- more -->

## ISAs

Earlier ISAs, such as MIPS I, MIPS II, MIPS III, and MIPS IV, can be selected by `gcc -march=mips[1-4]`.
MIPS V has no implementation and GCC doesn't support `-march=mips5`.

Successive versions are split into MIPS32 and MIPS64, named Release 1, Release 2, Release 3, and Release 5, can be selected by `-march=mips{32,64}` and `-march=mips{32,64}r{2,3,5}`.
Release 4 is skipped and `-march=mips{32,64}r4` is not supported.

Release 6 is very different from prior releases, with removal and reorganization of some instructions.

* `-march=mips{32,64}` define `__mips_isa_rev` to 1.
* `-march=mips{32,64}r2` define `__mips_isa_rev` to 2.
* `-march=mips{32,64}r3` define `__mips_isa_rev` to 3.
* `-march=mips{32,64}r5` define `__mips_isa_rev` to 5.
* `-march=mips{32,64}r6` define `__mips_isa_rev` to 6.
* Older processors don't define `__mips_isa_rev`.

The ISA revisions of GCC supported `-march=` values can be found at `gcc/config/mips/mips-cpus.def`.

The `EF_MIPS_ARCH` (0xf0000000) part of `e_flags` indicates the ISA.
This is not really a good design because ISA levels may not be linear and the small number of bits easily run out.
`e_flags` has only 32 bits in both ELFCLASS32 and ELFCLASS64 objects. Allocating every spare bit should be done extremely carefully.

## ABIs

### o32

<https://refspecs.linuxfoundation.org/elf/mipsabi.pdf>

This is an ILP32 ABI designed for the 32-bit CPU MIPS R3000 that implements the MIPS I ISA.
It is the original System V Processor Supplement ABI for MIPS.
Common target triples: `mips[el]-unknown-linux-gnu`.

In GCC, `-mabi=32` selects this ABI, which is identified as `ABI_32`.
The macro `_ABIO32=1` is defined.

Assemblers set the `EF_MIPS_ABI_O32` flag in `e_flags`.

There are some major flaws:

* The calling convention provides just 4 integer registers for arguments (inadequate).
* The ABI is committed to MIPS I and makes odd-numbered floating-point registers inaccessible.
* Only two floating-point registers $12 and $14 hold arguments. The rest use integer registers, which are severely limited.

MIPS I has 32 floating-point registers. Two registers are paired for holding a double precision number.

* For single-precision values, the even-numbered floating-point register holds the value.
* For double-precision values, the even-numbered floating-point register holds the least significant 32 bits of the value and the odd-numbered floating-point register holds the most significant 32 bits of the value.

For o32, `-mfp64` selects a variant that enables 64-bit floating-point registers.
This requires at least `-march=mips32r2` and GCC will emit `.module fp=64; .module oddspreg`.

When a 64-bit capable CPU is used with o32, assemblers set the `EF_MIPS_32BITMODE` flag in `e_flags`.

### n64

This is an LP64 ABI designed for 64-bit CPUs (MIPS III and newer).
Common target triples: `mips64[el]-unknown-linux-gnuabi64`.

Does anyone know where I can get a copy of the ABI document?

In GCC, `-mabi=64` selects this ABI, which is identified as `ABI_64`.
The macro `_ABI64=3` is defined.

Assemblers do not set a particular bit in `e_flags`.

This ABI has fixed some flaws of o32 but introduces an issue related to compiler optimization: $gp is now callee-saved.
We will discuss this later.

### n32

This is an ILP32 ABI for 64-bit CPUs (MIPS III and newer), often named n32.
Common target triples: `mips64[el]-unknown-linux-gnuabin32`

In GCC, `-mabi=n32` selects this ABI, which is identified as `ABI_N32`.
The macro `_ABIN32=2` is defined.
`-march=` values such as `mips3` and `mips64` are compatible.

Assemblers set the `EF_MIPS_ABI2` bit in `e_flags`.

### o64

This is o32 extended for 64-bit CPUs.
<https://gcc.gnu.org/projects/mipso64-abi.html>

In GCC, `-mabi=o64` selects this ABI, which is identified as [`ABI_O64`](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=a53f72db6c5b94405ec1e84970177aacbf477a7c).
The macro `_ABIO64=4` is defined.

Assemblers set the `EF_MIPS_ABI_O64` flag in `e_flags`.

### EABI

Assemblers set the `EF_MIPS_ABI_EABI32` flag in `e_flags`.

---

o32 and o64 are called `TARGET_OLDABI` in GCC.
n32 and n64 are called `TARGET_NEWABI` in GCC.

`gcc/config.gcc` defines the default ABI (macro `MIPS_ABI_DEFAULT`) for different target triples.

In the old ABIs, the local label prefix for assembly is `$`, which is strange.
This is confusing as register names are also prefixed with `$`.

---

GCC emits a special section to indicate the ABI for GDB.

* o32: `.mdebug.abi32`
* n32: `.mdebug.abiN32`
* n64: `.mdebug.abi64`
* o64: `mdebug.abiO64`
* 32-bit EABI: `.mdebug.eabi32`
* 64-bit EABI: `.mdebug.eabi64`

### MIPS16 and microMIPS

In MIPS16 and microMIPS modes, text labels are marked with the `STO_MIPS_MIPS16` or `STO_MIPS_MICROMIPS` bit, and their addresses have bit 0 set.
This affects how labels and their differences are handled in assembly code.

Consider the following example:

```asm
B0 = .
B1:
  move    $25,$4
C0 = .
C1:
  move    $25,$4
.data
.long C0-B0, C1-B0, C0-B1, C1-B1   // Output: 2, 3, 1, 2
.long B0, B1, C0, C1               // Output: 0, 1, 2, 3
```

In the `.gcc_except_table` section, the call site table uses similar label differences to define code ranges. For example:

```asm
$LFB0 = .
...
$LEHB0 = .
...
$LEHE0 = .
...
$L3:

        .uleb128 $LEHB0-$LFB0   // call site start, variable minus variable
        .uleb128 $LEHE0-$LEHB0  // call site end, variable minus variable
        .uleb128 $L3-$LFB0      // landing pad offset, label minus variable
```

These differences ensure exception handling, as documented in the GCC fix for MIPS16 EH issues ([[committed] Fix a lot of MIPS16 EH failures](https://gcc.gnu.org/pipermail/gcc-patches/2008-November/252316.html)).

In LLVM, `MCAsmInfo::UseAssignmentForEHBegin` is [introduced](https://github.com/llvm/llvm-project/commit/f491704e2297e5986b0fe7277403e35ffabaaec4) to declare the function begin symbol as `$func_begin0 = .` instead of `$func_begin0:`.

## gp register

In n32/n64 ABIs, $gp is callee-saved. This is unfortunate and often leads to more instructions compared with o32.
Prologue/epilogue code needs to save and restore $gp, so tail calls are often inhibited.
Technically, the GOT address is not necessarily loaded in $gp and GCC can pick a volatile register if profitable, but GCC doesn't have the optimizion.

`.cpsetup $reg1, offset|$reg2, label` pseudo-op was introduced ([gas](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=6478892d2e3f83fe2db6d2a904a068bdfba9a2c6)) to assist hand-written assembly.

## Floating-point numbers

See [NaN encodings](https://sourceware.org/binutils/docs/as/MIPS-NaN-Encodings.html).

IEEE 754-2008 says

> A quiet NaN bit string should be encoded with the first bit (d1) of the trailing significand field T being 1.

`-mnan=2008` instructs GCC to emit a `.nan 2008` directive, which causes GNU assembler to set the `EF_MIPS_NAN2008` bit in `e_flags`.

MIPS32/MIPS64 Release 6 defaults to `-mnan=2008` while prior ISAs default to `-mnan=legacy`.

### `-modd-spreg` for o32

Enable the use of odd-numbered single-precision floating-point registers for the o32 ABI.
This option requires `-march=mips32` or above.

I think the option should not be used with n32 or n64.

## Dynamic tags

```
% grep DT_MIPS_ include/elf/mips.h
#define DT_MIPS_RLD_VERSION     0x70000001
#define DT_MIPS_TIME_STAMP      0x70000002
#define DT_MIPS_ICHECKSUM       0x70000003
#define DT_MIPS_IVERSION        0x70000004
#define DT_MIPS_FLAGS           0x70000005
#define DT_MIPS_BASE_ADDRESS    0x70000006
#define DT_MIPS_MSYM            0x70000007
#define DT_MIPS_CONFLICT        0x70000008
#define DT_MIPS_LIBLIST         0x70000009
#define DT_MIPS_LOCAL_GOTNO     0x7000000a
#define DT_MIPS_CONFLICTNO      0x7000000b
#define DT_MIPS_LIBLISTNO       0x70000010
#define DT_MIPS_SYMTABNO        0x70000011
#define DT_MIPS_UNREFEXTNO      0x70000012
#define DT_MIPS_GOTSYM          0x70000013
#define DT_MIPS_HIPAGENO        0x70000014
#define DT_MIPS_RLD_MAP         0x70000016
#define DT_MIPS_DELTA_CLASS     0x70000017
/* Number of entries in DT_MIPS_DELTA_CLASS.  */
#define DT_MIPS_DELTA_CLASS_NO  0x70000018
#define DT_MIPS_DELTA_INSTANCE  0x70000019
/* Number of entries in DT_MIPS_DELTA_INSTANCE.  */
#define DT_MIPS_DELTA_INSTANCE_NO       0x7000001a
#define DT_MIPS_DELTA_RELOC     0x7000001b
/* Number of entries in DT_MIPS_DELTA_RELOC.  */
#define DT_MIPS_DELTA_RELOC_NO  0x7000001c
#define DT_MIPS_DELTA_SYM       0x7000001d
/* Number of entries in DT_MIPS_DELTA_SYM.  */
#define DT_MIPS_DELTA_SYM_NO    0x7000001e
#define DT_MIPS_DELTA_CLASSSYM  0x70000020
/* Number of entries in DT_MIPS_DELTA_CLASSSYM.  */
#define DT_MIPS_DELTA_CLASSSYM_NO       0x70000021
#define DT_MIPS_CXX_FLAGS       0x70000022
#define DT_MIPS_PIXIE_INIT      0x70000023
#define DT_MIPS_SYMBOL_LIB      0x70000024
#define DT_MIPS_LOCALPAGE_GOTIDX        0x70000025
#define DT_MIPS_LOCAL_GOTIDX    0x70000026
#define DT_MIPS_HIDDEN_GOTIDX   0x70000027
#define DT_MIPS_PROTECTED_GOTIDX        0x70000028
#define DT_MIPS_OPTIONS         0x70000029
#define DT_MIPS_INTERFACE       0x7000002a
#define DT_MIPS_DYNSTR_ALIGN    0x7000002b
#define DT_MIPS_INTERFACE_SIZE  0x7000002c
#define DT_MIPS_RLD_TEXT_RESOLVE_ADDR   0x7000002d
#define DT_MIPS_PERF_SUFFIX     0x7000002e
#define DT_MIPS_COMPACT_SIZE    0x7000002f
#define DT_MIPS_GP_VALUE        0x70000030
#define DT_MIPS_AUX_DYNAMIC     0x70000031
#define DT_MIPS_PLTGOT         0x70000032
#define DT_MIPS_RWPLT          0x70000034
#define DT_MIPS_RLD_MAP_REL    0x70000035
#define DT_MIPS_XHASH          0x70000036
/* Flags which may appear in a DT_MIPS_FLAGS entry.  */
```

Among these dynamic tags, only `DT_MIPS_LOCAL_GOTNO`, `DT_MIPS_GOTSYM`, `DT_MIPS_SYMTABNO`, and `DT_MIPS_PLTGOT` are really needed in rtld.
[All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table) analyzed `DT_MIPS_LOCAL_GOTNO` and `DT_MIPS_SYMTABNO-DT_MIPS_GOTSYM`.

For executable output, `DT_MIPS_RLD_MAP_REL` holds the offset to the linker synthesized `.rld_map`.
If `DT_MIPS_RLD_MAP_REL` is unavailable, glibc rtld looks for `DT_MIPS_RLD_MAP`, which is emitted for `ET_EXEC` executables.

## Position-independent code

In all the ABIs, "when calling position independent functions $25 must contain the address of the called function."

We will see below that non-PIC code may call PIC functions (in another translation unit) with `j` or `jal` instructions.
When the callee is defined in the executable, to ensure that register $25 is correct at the callee entry, the linker must insert a trampoline.
```asm
// Prior to Release 6
  jal func

=>

  jal __LA25Thunk_func

__LA25Thunk_foo:
  lui $25, %hi(func)
  j func
  addiu $25, $25, %lo(func)
  nop
```

When the callee is defined in a shared object, the linker will create a PLT entry to set up register $25.

See "Linking PIC and non-PIC in the same object" on [RFC: Adding non-PIC executable support to MIPS](https://gcc.gnu.org/legacy-ml/gcc/2008-06/msg00670.html) for detail.
This RFC [added `STO_MIPS_PLT` to binutils](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=861fb55ab50ac986e273334639c4c44bb3353efb).

## `-mabicalls`, `-mshared`, `-fpic`

GCC's MIPS port uses several interrelated options and internal macros to control PIC code generation.
The key macros in `gcc/config/mips/mips.h` are:

```c
// True for n64 ABI (64-bit symbols in ELF)
#define FILE_HAS_64BIT_SYMBOLS  (mips_abi == ABI_64)
#define ABI_HAS_64BIT_SYMBOLS   (FILE_HAS_64BIT_SYMBOLS && Pmode == DImode && !TARGET_SYM32)

// True when we can use absolute addressing with -mabicalls
// Key: disabled for n64 because "GOT accesses are so much shorter" than 64-bit absolute sequences
#define TARGET_ABSOLUTE_ABICALLS \
  (TARGET_ABICALLS && !TARGET_SHARED && TARGET_EXPLICIT_RELOCS && !ABI_HAS_64BIT_SYMBOLS)

// .option pic0 (non-PIC with abicalls) vs .option pic2 (full PIC)
#define TARGET_ABICALLS_PIC0  (TARGET_ABSOLUTE_ABICALLS && TARGET_PLT)
#define TARGET_ABICALLS_PIC2  (TARGET_ABICALLS && !TARGET_ABICALLS_PIC0)

// Can use J and JAL instructions
#define TARGET_ABSOLUTE_JUMPS  (!flag_pic || TARGET_ABSOLUTE_ABICALLS)
```

### How the options interact

`-mabicalls` (default for Linux) enables SVR4-style PIC code generation.

In `mips.opt`, `TARGET_SHARED` defaults to 1 (`-mshared`). However, the driver spec in `gnu-user.h` modifies this:
```c
/* Default to -mno-shared for non-PIC.  */
#define NO_SHARED_SPECS \
  " %{mshared|mno-shared:;:%{" NO_FPIE_AND_FPIC_SPEC ":-mno-shared}}"
```

This means:

* If `-mshared` or `-mno-shared` is explicit: use that
* Else if no `-fpic`/`-fpie`: add `-mno-shared`
* Else (`-fpic`/`-fpie` given): keep default `-mshared`

So effectively:

* Default (no flags): `-mno-shared` added → non-PIC for o32/n32
* `-fpic`: stays `-mshared` → full PIC
* `-fno-pic`: `-mno-shared` added → non-PIC
* Explicit `-mshared`/`-mno-shared` overrides `-fpic`/`-fno-pic`

**For o32/n32** (`ABI_HAS_64BIT_SYMBOLS` is false):

* `-mno-shared`: `TARGET_ABSOLUTE_ABICALLS` can be true → `.option pic0`, direct `jal`, absolute addressing
* `-mshared`: `TARGET_ABSOLUTE_ABICALLS` is false → `.option pic2`, full PIC

**For n64** (`ABI_HAS_64BIT_SYMBOLS` is true):

* `TARGET_ABSOLUTE_ABICALLS` is **always false** due to `!ABI_HAS_64BIT_SYMBOLS`
* With `-mabicalls` (default), `TARGET_ABICALLS_PIC2` is always true - full PIC regardless of `-mshared`/`-fpic`
* With `-mno-abicalls`, `TARGET_ABICALLS_PIC2` is false (non-PIC code)
* Rationale for no pic0 optimization: building a 64-bit address requires 6 instructions, while GOT access is just 2-3 instructions

### `-mabicalls` vs `-mno-abicalls`

`-mabicalls` is the default for Linux targets.

`__PIC__` is defined only when `TARGET_ABICALLS_PIC2` is true (see `flag_pic = 1` in mips.cc):

**n64 with `-mabicalls`** (always PIC2, `__PIC__` defined, LOADGP_NEWABI):

```asm
// GP setup via %gp_rel
lui     $28,%hi(%neg(%gp_rel(test)))
daddu   $28,$28,$25
daddiu  $28,$28,%lo(%neg(%gp_rel(test)))
// Hidden function call: %got_disp
ld      $25,%got_disp(hidden_func)($28)
jalr    $25
// Extern function call: %call16
ld      $25,%call16(ext_func)($28)
jalr    $25
// Hidden/extern variable access: %got_disp
ld      $2,%got_disp(hidden_var)($28)
lw      $3,0($2)
ld      $2,%got_disp(ext_var)($28)
lw      $2,0($2)
```

**o32 with `-mabicalls -fpic`** (PIC2, `__PIC__` defined, LOADGP_OLDABI):
```asm
// GP setup via .cpload pseudo-op
.cpload $25
// Hidden function call: %got
lw      $25,%got(hidden_func)($28)
jalr    $25
// Extern function call: %call16
lw      $25,%call16(ext_func)($28)
jalr    $25
// Hidden/extern variable access: %got
lw      $2,%got(hidden_var)($28)
lw      $3,0($2)
lw      $2,%got(ext_var)($28)
lw      $2,0($2)
```

**n32 with `-mabicalls -fpic`** (PIC2, `__PIC__` defined, LOADGP_NEWABI):
```asm
// GP setup via %gp_rel
lui     $28,%hi(%neg(%gp_rel(test)))
addu    $28,$28,$25
addiu   $28,$28,%lo(%neg(%gp_rel(test)))
// Hidden function call: %got_disp (no lazy binding)
lw      $25,%got_disp(hidden_func)($28)
jalr    $25
// Extern function call: %call16 (lazy binding)
lw      $25,%call16(ext_func)($28)
jalr    $25
// Hidden/extern variable access: %got_disp
lw      $2,%got_disp(hidden_var)($28)
lw      $3,0($2)
lw      $2,%got_disp(ext_var)($28)
lw      $2,0($2)
```

**o32/n32 with `-mabicalls -fno-pic`** (default) or **`-mno-abicalls`** (`__PIC__` NOT defined):
```asm
// Hidden/extern function call: direct
jal     hidden_func
jal     ext_func
// Hidden/extern variable access: absolute
lui     $2,%hi(hidden_var)
lw      $3,%lo(hidden_var)($2)
lui     $2,%hi(ext_var)
lw      $2,%lo(ext_var)($2)
```

Both generate identical code. The difference is that `-mabicalls -fno-pic` emits `.abicalls` and `.option pic0` directives (for linker LA25 thunk generation when called from PIC code), while `-mno-abicalls` omits these directives entirely.

**n64 with `-mno-abicalls`** (`__PIC__` NOT defined):
```asm
// Hidden/extern function call: direct
jal     hidden_func
jal     ext_func
// Hidden/extern variable access: absolute (64-bit address, 6 instructions each)
lui     $2,%highest(hidden_var)
daddiu  $2,$2,%higher(hidden_var)
lui     $3,%hi(hidden_var)
dsll    $2,$2,32
daddu   $2,$2,$3
lw      $3,%lo(hidden_var)($2)
```

GCC errors if you specify `-fpic -mno-abicalls`.

## `-mshared`/`-mno-shared` code generation

As explained above, these options only affect o32/n32 (not n64).
With `-mno-shared`, the assembler is invoked with `-call_nonpic`; with `-mshared`, it uses `-KPIC`.

## `-mlong-calls`

GCC's mips port [added](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=d1399bd0ff3893bb9ebea7b977c7f3ec91b728b0) `-mlong-calls` in 1993-03.
This option is only useful with `-mno-abicalls`.

The `jal`/`j` instructions can only reach targets within a 256MB region (26-bit offset shifted left 2 bits, upper 4 bits of PC preserved).
`-mlong-calls` generates code that builds the full address in a register:

```asm
// Without -mlong-calls (direct call, limited range)
jal  external_func

// With -mlong-calls on n64 (full 64-bit address)
lui     $2, %highest(external_func)   // bits 63-48
lui     $3, %hi(external_func)        // bits 31-16
daddiu  $2, $2, %higher(external_func) // bits 47-32
daddiu  $3, $3, %lo(external_func)    // bits 15-0
dsll    $2, $2, 32
daddu   $2, $2, $3
jalr    $2
```

When `-mabicalls` is enabled, `-mlong-calls` has no effect because calls already go through the GOT, which naturally supports the full address range.

### GP initialization styles

GCC uses different GP initialization depending on `mips_current_loadgp_style()`:

**`LOADGP_OLDABI`** (o32 with `-mshared` or `-mno-explicit-relocs`):
Uses `.cpload $25` pseudo-op:
```asm
.cpload $25
```
Which the assembler expands to:
```asm
lui     $gp,%hi(_gp_disp)
addiu   $gp,$gp,%lo(_gp_disp)
addu    $gp,$gp,$25
```

**`LOADGP_NEWABI`** (n32/n64 with `-mshared`, or n64 with `-mabicalls`):
Uses `%gp_rel` to compute GP from the function address in `$25`:
```asm
lui     $28,%hi(%neg(%gp_rel(func)))
addu    $28,$28,$25
addiu   $28,$28,%lo(%neg(%gp_rel(func)))
```

**`LOADGP_ABSOLUTE`** (o32/n32 with `-mabicalls -mno-shared -mexplicit-relocs`):
When `TARGET_ABSOLUTE_ABICALLS` is true and the function needs GP (e.g., for GOT-based TLS access), uses `__gnu_local_gp`:
```c
// mips64el-linux-gnuabi64-gcc-14 -mabi=32 -fno-pic -S -O2
extern __thread int extern_tls_var;
int test(void) { return extern_tls_var; }
```
```asm
lui     $28,%hi(__gnu_local_gp)
addiu   $28,$28,%lo(__gnu_local_gp)
lw      $2,%gottprel(extern_tls_var)($28)
```

`__gnu_local_gp` is defined by the linker. Most functions in this mode don't need GP since they use absolute `%hi/%lo` addressing for regular symbols.

## `-mplt` for o32/n32 non-PIC

MIPS did not have PLT support in the linker until 2008.
Before that, for default visibility external functions, GCC generated GOT-based call sequences (like x86 `-fno-plt`).

Since 2008, `-mplt` is **enabled by default** for `-fno-pic` code ([RFC: Adding non-PIC executable support to MIPS](https://gcc.gnu.org/legacy-ml/gcc/2008-06/msg00670.html)).
This generates direct `jal` calls for all functions. If the callee ends up defined in a shared object, a PLT entry will be needed.

`-fno-pic` (equivalently `-fno-pic -mplt`):
```asm
// Hidden/extern function: direct call
jal     hidden_func
jal     ext_func
// Hidden/extern variable: absolute
lui     $2,%hi(hidden_var)
lw      $3,%lo(hidden_var)($2)
lui     $2,%hi(ext_var)
lw      $2,%lo(ext_var)($2)
```

`-mno-plt` disables this and creates a hybrid mode where hidden symbols use direct addressing while extern symbols use GOT-based access:

`-fno-pic -mno-plt`:
```asm
// GP setup via __gnu_local_gp
lui     $28,%hi(__gnu_local_gp)
addiu   $28,$28,%lo(__gnu_local_gp)
// Hidden function: direct call
jal     hidden_func
// Extern function: via GOT
lw      $25,%call16(ext_func)($28)
jalr    $25
// Hidden variable: absolute
lui     $2,%hi(hidden_var)
lw      $3,%lo(hidden_var)($2)
// Extern variable: via GOT
lw      $2,%got(ext_var)($28)
lw      $2,0($2)
```

`-mplt` also works with n64 `-msym32`.

Technically the optimization applies to `-fpie` as well, but GCC does not implement it because there is PIC PLT entry.
The existing PLT entry utilizes absolute relocation types: `lui $15, %hi(.got.plt entry); l[wd] $25, %lo(.got.plt entry)($15)`

Unfortunately, `-mplt` conflates function calls and variable accesses. When `-mplt` is in effect, GCC generates an absolute relocation accessing external data, which may lead to copy relocations if the data ends up defined by a shared object.

## `-mno-explicit-relocs`

Don't use assembler relocation operators such as `%hi`, `%lo`, `%call16`, `%got_disp`.
Instead, use assembler macros.

`-mexplicit-relocs` (default) facilitates instruction reordering.

## `-msym32` for n64

Assume that all symbols have 32-bit values, regardless of the selected ABI.
This is similar to `-mcmodel=small` for other architectures.

When `-msym32` is used with the n64 ABI, the `-mplt` optimization will apply.

## Special sections

There are many special sections. Let me just quote a comment from `binutils/readelf.c:process_mips_specific`:
```c
  /* We have a lot of special sections.  Thanks SGI!  */
```

There are many sections from other vendors. Anyway, this remark sets up the mood.

### `.reginfo` section

GNU assembler creates this section to hold `Elf32_RegInfo` object for o32 and n32 ABIs.

```c
/* A section of type SHT_MIPS_REGINFO contains the following
   structure.  */
typedef struct
{
  /* Mask of general purpose registers used.  */
  uint32_t ri_gprmask;
  /* Mask of co-processor registers used.  */
  uint32_t ri_cprmask[4];
  /* GP register value for this object file.  */
  uint32_t ri_gp_value;
} Elf32_RegInfo;
```

### `.MIPS.options` section

The n64 ABI uses `.MIPS.options` instead of `.reginfo`.

### `.MIPS.abiflags` section

In 2014, [[MIPS] Implement O32 FPXX, FP64 and FP64A ABI extensions](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=351cdf24d223290b15fa991e5052ec9e9bd1e284) introduced `.MIPS.abiflags` (type `SHT_MIPS_ABIFLAGS`) and `PT_MIPS_ABIFLAGS`.

```
A new implicitly generated section will be present on all new modules. The section contains a versioned data structure which represents essential information to allow a program loader to determine the requirements of the application. ELF e_flags currently contain some of this information but space is limited and e_flags are not available after loading an application.

The structure is versioned to allow for future extensions. The initial version is 0.

Register size fields record the maximum size register required for each class of registers
The floating point ABI is recorded using the same attribute values used for gnu_attribute 4. These are listed in Appendix B.
ASEs are represented by a bitmask of ASEs that have been used.
ISA extensions are represented as a single enumeration value.
The flags1 field includes a flag to record whether odd-numbered single-precision registers were enabled.
```

GNU ld has quite involved merging strategy for this section.

## `.option pic0` and `.option pic2` directives

These assembler directives control whether position-independent or position-dependent code is generated.

* `.option pic0` - Generate position-dependent (non-PIC) code. Enables direct `jal`/`j` instructions and absolute addressing.
* `.option pic2` - Generate position-independent code using the GOT.

GCC emits `.option pic0` for o32/n32 when `-mno-shared` is in effect (the default).
n64 does not use `.option pic0` - it always generates full PIC-style code with `-mabicalls`.

The assembler uses these directives to determine which relocation types to emit and whether certain optimizations are valid.
When compiling with `-mno-shared`, the assembler is also invoked with `-call_nonpic`; with `-mshared`, it uses `-KPIC`.

## `.set noreorder` directive

By default the assembler does smart multi-instruction rewriting including:

* filling a branch or delay slot automatically by reordering the instructions around it.
* inserting nops to avoid data hazards
* working around errata.

The `.set noreorder` directive disables the behavior.

## Relocations

<https://web.archive.org/web/20140908233719/https://dmz-portal.mips.com/wiki/MIPS_relocation_types>

While all ABIs use REL for dynamic relocations, the o32 ABI also uses REL for relocatable files while n64 uses RELA.
The little-endian n64 ABI messed up the `r_info` field in relocations.

The n64 ABI can pack up to 3 relocations with the same offset into one record (`r_type, r_type2, r_type3`).
This is primarily for $gp setup `R_MIPS_GPREL16 + R_MIPS_SUB + R_MIPS_{HI,LO}16`.

### Different calculation when the referenced symbol is local/external

Certain relocation types, such as `R_MIPS_GOT16`/`R_MIPS_GPREL16`, have different calculation when the referenced symbol is local or external.
This is a bad design.

When referencing a local symbol, the n32/n64 ABIs replace a `R_MIPS_GOT16`/`R_MIPS_LO16` pair with a `R_MIPS_GOT_PAGE`/`R_MIPS_GOT_OFST` pair.

### `R_MIPS_PC64`

This relocation type does not exist, but we can emulate a 64-bit PC-relative relocation with `R_MIPS_PC32 + R_MIPS_64 + R_MIPS_NONE`.
LLVM implemented this relocation in <https://reviews.llvm.org/D80390> while binutils [doesn't support it yet](https://sourceware.org/bugzilla/show_bug.cgi?id=26222).

```asm
.globl foo
foo:
.data
.quad foo-.
// R_MIPS_PC32 + R_MIPS_64 + R_MIPS_NONE
```

### Paired relocations

> In a .rel section, a pair of adjacent relocations, one a hi16 and the other a lo16, each provide a 16-bit partial addend. The hi16 halfword is shifted left 16 bits, the lo16 halfword is sign extended, and the two resulting values are added. (The two relocations need not actually be adjacent in a .rel section -- a single hi16 addend may be used with multiple lo16 addends -- but processing this combination requires fallable heuristics, so these relocations should not be used in .rel sections.)

```
R_MIPS_HI16 , R_MIPS_LO16 
R_MIPS_GOT16 (if the symbol is non-local), R_MIPS_LO16 
R_MIPS_HI16, R_MIPS_PCLO16
```

## Distributions

Here is an incomplete list.

* <https://www.debian.org/ports/mips/>
* <https://fedoraproject.org/wiki/Architectures>
* <https://wiki.gentoo.org/wiki/Project:MIPS>
* <https://wiki.netbsd.org/ports/>
* <https://www.openbsd.org/plat.html>
* <https://openwrt.org/docs/techref/targets/start>

FreeBSD 14 discontinued MIPS.
MIPS code was removed [starting since 2021-12](https://github.com/freebsd/freebsd-src/commit/b8cacb38989a9abe26348052fc477319f26c52cb).

## Misc

The entry point for an executable or shared object is `__start` instead of `_start`.

`E_MIPS_*` macros are [considered deprecated](https://sourceware.org/pipermail/binutils/2023-August/128993.html).

DWARF `.debug_*` sections have the `SHT_MIPS_DWARF` flag.

`%` modifiers (like unary operators) can nest, which makes parsing difficult.
LLVM integrated assembler parses these operators in the generic code (<https://reviews.llvm.org/D23110>), which is not good taste.
```asm
lui   $gp, %hi(%neg(%gp_rel(foo+4)))
```

The following script demonstrate that we can use a GCC cross compiler configured with 32-bit MIPS to build 64-bit object files.

```sh
#!/bin/zsh
compile() {
  ${@:2} a.c -O2 -c -o $1.o
  ${@:2} a.c -O2 -S -o $1.s
}

a() {
  compile o32$1 mipsel-linux-gnu-gcc -march=mips32r5 ${@:2}
  compile n64$1 mipsel-linux-gnu-gcc -march=mips64r5 -mabi=64 ${@:2}
  compile n32$1 mipsel-linux-gnu-gcc -march=mips64r5 -mabi=n32 ${@:2}
}

a ""
a "-plt" -mplt
a "-noplt" -mno-plt

a "-nopic" -fno-pic
a "-nopic-plt" -fno-pic -mplt
a "-nopic-noplt" -fno-pic -mno-plt

a "-nopic-mshared" -fno-pic -mshared
```
