layout: post
title: lld linked musl on PowerPC64
author: MaskRay
tags: [llvm,linker,kernel,musl]
---

I was asked about a segfault related to lld linked musl libc.so on PowerPC64.

* `/usr/lib/ld-musl-powerpc64le.so.1 /path/to/thing` worked. The kernel ELF loader loads rtld and rtld loads the executable.
* `/path/to/thing` segfaulted. The kernel ELF loader loads both rtld and the executable.

Therefore the bug is likely due to a difference between the two modes.

<!-- more -->

The section and program header dump from readelf looked like the following.
I annotated the interesting lines with `!!!`.

```text
There are 36 section headers, starting at offset 0x2e4890:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .note.gnu.build-id NOTE           0000000000000270 000270 000018 00   A  0   0  4
  [ 2] .dynsym           DYNSYM          0000000000000288 000288 009fc0 18   A  5   1  8
  [ 3] .gnu.hash         GNU_HASH        000000000000a248 00a248 003150 00   A  2   0  8
  [ 4] .hash             HASH            000000000000d398 00d398 003548 04   A  2   0  4
  [ 5] .dynstr           STRTAB          00000000000108e0 0108e0 004564 00   A  0   0  1
  [ 6] .rela.dyn         RELA            0000000000014e48 014e48 000e28 18   A  2   0  8
  [ 7] .rela.plt         RELA            0000000000015c70 015c70 000090 18  AI  2  17  8
  [ 8] .rodata           PROGBITS        0000000000015d00 015d00 0336fd 00 AMS  0   0 16
  [ 9] .eh_frame_hdr     PROGBITS        0000000000049400 049400 00001c 00   A  0   0  4
  [10] .eh_frame         PROGBITS        0000000000049420 049420 00004c 00   A  0   0  8
  [11] .text             PROGBITS        0000000000059480 049480 089308 00  AX  0   0 32
  [12] .glink            PROGBITS        00000000000e2788 0d2788 000054 00  AX  0   0  4
  [13] .data.rel.ro      PROGBITS        00000000000f27e0 0d27e0 000408 00  WA  0   0  8
  [14] .dynamic          DYNAMIC         00000000000f2be8 0d2be8 000140 10  WA  5   0  8
  [15] .got              PROGBITS        00000000000f2d28 0d2d28 000008 00  WA  0   0  8
  [16] .toc              PROGBITS        00000000000f2d30 0d2d30 0002a0 00  WA  0   0  8
  [17] .plt              NOBITS          00000000000f2fd0 0d2fd0 000040 00  WA  0   0  8 !!!
  [18] .data             PROGBITS        0000000000103010 0d3010 0003c0 00  WA  0   0  8
  [19] .bss              NOBITS          00000000001033d0 0d33d0 002ac8 00  WA  0   0 16
  [20] .branch_lt        NOBITS          0000000000105e98 0d33d0 000000 00  WA  0   0  8
  [21] .debug_loclists   PROGBITS        0000000000000000 0d33d0 046be2 00      0   0  1
  [22] .debug_abbrev     PROGBITS        0000000000000000 119fb2 0466c3 00      0   0  1
  [23] .debug_info       PROGBITS        0000000000000000 160675 089b79 00      0   0  1
  [24] .debug_rnglists   PROGBITS        0000000000000000 1ea1ee 006693 00      0   0  1
  [25] .debug_str_offsets PROGBITS       0000000000000000 1f0881 02783c 00      0   0  1
  [26] .debug_str        PROGBITS        0000000000000000 2180bd 013f8b 01  MS  0   0  1
  [27] .debug_addr       PROGBITS        0000000000000000 22c048 011780 00      0   0  1
  [28] .comment          PROGBITS        0000000000000000 23d7c8 000029 01  MS  0   0  1
  [29] .debug_frame      PROGBITS        0000000000000000 23d7f8 0176f0 00      0   0  8
  [30] .debug_line       PROGBITS        0000000000000000 254ee8 0657e4 00      0   0  1
  [31] .debug_line_str   PROGBITS        0000000000000000 2ba6cc 0089ee 01  MS  0   0  1
  [32] .debug_aranges    PROGBITS        0000000000000000 2c30ba 0001b0 00      0   0  1
  [33] .symtab           SYMTAB          0000000000000000 2c3270 016b90 18     35 2175  8
  [34] .shstrtab         STRTAB          0000000000000000 2d9e00 00016f 00      0   0  1
  [35] .strtab           STRTAB          0000000000000000 2d9f6f 00a91f 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)

Elf file type is DYN (Shared object file)
Entry point 0xdb8bc
There are 10 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x000230 0x000230 R   0x8
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x04946c 0x04946c R   0x10000
  LOAD           0x049480 0x0000000000059480 0x0000000000059480 0x08935c 0x08935c R E 0x10000
  LOAD           0x0d27e0 0x00000000000f27e0 0x00000000000f27e0 0x0007f0 0x000830 RW  0x10000 !!!
  LOAD           0x0d3010 0x0000000000103010 0x0000000000103010 0x0003c0 0x002e88 RW  0x10000
  DYNAMIC        0x0d2be8 0x00000000000f2be8 0x00000000000f2be8 0x000140 0x000140 RW  0x8
  GNU_RELRO      0x0d27e0 0x00000000000f27e0 0x00000000000f27e0 0x0007f0 0x001820 R   0x1
  GNU_EH_FRAME   0x049400 0x0000000000049400 0x0000000000049400 0x00001c 0x00001c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x0
  NOTE           0x000270 0x0000000000000270 0x0000000000000270 0x000018 0x000018 R   0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .note.gnu.build-id .dynsym .gnu.hash .hash .dynstr .rela.dyn .rela.plt .rodata .eh_frame_hdr .eh_frame
   02     .text .glink
   03     .data.rel.ro .dynamic .got .toc .plt
   04     .data .bss
   05     .dynamic
   06     .data.rel.ro .dynamic .got .toc .plt
   07     .eh_frame_hdr
   08
   09     .note.gnu.build-id
   None   .branch_lt .debug_loclists .debug_abbrev .debug_info .debug_rnglists .debug_str_offsets .debug_str .debug_addr .comment .debug_frame .debug_line .debug_line_str .debug_aranges .symtab .shstrtab .strtab
```

There were two `PT_LOAD` program headers with the `PF_R|PF_W` flags and the unusual property `p_filesz < p_memsz`.
For both RW `PT_LOAD` program headers, we had `roundUp(p_vaddr+p_filesz, pagesz) < roundUp(p_vaddr+p_memsz, pagesz)`.
It turns out that when the Linux kernel loads an interpreter (`PT_INTERP`; see `fs/binfmt_elf.c:load_elf_interp`), it only supports one `PT_LOAD` with `p_filesz < p_memsz`.

Note: it is typical for lld output to have two RW `PT_LOAD` program headers, one for RELRO sections (`PT_GNU_RELRO`) and the other for non-RELRO sections.
This may look unusual at the first glance but it avoids an alignment padding as used in GNU ld's single RW `PT_LOAD` layout.
See [Explain GNU style linker options#-z relro](/blog/2020-11-15-explain-gnu-linker-options#z-relro).

In the PowerPC ELFv2 ABI, `.plt` is like GOTPLT on other architectures (it holds resolved addresses for PLT entries) and has the `SHT_NOBITS` type.
With `-z now`, `.plt` can be eagerly resolved and become read-only after relocation resolving, therefore it is part of `PT_GNU_RELRO`.
When lld layouts sections, it is part of the first RW `PT_LOAD`. In the unlucky `libc.so`, `.plt` is 64 bytes (2 reserved pointer entries plus 6 pointer entries for `malloc/calloc/realloc/memalign/aligned_alloc/free`).
`p_memsz = p_filesz - 64`.
If `roundUp(p_vaddr+p_filesz, pagesz) < roundUp(p_vaddr+p_memsz, pagesz)`, relocation resolving will access an unmapped memory page and segfault.
If the comparison result is equal and we just have `p_filesz < p_memsz`, the kernel will fail to zero some bytes but the bytes will be overwritten by rtld anyway.

Clang passes `-z now` to ld for Alpine Linux. Chimera Linux has patched Clang Driver to pass `-z now` for all musl target triples.

* Pros: GOTPLT is part of RELRO and provides security hardening values. In addition, the `DF_1_NOW` flag avoids an allocation in its rtld. See this commit [emulate lazy relocation as deferrable relocation](https://git.musl-libc.org/cgit/musl/commit/?id=6476b8135760659b25c93ff9308425ca98a9e777).
* Cons: There is a slight size increase of `.dynamic`: it will always have a `DT_FLAGS` holding `DF_NOW`. In most cases `DT_FLAGS` can actually be absent if `-z now` is not used.

## Workarounds

There are multiple ways to work around the issue.

### `-z lazy`

The easiest is to build musl with `LDFLAGS=-Wl,-z,lazy` to override driver specified `-z now`. I verified with a local cross-compilation build.
```sh
mkdir out/ppc64le && cd out/ppc64le
../../configure --target=powerpc64le-linux-gnu CC=clang CFLAGS='--target=powerpc64le-linux-gnu -mlong-double-64' LDFLAGS=-fuse-ld=lld
make -j$(nproc)
```

Cons: loses some security hardening.

(If you use GCC's powerpc64 port, avoid `-Os`. lld has not implemented `_savefpr*` and `_restfpr*` functions.)

### `SHT_PROGBITS` `.plt`

The linker synthesized `.plt` has the `SHT_NOBITS` type. We can link a relocatable object file with an empty `SHT_PROGBITS` `.plt`.
```asm
.section .plt,"awR",@progbits
```

The output section will have the `SHT_NOBITS` type.

Note `R` for `SHF_RETAIN`. Without the flag, the linker option `--gc-sections` drops the input `.plt` so that it cannot affect the output section type.

### Prevent `roundUp(p_vaddr+p_filesz, pagesz) < roundUp(p_vaddr+p_memsz, pagesz)`

The musl build system forces `-Wl,--hash-style=both`. We can specify `LDFLAGS=-Wl,--hash-style=gnu` to drop `.hash`.

Alternatively, we may pad `.plt` or any preceding output section so that the property no longer holds.
```text
// a.lds
OVERWRITE_SECTIONS { .plt : { *(.plt) QUAD(0) QUAD(0) } };
```

Use `clang -o libc.so ... a.lds`.

You may be attempted to keep `-z now` and link `libc.so` with a linker script:
```text
SECTIONS { .plt : {} } INSERT AFTER .bss;
```

Unfortunately that would create discontinued RELRO sections, which is unsupported by linkers and most rtld implementations.

## glibc

glibc adopts a separate rtld and libc.so design. Its rtld has no `JUMP_SLOT` (`JMP_SLOT`) relocations.

The powerpc64 port has been buildable since [lld 13](/blog/2021-09-05-build-glibc-with-lld).
There is no `.plt` section, therefore the first RW `PT_LOAD` has `p_filesz == p_memsz`.
The built rtld works with Linux kernel.

I got `powerpc64le-linux-gnu-gcc` and binutils from system packages. I have installed `/usr/local/bin/powerpc64le-linux-gnu-ld.lld` so that `powerpc64le-linux-gnu-gcc -fuse-ld=lld` works.

```sh
mkdir out/ppc64le && cd out/ppc64le
LDFLAGS=-fuse-ld=lld ../../configure --prefix=/tmp/glibc/ppc64le --host=powerpc64le-linux-gnu --with-default-link --enable-hardcoded-path-in-tests
LDFLAGS=-fuse-ld=lld make -j $(nproc)
```

## Reliable reproduce

Here is the main trick: assemble the following assembly file `toc.s` and link it into musl `lib/libc.so`.
```text
.section .toc,"aw",@nobits
.globl toc
toc:
.space 4096
```

`.toc` is recognized as a RELRO section in ld.lld, even if the architecture is not PowerPC64:)

The section is 4096 (page size), therefore we can ensure `roundUp(p_vaddr+p_filesz, pagesz) < roundUp(p_vaddr+p_memsz, pagesz)`.

Compile the following C program, link it with `toc.o` using `-Wl,--dynamic-linker=path/to/libc.so`.
```c
#include <assert.h>
#include <stdio.h>

extern const char toc[];

int main(void) {
  assert(toc[4096-1] == 0);
  puts("hello");
}
```

The output will have two RW `PT_LOAD` program headers with `p_filesz < p_memsz`.

You can add custom sections to the `PT_GNU_RELRO` program header using a full linker script with `DATA_SEGMENT_ALIGN` and `DATA_SEGMENT_RELRO_END` ([implemented](https://reviews.llvm.org/D124656) in ld.lld 15).
```text
. = DATA_SEGMENT_ALIGN (CONSTANT (MAXPAGESIZE), CONSTANT (COMMONPAGESIZE));

/* RELRO sections */

. = DATA_SEGMENT_RELRO_END (0, .);
```

### Stress test

To test that the kernel ELF loader can handle more RW `PT_LOAD` program headers, we can add a few more `SHF_ALLOC|SHF_WRITE` sections (abbreviated as RW below).
We can place a read-only section after `.bss` followed by a RW section. The read-only section will form a read-only `PT_LOAD` and the RW section will form a RW `PT_LOAD`.

Create some files. If you have split-file (a [test utility](https://llvm.org/docs/TestingGuide.html#extra-files) from llvm-project), you may place the following content into `a.txt`.

```text
#--- a.c
#include <assert.h>
#include <stdio.h>

extern const char toc[];
extern char nobits0[], nobits1[];

int main(void) {
  assert(toc[4096-1] == 0);
  for (int i = 0; i < 1024; i++) {
    assert(nobits0[i] == 0);
    nobits0[i] = 1;
  }
  for (int i = 0; i < 8192; i++) {
    assert(nobits1[i] == 0);
    nobits1[i] = 1;
  }

  puts("hello");
}

#--- toc.s
.globl toc, nobits0, nobits1

.section .toc,"aw",@nobits; toc: .space 4096

.section .ro0,"a"; .byte 255
.section .nobits0,"aw",@nobits; nobits0: .space 1024
.section .ro1,"a"; .byte 255
.section .nobits1,"aw",@nobits; nobits1: .space 8192

#--- a.lds
SECTIONS { .ro0 : {} .nobits0 : {} .ro1 : {} .nobits1 : {} } INSERT AFTER .bss;
```

Then run:
```sh
split-file a.txt a
path/to/musl-gcc -Wl,--dynamic-linker=/lib/libc.so a/a.c a/a.lds -o toy
```

Note: when a `SHT_NOBITS` section is followed by another section, the `SHT_NOBITS` section behaves as if it occupies the file offset range. This is because ld.lld does not implement a file size optimization.

## Test a patched kernel

Pedro Falcato has a kernel patch [[PATCH] fs/binfmt_elf: Fix memsz > filesz handling](https://lore.kernel.org/all/20221106021657.1145519-1-pedro.falcato@gmail.com/) to fix the issue.
Let's verify it.

```sh
// In linux
patch -p1 -i /tmp/c/0001-fs-binfmt_elf-Fix-memsz-filesz-handling.patch
make -j $(nproc) O=/tmp/linux/x86_64 defconfig all
# Specify LLVM=1 if you want to use clang lld llvm-{ar,nm,objcopy,objdump,readelf,strings}
```

Now prepare an initrd image with the test program.
<https://github.com/ClangBuiltLinux/boot-utils> has a prebuilt image and adding extra files is convenient.
```sh
mkdir /tmp/initrd && cd /tmp/initrd
sudo cpio -i -F ~/Dev/ClangBuiltLinux/boot-utils/images/x86_64/rootfs.cpio
```

Copy musl `lib/libc.so` to `/tmp/initrd/lib/libc.so` and our toy program to `/tmp/initrd/toy`.
Edit `/tmp/initrd/init` to run `/toy || echo failed: $?`.
Rebuild the initrd image.

```sh
find . | sudo cpio -o --format=newc | zstd > ~/Dev/ClangBuiltLinux/boot-utils/images/x86_64/rootfs.cpio.zst
```

With an unpatched kernel, `/toy` segfaults as expected:
```
% ~/Dev/ClangBuiltLinux/boot-utils/boot-qemu.py -a x86_64 -k /tmp/linux/x86_64
...
Error relocating /lib/libc.so: RELRO protection failed: No error information
failed: 127
...
```

With a patched kernel, `/toy` succeeds.
```sh
% ~/Dev/ClangBuiltLinux/boot-utils/boot-qemu.py -a x86_64 -k /tmp/linux/x86_64
...
hello
...
```
