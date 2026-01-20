---
layout: post
title: Relative relocations and RELR
author: MaskRay
tags: [binutils,glibc,linker]
---

Updated in 2024-02.

(In celebration of my 2800th llvm-project commit) Happy Halloween!

This article describes relative relocations and how the RELR format can greatly decrease file sizes.

An ELF linker performs the following steps to process an absolute relocation type whose width equals the word size (e.g. `R_AARCH64_ABS64`, `R_X86_64_64`).

```c
if (undefined_weak || (!preemptible && (no_pie || is_shn_abs)))
  link-time constant
else if (SHF_WRITE || znotext) {
  if (preemptible)
    emit a symbolic relocation (e.g. R_X86_64_64)
  else
    emit a relative relocation (e.g. R_X86_64_RELATIVE)
} else if (!shared && (copy_relocation || canonical_plt_entry)) {
  ...
} else {
  error
}
```

Note: in [FDPIC ABIs](/blog/2024-02-20-mmu-less-systems-and-fdpic), there is no single base address.
Relative relocations can still be used, but the runtime operation is not simply `*loc += base`.

<!-- more -->

In `-pie` or `-shared` mode, the linker produces a relative relocation (`R_*_RELATIVE`) if the symbol is non-preemptible.
The dynamic relocation is called a relative relocation.
`-no-pie` mode does not produce relative relocations.

```asm
.section .meta.foo,"a",@progbits
.quad .text.foo-.    # link-time constant

# Without 'w', text relocation.
.section .meta.foo,"aw",@progbits
.quad .text.foo      # R_*_RELATIVE dynamic relocation if -pie or -shared
```

Another source of `R_*_RELATIVE` relocations are GOT entries.
See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table).
```c
if (preemptible)
  emit an R_*_GLOB_DAT  // the third case
else if (!pic || is_shn_abs)
  link-time constant  // the first case
else
  emit a relative relocation  // the second case
```

The linker can produce an `R_*_RELATIVE` relocation in some other circumstances, but they are rare, e.g. the unfortunate `R_386_GOT32X/R_386_TLS_IE`, PowerPC64's position-independent long branch thunks.

Some architectures have `R_*_RELATIVE` variants.
On x86-32 (ILP32), `R_X86_64_RELATIVE` applies to a word32 location while `R_X86_64_RELATIVE64` applies to a word64 location.
Itanium seems to have multiple relative relocation types for different endianness.

On AArch64, when PAuth is enabled, `R_AARCH64_AUTH_RELATIVE` may be produced instead of `R_AARCH64_RELATIVE`.

## Representation

ELF has two relocation formats, REL and RELA.
64-bit capability and RELA are improvements on [previous formats used by a.out and COFF](/blog/2024-01-14-exploring-object-file-formats).

```c
typedef struct {
  Elf64_Addr    r_offset;   // Address
  Elf64_Xword   r_info;	    // 32-bit relocation type; 32-bit symbol index
} Elf64_Rel;

typedef struct {
  Elf64_Addr    r_offset;   // Address
  Elf64_Xword   r_info;	    // 32-bit relocation type; 32-bit symbol index
  Elf64_Sxword  r_addend;   // Addend
} Elf64_Rela;
```

RELA is nice for static relocations on RISC architectures but is very size inefficient for dynamic relocations.
REL is 33% more efficient but is still bloated when encoding relative relocations.
For a relative relocation, the symbol index is 0, but we have to pay a word for `r_info`.

The dynamic loader performs `*(Elf_Addr*)(base+r_offset) += base + addend;`.
For the REL format (used by arm and x86-32), `addend` is an implicit addend read from the to-be-relocated location.
For the RELA format (used by most architectures), `addend` is `r_addend` stored in the relocation record.

Relative relocations have great locality. On a 64-bit platform, it's common to have a consecutive sequence of locations which are all relocated, e.g.

```c
{r_offset = 0x2000, r_info = (R_X86_64_RELATIVE << 32) | 0},
{r_offset = 0x2008, r_info = (R_X86_64_RELATIVE << 32) | 0},
{r_offset = 0x2010, r_info = (R_X86_64_RELATIVE << 32) | 0},
{r_offset = 0x2018, r_info = (R_X86_64_RELATIVE << 32) | 0},
```

## Sources of relative relocations

Before we jump into compressing relative relocations, let's think of the sources of relative relocations and whether we can decrease their number.

In a position-independent executable, `R_*_RELATIVE` relocations typically dominate: they easily take 90% of the total dynamic relocations.
The ratio is larger if there are fewer `R_*_JUMP_SLOT` relocations.
For a mostly statically linked position-independent executable, the ratio can be as large as 99%.

A symbolic shared object has a similar distribution of dynamic relocations.
But non-symbolic shared objects are much more common and `R_*_RELATIVE` relocations usually take a very small portion of their dynamic relocations.
(See [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic) about symbol interposition.)

```c
// R_*_RELATIVE if non-preemptible
void *ptr = &foo;

// R_*_RELATIVE (guaranteed non-preemptible)
static void *ptr = &foo;

// 2 R_*_RELATIVE (guaranteed non-preemptible)
const char *str[] = {"a", "b"};
```

For string literal arrays, if setting a capacity wastes more space (`const char a[][CAP] = {"a", "b", ...};`), then there is no elegant way in C.
If you are not concerned of making the code less readable, you can leverage the trick from musl's `strerror` implementation:

```c
// __strerror.h
E(0,            "No error information")

E(EILSEQ,       "Illegal byte sequence")
E(EDOM,         "Domain error")

// a.c
#include <errno.h>
#include <stddef.h>

static const struct errmsgstr_t {
#define E(n, s) char str##n[sizeof(s)];
#include "__strerror.h"
#undef E
} errmsgstr = {
#define E(n, s) s,
#include "__strerror.h"
#undef E
};

static const unsigned short errmsgidx[] = {
#define E(n, s) [n] = offsetof(struct errmsgstr_t, str##n),
#include "__strerror.h"
#undef E
};

const char *foo(int i) { return (const char *)&errmsgstr + errmsgidx[i]; }
```

For C++, virtual tables can contribute many `R_*_RELATIVE` relocations through function pointers.
Fuchsia folks contributed `-fexperimental-relative-c++-abi-vtables` to Clang which is also available on Linux.
(underneath `i64 sub (lhs, rhs)` in a LLVM IR constant expression uses a PC-relative relocation.)
This can make a large portion of the memory image read-only and save a lot of space (32-bit PC-relative offsets instead of 64-bit absolute addresses), but is difficult to deploy for Linux in practice because of the ABI change.

Some programming languages (e.g. Jai, Odin) provide relative pointers. If the construct can be used with global objects, we can avoid many relative relocations.

For some modern architectures (AArch64, RISC-V, x86-64), PIC does not have a size penalty on text sections compared to non-PIC.
The number of `R_*_RELATIVE` relocations is the most significant source of code size bloat when changing from non-PIC to PIC.

I believe that for many applications relative relocations can be decreased but that may come with costs to readability or ABI issues.
Sometimes you can't have it both ways.

## Compressing REL/RELA relocations

One intuitive idea is to omit `r_type`. We will need a new section but the size has been cut in half compared to REL.

```asm
.section naive,"a"
  .quad 0x2000
  .quad 0x2008
  .quad 0x2010
  .quad 0x2018
```

Next, we can think of delta encoding and narrower entries.
But note that ELF tries to avoid unaligned/packed structures.
In addition, delta encoding, if not designed carefully, can make medium/large code models hard.

For a new format, the minimum is linker support and dynamic loader support.
Debuggers and binary manipulation tools may need to understand the format as well.

---

Over the years, there have been multiple attempts compressing the ELF relocation formats.

In 2010, Mike Hommey added "elfhack" to Firefox (<https://bugzilla.mozilla.org/show_bug.cgi?id=606145>).
[Improving libxul startup I/O by hacking the ELF format](https://glandium.org/blog/?p=1177#relocations) is a write-up.
It appeared to move most relative relocations from `.rel.dyn/.rela.dyn` into a custom section.
The section basically has multiple pairs of a base offset and a count of subsequent consecutive relocations.
The savings are quite significant.

In 2015, Android bionic got `DT_ANDROID_REL`/`DT_ANDROID_RELA`.
This is somewhat over-engineered but does not optimize relative relocations well.

## RELR relative relocation format

In 2017, Cary Coutant proposed a prototype of the RELR relative relocation format <https://sourceware.org/legacy-ml/gnu-gabi/2017-q2/msg00003.html>.
Rahul Chaudhry started a thread on the generic-abi forum: [Proposal for a new section type SHT\_RELR](https://groups.google.com/g/generic-abi/c/bX460iggiKg).
Ali Bahrami refined it to the format we have today: an even entry indicates an address while an odd entry indicates a bitmap.

In 2018, Rahul Chaudhry added [ld.lld](https://reviews.llvm.org/D48247)/[llvm-readelf](https://reviews.llvm.org/D47919) support for RELR and added patches in Chrome OS's binutils-gdb and glibc repositories.

Android and Fuchsia followed up and adopted `DT_RELR`.
These operating systems use `ld.lld --pack-dyn-relocs=relr` and a `DT_RELR` capable loader.

In 2019, I fixed an [ld.lld bug](https://reviews.llvm.org/D67164) which could cause the size of `.relr.dyn` to oscillate between 2 numbers.

In 2021-10, I added `DT_RELR` support to FreeBSD [`libexec/rtld-elf`](https://reviews.freebsd.org/D32524).
kib@ reviewed/commited it and made it available to the releng/13.1 branch.

In 2021, I attempted to bring the format to the GNU toolchain community.
I reimplemented some binutils-gdb and glibc patches and managed to submit [the GNU readelf change](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=aa7fd11862703e45d2774981a4888bc127d473b06) to the official repository in November and [the BFD change](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=a619b58721f0a03fd91c27670d3e4c2fb0d88f1e) was submitted in December 2021.
The BFD change is used by gdb and objcopy.
[gdb before 11.2](https://sourceware.org/bugzilla/show_bug.cgi?id=28758) does not include the change and gives diagnostics for RELR binaries: ``BFD: $file: unknown type [0x13] section `.relr.dyn'``. This is not cosmetic: it seems to affect debuggability as well.

On 2022-01-11, HJ Lu implemented `-z pack-relative-relocs` for GNU ld's [x86 port](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=5af6f000d88622107e7382d337af2884fd211da2). See [{pack-relative-relocs}](#pack-relative-relocs) below.
Alan Modra implemented `-z pack-relative-relocs` for GNU ld's [PowerPC64 port](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=1657026ccd8aa05a97cb35f1d9bff646937a244e).
Both changes are released in binutils 2.38.

We have Cary Coutant's agreement that after he converts generic-abi to Markdown and makes it open, RELR will be applied.

In 2022-04, `DT_RELR` changes finally landed in glibc (milestone: 2.36). The main ones are [Add GLIBC_ABI_DT_RELR for DT_RELR support](https://sourceware.org/git/?p=glibc.git;a=commit;h=57292f574156f817b7cbeb33ea6278c6eab22bcc) and [elf: Support DT_RELR relative relocation format [BZ #27924]](https://sourceware.org/git/?p=glibc.git;a=commit;h=e895cff59aa562cad83fa0fdd187bfe4b45312d5).
(The change failed to make it to 2.35, but this was not too bad. See [Can DT\_RELR catch up glibc 2.35?](https://sourceware.org/pipermail/libc-alpha/2021-November/133009.html))

In 2022-08, my musl change was merged: [ldso: support DT_RELR relative relocation format](https://git.musl-libc.org/cgit/musl/commit/?id=d32dadd60efb9d3b255351a3b532f8e4c3dd0db1).
This was a retry after my upstream attempt in 2019-03 <https://www.openwall.com/lists/musl/2019/03/06/3> (at that time, the BDFL said RELR was a nice improvement but not a critical change blocking anything, so the fate of the feature was unclear).

In 2023-08, Mark Wielaard added RELR support to elfutils.

In 2024-05, Szabolcs Nagy implemented `-z pack-relative-relocs` for GNU ld's aarch64 port.

In 2024-07, Xi Ruoyao implemented `-z pack-relative-relocs` for GNU ld's loongarch port.

### (Pre)standard wording

From [Proposal for a new section type SHT_RELR](https://groups.google.com/g/generic-abi/c/bX460iggiKg)

```c
typedef Elf32_Word Elf32_Relr;
typedef Elf64_Xword Elf64_Relr;
```

> Description
>
> SHT_RELR: The section holds an array of relocation entries, used to encode relative relocations that do not require explicit addends or other information. Array elements are of type Elf32_Relr for ELFCLASS32 objects, and Elf64_Relr for ELFCLASS64 objects. SHT_RELR sections are for dynamic linking, and may only appear in object files of type ET_EXEC or ET_DYN. An object file may have multiple relocation sections. See ``Relocation'' below for details.
>
> [...]

The format is best described by code.
When `--pack-dyn-relocs=relr` is specified (the feature is enabled), ld.lld creates `.relr.dyn` (of type `SHT_RELR`) which holds an array of relocation entries, used to encode relative relocations that do not require explicit addends.
Regular `R_*_RELATIVE` from `.rel.dyn/.rela.dyn` are removed.

In the `.relr.dyn` section,

* An even entry indicates a location which needs a relocation and sets up `where` for subsequent odd entries.
* An odd entry indicates a bitmap encoding up to 63 locations following `where`.
* Odd entries can be chained.

```c
relrlim = (const Elf_Relr *)((const char *)obj->relr + obj->relrsize);
for (relr = obj->relr; relr < relrlim; relr++) {
  Elf_Relr entry = *relr;
  if ((entry & 1) == 0) {
    where = (Elf_Addr *)(obj->relocbase + entry);
    *where++ += (Elf_Addr)obj->relocbase;
  } else {
    for (long i = 0; (entry >>= 1) != 0; i++)
      if ((entry & 1) != 0)
        where[i] += (Elf_Addr)obj->relocbase;
    where += CHAR_BIT * sizeof(Elf_Relr) - 1;
  }
}
```

RELR can typically encode the same information in `.rela.dyn` in less than 3% space.

```sh
q() { file $1 | grep -q ELF || return; local s=$(stat -c %s $1); printf "$1\t$s\t"; bc <<< "scale=2; $(readelf -Wr $1 | grep -c _RELATIVE)*24 * 100 / $s" | sed 's/^\./0./' }

for i in /usr/bin/*(.x); do q $i; done > /tmp/0
awk '{c+=$2*$3/100; s+=$2} END {print s, " ", c/s}' /tmp/0
```

On my Arch Linux, among `/usr/bin/*` executables, relative relocations take 7.9% of the total file size.
These large executables spend 10+% in the `.rela.dyn` section: `as`, `audacity`, `fzf`, `ndisarm`, `objdump`, `ocaml*`, `perf_*`, `qemu-system-*`, `pdftosrc`, `php*`, `strace`, `virt-*`.
Among `/usr/lib/lib*.so*` shared objects, relative relocations take 4.92% of the total file size.

I get similar savings on my Debian Linux.

### Time travel compatibility

Programs with new features run the risk of mysterious crashes or malfunction on old systems.
Ali Bahrami calls new objects on old systems "time travel compatibility".
For Solaris: "And yet, our experience is that although we don't go to great effort to catch time traveling objects, they very rarely cause problems for us. People seem to understand that they can't do that."

For `DT_RELR` objects, they will immediately segfault on an old glibc.
For some groups (ChromeOS, FreeBSD, Fuchsia, NetBSD, and Solaris), they just don't support "time travel compatibility" development model and can accept that new objects may have poor diagnostics.

However, the glibc community tends to be more cautious about a nice diagnostic.
Note: such "time travel compatibility" mechanism is not consistently applied. See [When to prevent execution of new binaries with old glibc](https://sourceware.org/pipermail/libc-alpha/2021-October/132168.html) for a summary.

Back in December 2021, my idea was that such a diagnostic should not be a blocker for the `DT_RELR` feature.

* We did not flip the default for GCC/Clang.
* GNU ld did not support RELR yet. For most Linux distributions, there was no impact to their packages.
* The users who opted in the feature accepted the poor diagnostic if they back ported that to older systems.
* If we merged the glibc patch early, we would make the future flip less of a problem.

I wished that the glibc community could listen to what [Ali suggested](https://groups.google.com/g/generic-abi/c/bX460iggiKg/m/0PMCJ0hjBAAJ) in the generic-abi post (January 2018):

> My free advice (worth what you paid for it) is to roll out the support, and then wait a bit before turning on the use widely, so that the support is in place before it is needed, and to not complicate things with a way to catch time travelers. The window of time where this can be a problem is finite, and once you're past it, you'll be glad to have a simpler system.

This advice applies to many things, including a GCC 5 regression for x86-64 `-fpie` I am trying to fix (but still pending) [`[PATCH] x86-64: Remove HAVE_LD_PIE_COPYRELOC`](https://gcc.gnu.org/pipermail/gcc-patches/2021-September/579949.html) and [copy relocations on protected data](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected#summary).

Anyway, since there is interest on time travel compatibility, let's discuss potential solutions.

#### `EI_ABIVERSION`

Some might ask whether we could use `EI_ABIVERSION`.

The System V ABI says:

> Byte e_ident[EI_ABIVERSION] identifies the version of the ABI to which the object is targeted. This field is used to distinguish among incompatible versions of an ABI. The interpretation of this version number is dependent on the ABI identified by the EI_OSABI field. If no values are specified for the EI_OSABI field by the processor supplement or no version values are specified for the ABI determined by a particular value of the EI_OSABI byte, the value 0 shall be used for the EI_ABIVERSION byte; it indicates unspecified.

`EI_ABIVERSION` is dependent on `EI_OSABI`. Operating systems decide their `EI_ABIVERSION`.
We need to discuss `ELFOSABI_NONE`/`ELFOSABI_GNU` separately.

For `ELFOSABI_GNU` (alias: `ELFOSABI_LINUX`), different architectures may have different `EI_ABIVERSION` values.
I know that mips may use `EI_ABIVERSION==1` for `(e_eflags & (EF_MIPS_PIC | EF_MIPS_CPIC)) == EF_MIPS_CPIC` position-dependent executables.
For glibc, we would need to bump `LIBC_ABI_MAX`.

`ELFOSABI_NONE` is the domain of the generic ABI.
Many Linux executables don't use `STB_GNU_UNIQUE`/`STT_GNU_IFUNC` and therefore use `ELFOSABI_NONE`.
Solaris folks have ruled out the possibility to bump `EI_OSABIVERSION` because this is not a development model they need.
I think Android bionic/Chrome OS/FreeBSD fall in the same boat.

Even if `EI_ABIVERSION` could be bumped: as of glibc 2.34, ld.so does not check `EI_ABIVERSION` for kernel mapped objects.
It does check ld.so mapped objects (including all `DT_NEEDED` shared objects), though.
```sh
% r2 -nwqc 'wx 01 @ 8' a  # Change EI_ABIVERSION to 1
% ./a
% /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./a
./a: error while loading shared libraries: ./a: ELF file ABI version invalid
```
For `ELFOSABI_GNU` (`r2 -nwqc 'wx 03 @ 7'`), changing `EI_ABIVERSION` to 4 or above will observe the failure with ld.so mapped objects but not kernel mapped objects.

In addition, bumping the ABI version can immediately lock out some ELF utilities which only deal with `e_ident[EI_ABIVERSION] == 0` objects.
For example, `elflint` from elfutils reports an error.

#### Synthesized undefined dynamic symbol

We could let the linker synthesizes an undefined symbol in the dynamic symbol table (`.dynsym`) to indicate the usage of `DT_RELR`, then let ld.so define the symbol to indicate feature availability.
(See [the `_dl_have_relr` idea by Michael Matz](https://sourceware.org/pipermail/binutils/2021-October/118347.html).)

In glibc, an unreferenced unversioned undefined dynamic symbol does not trigger an `undefined symbol` error, so the linker would need to synthesize a fake relocation.
`R_*_NONE` is typically ignored so we would need a relocation which does something, e.g. `R_X86_64_64`.
Then the linker has to synthesize a `.data.rel.ro` section to accommodate the relocation.

As an alternative, the linker can make the undefined dynamic symbol versioned, then a diagnostic will be reported even if the symbol is unreferenced.
A versioned undefined symbol can lead to a link-time error even for `-z undefs` (unfortunate default for `ld -shared`).

If GNU ld adds, say, `--pack-dyn-relocs=relr-glibc` or `-z relr-glibc`, with the functionality, I will probably just add a compatibility alias to ld.lld but not actually add the symbol (<https://sourceware.org/pipermail/libc-alpha/2021-October/132460.html>):

* Some users don't need "time travel compatibility".
* `--pack-dyn-relocs=relr` is quite popular in some platforms now. `--pack-dyn-relocs=relr` would still be usable to bypass the `relr-glibc` restriction on glibc.
* I don't want users to migrate away from `--pack-dyn-relocs=relr` (churn) just because glibc has a different development model.

#### The `DT_CRITICAL_*` proposal

See [Critical program headers and dynamic tags](https://groups.google.com/g/generic-abi/c/vdG_G4l3N-Y).

Solaris does not support the development model, so this proposal will unlikely enter generic ABI.

AFAIK this is not in the GNU ABI and glibc does not support this feature.
It is not suitable to do time travel compatibility with this proposal.

#### Symbol versioning

glibc 2.34 added `__libc_start_main@@GLIBC_2.34`.

A `*crt1.o` file references `__libc_start_main`.
A new program linking against new `libc.so` cannot run on glibc older than 2.34.

`*crt1.o` is for executables, but a similar trick can be used in `crti.o` or `crtbegin*.o` to catch both executables and shared objects.

This mechanism is coarse in that it does not check whether `DT_RELR` is used.

<a name="pack-relative-relocs"></a>
On 2022-01-11, HJ Lu added `-z pack-relative-relocs` to GNU ld.
If the linked output has a `GLIBC_2.*` version dependency on a shared object named `libc.so.*`, GNU ld will add an extra version dependency named `GLIBC_ABI_DT_RELR`.
In the glibc 2.36 `DT_RELR` implementation, glibc reports an error if a dynamically linked executable/shared object using `DT_RELR` does not have the `GLIBC_ABI_DT_RELR` version dependency.

### RELR adoption among Linux distributions

I have filed tickets for several large Linux distributions to bring the size saving to their attention.

* Arch Linux: <https://bugs.archlinux.org/task/72433>
* Debian: <https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=996598>
* Fedora: <https://bugzilla.redhat.com/show_bug.cgi?id=2014699>. Obsoleted by <https://bugzilla.redhat.com/show_bug.cgi?id=2218018>
* Gentoo: <https://bugs.gentoo.org/818376>

## CREL relocation format

In 2024, I proposed [CREL](/blog/2024-03-09-a-compact-relocation-format-for-elf) to optimize static relocations and non-relative dynamic relocations.

## Other object file formats

### Mach-O

Mach-O `__LINKEDIT,__rebase` serves a similar purpose as ELF `R_*_RELATIVE` relocations.
It uses a bytecode encoding to decrease space consumption.
The bytecode schemes uses run-length encoding and encodes the delta between two offsets to save bytes on encoding the offset.
Empirically I think it is inferior to ELF RELR in the majority of cases.

### PE/COFF

On Windows, [`.reloc`](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#base-relocation-block) contains all base relocations in the image.
This format is more efficient than ELF's REL/RELA but less efficient than RELR (when encoding relative relocations).
When the base relocation type is (32-bit architecture) `IMAGE_REL_BASED_HIGHLOW` or (64-bit architecture) `IMAGE_REL_BASED_DIR64`, it is like a relative relocation on ELF.
