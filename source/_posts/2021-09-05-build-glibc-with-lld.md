---
layout: post
title: Build glibc with LLD 13
author: MaskRay
tags: [binutils,glibc,linker]
---

LLD is the LLVM linker.
It started at the end of 2011 as a work-in-progress rewrite of ld64 for the Mach-O binary format based on the atom model.
COFF and ELF ports based on the atom model were contributed subsequently.
They shared one symbol resolution model.
(IMO due to Mach-O's unfortunate limitation of 255 section `.subsections_via_symbols` was invented. The atom model was an incarnation of the concept but it did not fit into ELF/PE where sections are the better basic units.)

In 2015, both COFF and ELF ports were rewritten. (See "LLD improvement plan")
Today, LLD is a mature and fast linker supporting multiple binary formats (ELF, Mach-O, PE/COFF, WebAssembly).
FreeBSD, Android, and Chrome OS have adopted it as the main linker.

As a main contributor of LLD's ELF port who has fixed numerous corner cases in recent years, I consider that its x86-64 support has been mature since the 8.0.0 release and is in a great shape since 9.0.0.
The AArch64 and PowerPC32/PowerPC64 support has been great since the 10.0.0 release.
The 11.0.0 release has very solid linker script support. (When people complain that GNU ld's linker script is not immediately usable with LLD, it is almost assuredly the problem of the script itself.)
So, what's next? Build glibc with LLD!

<!-- more -->

glibc is known for tricks used here and there and tons of GNU extensions which challenge a "foreign" toolchain like llvm-project (Clang, LLD, etc).
I just expected quirky linker usage which should be fixed on glibc's side, not anything to improve on LLD' side:)

My adventure concluded with the final toggle [configure: Allow LD to be LLD 13.0.0 or above [BZ #26558]](https://sourceware.org/git/?p=glibc.git;a=commit;h=224edada607ebc6aaa1aadaae423128fae7880df).
The next release glibc 2.35 should be buildable with LLD 13.0.0, with all tests passing on aarch64/i386/x86-64.
(I lied. It seems that you cannot assume all tests pass with GNU ld. On Debian and its derivatives, you may observe more failures than Fedora. Anyway, I just wanted to say LLD does not have more failures than GNU ld.)

Read on.

## Build

### `librtld.map`

There is a bootstrapping problem between ld.so and libc because they are separate.
In a nutshell, `elf/Makefile` performs the following steps to build `elf/ld.so`:

* Create `elf/libc_pic.a` from libc `.os` files
* Create `elf/dl-allobjs.os` from a relocatable link of rtld `.os` files
* Create link map `elf/librtld.map` from a relocatable link of `elf/dl-allobjs.os`, `elf/libc_pic.a`, and `-lgcc`
* Get a list of extracted archive members (`elf/librtld.mk`) from `elf/librtld.map` and create `elf/rtld-libc.a`
* Create `elf/librtld.os` from a relocatable link of `elf/dl-allobjs.os` and `elf/rtld-libc.a`
* Create `elf/ld.so` from a `-shared` link of `elf/librtld.os` with the version script `ld.map`

In a link map printed by GNU ld, `Archive member included to satisfy reference by file (symbol)` is followed by extracted archive members.
`elf/Makefile` made use of `sed -n 's@^$(common-objpfx)\([^(]*\)(\([^)]*\.os\)) *.*$$@\1 \2@p'` to extract the archive members.
ld.lld doesn't implement `Archive member included to satisfy reference by file (symbol)`.
Fortunately, ld.lld's output has lines like
```text
           1f350            1f350       1e    16         /home/maskray/Dev/glibc/out/lld/elf/rtld-libc.a(rtld-access.os):(.text)
           1f350            1f350       1e     1                 __access
           1f350            1f350       1e     1                 access
```

We can use `sed -n 's@^[0-9a-f ]*$(common-objpfx)\([^(]*\)(\([^)]*\.os\)) *.*$$@\1 \2@p'` to extract the archive members.

### `scripts/output-format.sed`

[BZ #26559](https://sourceware.org/bugzilla/show_bug.cgi?id=26559)

`libc.so` is a linker script. On Debian GNU/Linux, it looks like:

```sh
% cat /lib/x86_64-linux-gnu/libc.so
/* GNU ld script
   Use the shared library, but some functions are only in
   the static library, so try that secondarily.  */
OUTPUT_FORMAT(elf64-x86-64)
GROUP ( /lib/x86_64-linux-gnu/libc.so.6 /usr/lib/x86_64-linux-gnu/libc_nonshared.a  AS_NEEDED ( /lib64/ld-linux-x86-64.so.2 ) )
```

The idea is that `-lc` can expand to something like `-( libc.so.6 libc_nonshared.a --push-state --as-needed ld-linux-x86-64.so.2 --pop-state -)`.
`libc_nonshared.a` contains functions which should be statically linked.
`ld-linux-x86-64.so.2` is mostly for `__tls_get_addr` used by general-dynamic/local-dynamic TLS models.
Commit d3f5f87569398d11756b3dcb7a66926bfd8ee047 (in 2015) added `AS_NEEDED` with no description of the purpose.
Retroactively, this can make a ld.so [performance difference](https://reviews.llvm.org/D55902) when an executable has O(1000) shared object dependencies because the overall shared object uniqueness check has quadratic time complexity.

The first non-comment line is an `OUTPUT_FORMAT` command, which is derived from the output of `ld --verbose`.
In GNU ld, `--verbose` prints the internal linker script, which is used when an external one (`-T`) is not used.
```text
...
using internal linker script:
==================================================
/* Script for -z combreloc -z separate-code */
/* Copyright (C) 2014-2020 Free Software Foundation, Inc.
   Copying and distribution of this script, with or without modification,
   are permitted in any medium without royalty provided the copyright
   notice and this notice are preserved.  */
OUTPUT_FORMAT("elf64-x86-64", "elf64-x86-64",
              "elf64-x86-64")
OUTPUT_ARCH(i386:x86-64)
```

`Makerules` extracted the `OUTPUT_FORMAT` line with a frightening sed script:
```text
/ld.*[  ]-E[BL]/b f
/collect.*[     ]-E[BL]/b f
/OUTPUT_FORMAT[^)]*$/{N
s/\n[    ]*/ /
}
t o
: o
s/^.*OUTPUT_FORMAT(\([^,]*\), \1, \1).*$/OUTPUT_FORMAT(\1)/
t q
s/^.*OUTPUT_FORMAT(\([^,]*\), \([^,]*\), \([^,]*\)).*$/\1,\2,\3/
t s
s/^.*OUTPUT_FORMAT(\([^,)]*\).*$)/OUTPUT_FORMAT(\1)/
t q
d
: s
s/"//g
G
s/\n//
s/^\([^,]*\),\([^,]*\),\([^,]*\),B/OUTPUT_FORMAT(\2)/p
s/^\([^,]*\),\([^,]*\),\([^,]*\),L/OUTPUT_FORMAT(\3)/p
s/^\([^,]*\),\([^,]*\),\([^,]*\)/OUTPUT_FORMAT(\1)/p
/,/s|^|*** BUG in libc/scripts/output-format.sed *** |p
q
: q
s/"//g
p
q
: f
s/^.*[  ]-E\([BL]\)[    ].*$/,\1/
t h
s/^.*[  ]-E\([BL]\)$/,\1/
t h
d
: h
h
```

ld.lld does not have an internal linker script so `libc.so` did not have the `OUTPUT_FORMAT` line.
(
Personally I think an internal linker script is [not useful](https://bugs.llvm.org/show_bug.cgi?id=51309).
It would have some exposition value but the language is not powerful enough to encode all built-in logic.
If ld.lld is to support the feature, we would need to emit a lot of conditional code which can add a huge amount of maintenance burden.
)

Inspired by a [Linux kernel usage](https://github.com/ClangBuiltLinux/linux/issues/779), I realized that there is a better way to get the output format (bfdname): we can just parse the output of `objdump -f`.
```text
% objdump -f elf/ld.so

elf/ld.so:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000150:
HAS_SYMS, DYNAMIC, D_PAGED
start address 0x0000000000001050

% llvm-objdump -f elf/ld.so

elf/ld.so:      file format elf64-x86-64

architecture: x86_64
start address: 0x0000000000001050

```

`llvm-objdump -f` somewhat printed upper-case output formats. I switched the case in [D76046](https://reviews.llvm.org/D76046).

Fixed by [`install: Replace scripts/output-format.sed with objdump -f [BZ #26559]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=02d393f2483aedc4ce74e6edf8a15f063198041d).

### `--defsym` suppressing archive member extraction

`elf/Makefile` specified `-Wl,--defsym=malloc=0` and other `malloc.os` definitions before `libc_pic.a` so that `libc_pic.a(malloc.os)` is not extracted.
This trick was used to avoid multiple definition errors.

```makefile
# These symbols need to be stubbed out during symbol discovery because
# their implementation is provided differently in rtld, and the symbol
# discovery mechanism is not compatible with the libc implementation
# when compiled for libc.
rtld-stubbed-symbols = \
  __GI___pthread_disable_asynccancel \
  __GI___pthread_enable_asynccancel \
  __pthread_disable_asynccancel \
  __pthread_enable_asynccancel \
  calloc \
  free \
  malloc \
  realloc

...

# The GCC arguments that implement $(rtld-stubbed-symbols).
rtld-stubbed-symbols-args = \
  $(patsubst %,-Wl$(comma)--defsym=%=0, $(rtld-stubbed-symbols))

$(objpfx)librtld.map: $(objpfx)dl-allobjs.os $(common-objpfx)libc_pic.a
        @-rm -f $@T
        $(reloc-link) -o $@.o $(rtld-stubbed-symbols-args) \
                '-Wl,-(' $^ -lgcc '-Wl,-)' -Wl,-Map,$@T
        rm -f $@.o
```

For the interaction between a linker option and an input file, ld.lld generally chooses the behavior so that their relative order does not matter.
Some options are inherently order dependent, e.g. `--as-needed` and `--no-as-needed`, `--whole-archive` and `--no-whole-archive`.
However, reducing order dependence can improve robustness of a build system.

I had a debate with others and finally I noticed one point: `--defsym` defines a `SHN_ABS` symbol while a normal definition is relative to the image base.
So a normal definition is better regardless.

I sent the patch in April 2020, pinged once in August. Since nobody responded, I sent it again in December.
Finally, this issue is fixed by [`elf: Replace a --defsym trick with an object file to be compatible with ld.lld`](https://sourceware.org/git/?p=glibc.git;a=commit;h=02d393f2483aedc4ce74e6edf8a15f063198041d).

### `-static-pie` and `__rela_iplt_start`/`__rela_iplt_end`

[BZ #27164](https://sourceware.org/bugzilla/show_bug.cgi?id=27164)

glibc's static non-pie and static pie modes take different code paths for `R_*_IRELATIVE` relocation resolving.
Its static pie mode expects that `__rela_iplt_start==__rela_iplt_end`, which is satisfied if ld leaves the symbols zero.
Before 13.0.0, ld.lld defined `__rela_iplt_start`/`__rela_iplt_end` for `-pie` and therefore broke glibc loader's assumption, causing a program to crash.
See [GNU indirect function](/blog/2021-01-18-gnu-indirect-function) for details about the encapsulation symbols.

I made good arguments but were dismissed by "Ulrich and I designed/implemented IFUNC on x86 and for x86. I consider x86 implementation of IFUC as the gold standard."

In the end, I conceded and changed ld.lld to only define the two encapsulation symbols for `-no-pie` with grumbling.
Otherwise, `{gcc,clang} -fuse-ld=lld -static-pie` produced static pie would crash, even if the used glibc was built with GNU ld.

### `_GLOBAL_OFFSET_TABLE_[0]`

[BZ #28203](https://sourceware.org/bugzilla/show_bug.cgi?id=28203) for aarch64.

In nearly every ELF port of GNU ld, `_GLOBAL_OFFSET_TABLE_[0]` is the link-time address of `_DYNAMIC` (the start of `.dynamic`/`PT_DYNAMIC`).
In glibc, `sysdeps/*/dl-machine.h` files used this approach to compute the load base (the virtual address of the ELF header).

```text
runtime_DYNAMIC = PC relative address of _DYNAMIC
load_base = runtime_DYNAMIC - linktime_DYNAMIC = runtime_DYNAMIC - _GLOBAL_OFFSET_TABLE_[0]
```

So you may ask: why can't glibc extract the `p_vaddr` field of the `PT_DYNAMIC` program header.
Well, its code has a poor organization and makes this elegant solution difficult...

Due to the glibc requirement, unfortunately `_GLOBAL_OFFSET_TABLE_[0]` has been a part of i386/x86-64 and PowerPC64 ELFv2 ABIs.

ld.lld's AArch64 port does not set `_GLOBAL_OFFSET_TABLE_[0]`, so the trick does not work.
I figured out an elegant fix without updating ld.lld:

In 2012, GNU ld and gold (included in binutils 2.23) started to define `__ehdr_start` which has the link-time address zero.
Using a PC relative code sequence to take the runtime address of `__ehdr_start` gives us a better way to get the load base.
I submitted patches to use the approach for aarch64/arm/riscv/i386/x86-64.
The aarch64 code looks like the following.
I originally intended to use inline assembly to avoid relying on compiler generating PC-relative addressing for hidden symbol access, but Szabolcs Nagy recommended the pure C approach.

```c
/* Return the run-time load address of the shared object.  */

static inline ElfW(Addr) __attribute__ ((unused))
elf_machine_load_address (void)
{
  extern const ElfW(Ehdr) __ehdr_start attribute_hidden;
  return (ElfW(Addr)) &__ehdr_start;
}

/* Return the link-time address of _DYNAMIC.  */

static inline ElfW(Addr) __attribute__ ((unused))
elf_machine_dynamic (void)
{
  extern ElfW(Dyn) _DYNAMIC[] attribute_hidden;
  return (ElfW(Addr)) _DYNAMIC - elf_machine_load_address ();
}
```

I used this as an argument to mark [ld.lld PR49672](https://bugs.llvm.org/show_bug.cgi?id=49672) (reported by the glibc creator:)) as wontfix.
This also allowed the AArch64 ABI not to be polluted by glibc `_GLOBAL_OFFSET_TABLE_[0]`.

Fixed by [`aarch64: Make elf_machine_{load_address,dynamic} robust [BZ #28203]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=43d06ed218fc8be58987bdfd00e21e5720f0b862).

* [`riscv: Drop reliance on _GLOBAL_OFFSET_TABLE_[0]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=34b4624b04fc8f038b2c329ca7560197320615b4)
* [`x86_64: Simplify elf_machine_{load_address,dynamic}`](https://sourceware.org/git/?p=glibc.git;a=commit;h=b37b75d269883a2c553bb7019a813094eb4e2dd1)
* [`i386: Port elf_machine_{load_address,dynamic} from x86-64`](https://sourceware.org/git/?p=glibc.git;a=commit;h=https://sourceware.org/git/?p=glibc.git;a=commit;h=91e92272caefad4b6156572fc41671dcbd93afe5)

For added fun, consider the addresses of two hidden symbols, like `__ehdr_start` and `_end`.
GCC may generate a vector load using a constant pool that requires dynamic relocation.
If the vector load is reordered before `ELF_DYNAMIC_RELOCATE`, the loaded value will be incorrect.
To address this, glibc 2.42 introduced inline assembly compiler barriers (<https://sourceware.org/bugzilla/show_bug.cgi?id=33088>).

```asm
	movq	.LC0(%rip), %xmm0
...

	.section	.data.rel.ro.local,"aw"
	.align 8
.LC0:
	.quad	__ehdr_start
	.hidden	__ehdr_start
	.hidden	_end
```

### Non-default version symbols

If linked with ld.lld<13.0.0 or gold ([PR28196](https://sourceware.org/bugzilla/show_bug.cgi?id=28196)), the built ld.so will define both `__free_hook@GLIBC_2.2.5` and `__free_hook@@GLIBC_2.2.5`.
This is due to an unfortunate GNU as/GNU ld implementation flaw of symbol versioning which happens to be [relied upon by glibc](https://sourceware.org/bugzilla/show_bug.cgi?id=28197).

```text
GLIBC_2.2.5 {
  global:
    __free_hook;
    ...
  local:
    *;
};
```

Anyway, I conceded and implemented [`[ELF] Combine foo@v1 and foo with the same versionId if both are defined`](https://reviews.llvm.org/D107235) not to cause confusion for the unfortunate corner case.

See the discussion on defined non-default version symbols on [All about symbol versioning](/blog/2020-11-26-all-about-symbol-versioning).

In addition, I implemented [`[ELF] Apply version script patterns to non-default version symbols`](https://reviews.llvm.org/D107234) so that `local:` works correctly.

## Tests

With the aforementioned issues addressed, ld.lld linked glibc was fully functioning.
However, they did not accept allowing ld.lld in `configure.ac`.
They expected that all tests passed.
So I had to fix the following issues.

### `.tls_common`

[BZ #28152](https://sourceware.org/bugzilla/show_bug.cgi?id=28152)

The LLVM integrated assembler does not support the assembler directive.
ld.lld does not support `SHN_COMMON STT_TLS`.
The ELF standard common symbol uses `STT_COMMON`, so it would be incompatible with `STT_TLS`.

[`elf: Drop elf/tls-macros.h in favor of __thread and tls_model attributes [BZ #28152] [BZ #28205]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=33c50ef42878b07ee6ead8b3f1a81d8c2c74697c) fixed the issue.

### AArch64's general-dynamic/local-dynamic TLS models

[BZ #28205](https://sourceware.org/bugzilla/show_bug.cgi?id=28205)

AArch64 toolchains use TLS descriptors by default.
For AArch64, ld.lld supports TLD descriptor relocation types but not relocation types for general-dynamic/local-dynamic (`R_AARCH64_TLSGD_*/R_AARCH64_TLSLD_*`).
In addition, the LLVM integrated assembler doesn't support the general-dynamic/local-dynamic modifiers.
So the following tests did not build with Clang or ld.lld.

* `elf/tst-tls1.c`
* `elf/tst-tls2.c`
* `elf/tst-tls3.c`
* `elf/tst-tlsmod1.c`
* `elf/tst-tlsmod2.c`
* `elf/tst-tlsmod3.c`
* `elf/tst-tlsmod4.c`

[`elf: Drop elf/tls-macros.h in favor of __thread and tls_model attributes [BZ #28152] [BZ #28205]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=33c50ef42878b07ee6ead8b3f1a81d8c2c74697c) fixed the issue.

As a follow-up, I pushed [`Remove sysdeps/*/tls-macros.h`](https://sourceware.org/git/?p=glibc.git;a=commit;h=710ba420fd417a4a82e0ad2e998e5f3b972cb503).
These files may have educational values for future architectures about how to implement TLS models:)

<!--
## Complaints

<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=94391> is my go-to example when I want to demonstrate that less amicable to a foreign toolchain.
"NO ld.lld is not implemented the ABI as PIE COPYRELOC is required by ABI these days".
The status was updated back and forth between "invalid", "wontfix" and "worksforme" until a maintainer realized GNU ld had indeed one bug (R_X86_64_[REX_]GOTPCRELX cannot be relaxed for SHN_ABS) and one enhancement (PC relative relocations to a non-preemptible symbol should be rejected).
-->

### `--no-tls-get-addr-optimize`

GNU ld's PowerPC64 port has implemented a general-dynamic/local-dynamic TLS model optimization.
ld.lld doesn't support `--{,no-}tls-get-addr-optimize`.

[`powerpc: Use --no-tls-get-addr-optimize in test only if the linker supports it`](https://sourceware.org/git/?p=glibc.git;a=commit;h=9926f6e2eeb374cf729d4bb3f092dd4b36a8f861) skipped the test if necessary.

### `--audit` and `--depaudit`

[BZ #28151](https://sourceware.org/bugzilla/show_bug.cgi?id=28151)

gold and ld.lld do not support `--audit` or `--depaudit`.

[`elf: Skip tst-auditlogmod-* if the linker doesn't support --depaudit [BZ #28151]`](https://sourceware.org/git/?p=glibc.git;a=commit;h=9926f6e2eeb374cf729d4bb3f092dd4b36a8f861) skipped the tests if necessary.

### ifunc resolver calls a lazy binding PLT

For the two tests `sysdeps/x86/tst-ifunc-isa-*`, an ifunc resolver calls a lazy binding PLT.
GNU ld's x86-64 port places the `R_*_IRELATIVE` relocation after the PLT's `R_*_JUMP_SLOT` relocation, so that the program can work without an [ifunc scheduler](https://sourceware.org/bugzilla/show_bug.cgi?id=28218#c4).
This is GNU ld doing something to make a special case work.
The right approach is for glibc to fix the issue systematically.

I just XFAILed the two tests.

## Epilogue

It is one giant step for me, but just one small step for glibc.
When my ambition of building glibc with ld.lld started in April 2020, I felt frustrated quickly due to lack of review.
For my `configure: Allow LD to be LLD 9.0.0 or above` patch series, I could not tell whether it was due to objection of a non-GNU toolchain or just sticking with strict requirements.
Now I was told that different from the past there is no actively blocking in support of a toolchain beside GCC/binutils.

I tried to recover after a few months: "let me try again".
I had put aside this work until someone on IRC told me that they were interested in making glibc static pie work with ld.lld.
I was glad that I picked up the patches, persisted, and finally finished the work in the end of August 2021.
Porting does not necessarily mean increased complexity. I actually happened to remove quite a bit of quirk from the code base.

Reviewer resources are never abundant.
That said, perhaps a little bit more amicability could have made me feel better.

I wish that Clang can build glibc some day and the glibc community can advertise that Clang is fully supported.
GCC and binutils-gdb have been buildable with Clang for a long time.
Many GCC/binutils-gdb contributors are even happy to make Clang portability fixes.
Hope that the GNU and LLVM communities can have more collaboration.

So, dear glibc, will you be happy with my sending Clang patches? :)

## Current state

* aarch64/x86-32/x86-64: on par with GNU ld
* powerpc64le: [8 more failures](https://sourceware.org/pipermail/libc-alpha/2021-November/132708.html)
