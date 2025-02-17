layout: post
title: Exploring the section layout in linker output
author: MaskRay
tags: [llvm,linker,linux]
---

This article describes section layout and its interaction with dynamic loaders and huge pages.

Let's begin with a Linux x86-64 example involving global variables exhibiting various properties such as read-only versus writable, zero-initialized versus non-zero, and more.

```c
#include <stdio.h>
const int ro = 1;
int w0, w1 = 1;
int *const pw0 = &w0;
int main() {
  printf("%d %d %d %p\n", ro, w0, w1, pw0);
}
```

<!-- more -->

```
% clang -c -fpie a.c
% clang -pie -fuse-ld=lld -Wl,-z,separate-loadable-segments a.o -o a
% objdump -wt a | grep -P 'main|w[01]|ro$'
00000000000010f0 g     F .text  000000000000002e              main
0000000000003044 g     O .bss   0000000000000004              w0
0000000000003010 g     O .data  0000000000000004              w1
000000000000058c g     O .rodata        0000000000000004              ro
0000000000002010 g     O .data.rel.ro   0000000000000008              pw0
% readelf -Wl a
...
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x000268 0x000268 R   0x8
  INTERP         0x0002a8 0x00000000000002a8 0x00000000000002a8 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000628 0x000628 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000180 0x000180 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x0001e0 0x001000 RW  0x1000
  LOAD           0x003000 0x0000000000003000 0x0000000000003000 0x000040 0x000048 RW  0x1000
  DYNAMIC        0x002018 0x0000000000002018 0x0000000000002018 0x0001a0 0x0001a0 RW  0x8
  GNU_RELRO      0x002000 0x0000000000002000 0x0000000000002000 0x0001e0 0x001000 R   0x1
  GNU_EH_FRAME   0x0005a0 0x00000000000005a0 0x00000000000005a0 0x00001c 0x00001c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0
  NOTE           0x0002c4 0x00000000000002c4 0x00000000000002c4 0x000020 0x000020 R   0x4
...
```

(We will discuss `-Wl,-z,separate-loadable-segments` [later](#z-separate-loadable-segments).)

We can see that these functions and global variables are placed in different sections.

* `.rodata`: read-only data without dynamic relocations, constant in the link unit
* `.text`: functions
* `.data.rel.ro`: read-only data associated with dynamic relocations, constant after relocation resolving, part of the `PT_GNU_RELRO` segment
* `.data`: writable data
* `.bss`: writable data known to be zeros

## Section and segment layout

TODO I may write more about how linkers layout sections and segments. <!-- sh_offset = sh_addr (mod sh_align) for non-NOBITS sections -->

Anyhow, the linker will place `.data` and `.bss` in the same `PT_LOAD` program header (segment) and the rest into different `PT_LOAD` segments.
(There are some nuances. If you use GNU ld's [`-z noseparate-code`](/blog/2020-11-15-explain-gnu-linker-options#z-noseparate-code) or lld's [`--no-rosegment`](/blog/2020-11-15-explain-gnu-linker-options#no-rosegment), `.rodata` and `.text` will be placed in the same `PT_LOAD` segment.)

The `PT_LOAD` segments have different flags (`p_flags`): `PF_R`, `PF_R|PF_X`, `PF_R|PF_W`.
Subsequently, the dynamic loader, also known as the dynamic linker, will invoke `mmap` to map the file into memory.
The memory areas (VMA) have different memory permissions corresponding to segment flags.

For a `PT_LOAD` segment, its associated memory area starts at `alignDown(p_vaddr, pagesize)` and ends at `alignUp(p_vaddr+p_memsz, pagesize)`.

```
          Start Addr           End Addr       Size     Offset  Perms  objfile
      0x555555554000     0x555555555000     0x1000        0x0  r--p   /tmp/c/a
      0x555555555000     0x555555556000     0x1000     0x1000  r-xp   /tmp/c/a
      0x555555556000     0x555555557000     0x1000     0x2000  r--p   /tmp/c/a
      0x555555557000     0x555555558000     0x1000     0x3000  rw-p   /tmp/c/a
```

Let's assume the page size is 4096 bytes. We'll calculate the `alignDown(p_vaddr, pagesize)` values and display them alongside the "Start Addr" values:
```
Start Addr       alignDown(p_vaddr, pagesize)
0x555555554000   0x0000000000000000
0x555555555000   0x0000000000001000
0x555555556000   0x0000000000002000
0x555555557000   0x0000000000003000
```

We observe that the start address equals the base address plus `alignDown(p_vaddr, pagesize)`.

When `SHT_NOTE` sections are present, they are placed at the beginning to make them more likely to be included in core files.
In the Linux kernel, the 2007 commit [Add MMF_DUMP_ELF_HEADERS](https://git.kernel.org/linus/82df39738ba9e02c057fa99b7461a56117d36119) is related.

## [`--no-rosegment`](/blog/2020-11-15-explain-gnu-linker-options#no-rosegment)

This option asks lld to combine the read-only and the RX segments.
The output file will consume less address space at run-time.

```
          Start Addr           End Addr       Size     Offset  Perms  objfile
      0x555555554000     0x555555555000     0x1000        0x0  r-xp   /tmp/c/a
      0x555555555000     0x555555556000     0x1000        0x0  r--p   /tmp/c/a
      0x555555556000     0x555555557000     0x1000     0x1000  rw-p   /tmp/c/a
```

## MAXPAGESIZE

A page serves as the granularity at which memory exhibits different permissions, and within a page, we cannot have varying permissions.
Using the previous example where `p_align` is 4096, if the page size is larger, for example, 65536 bytes, the program might crash.

Typically, the dynamic loader allocates memory for the first `PT_LOAD` segment (`PF_R`) at a specific address allocated by the kernel.
Subsequent `PT_LOAD` segments then overwrite the previous memory regions. Consequently, certain code pages or significant global variables might be replaced by garbage, leading to a crash.

So, how can we create a link unit that works across different page sizes?
We simply determine the maximum page size, let's say, 2097152, and then pass `-z max-page-size=2097152` to the linker.
The linker will set `p_align` values of `PT_LOAD` segments to MAXPAGESIZE.

```
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x000268 0x000268 R   0x8
  INTERP         0x0002a8 0x00000000000002a8 0x00000000000002a8 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000640 0x000640 R   0x200000
  LOAD           0x200000 0x0000000000200000 0x0000000000200000 0x000180 0x000180 R E 0x200000
  LOAD           0x400000 0x0000000000400000 0x0000000000400000 0x0001e0 0x001000 RW  0x200000
  LOAD           0x600000 0x0000000000600000 0x0000000000600000 0x000040 0x000048 RW  0x200000
  DYNAMIC        0x400018 0x0000000000400018 0x0000000000400018 0x0001a0 0x0001a0 RW  0x8
  GNU_RELRO      0x400000 0x0000000000400000 0x0000000000400000 0x0001e0 0x001000 R   0x1
  GNU_EH_FRAME   0x0005b8 0x00000000000005b8 0x00000000000005b8 0x00001c 0x00001c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0
  NOTE           0x0002c4 0x00000000000002c4 0x00000000000002c4 0x000038 0x000038 R   0x4
```

In a linker script, the `max-page-size` can be obtained using `CONSTANT(MAXPAGESIZE)`.

For completeness, if you need to run a prebuilt executable on a system with a larger page size, you can modify the executable by merging `PT_LOAD` segments and combining their permissions.
It's likely there will be a sizable RWX `PT_LOAD` segment, reminiscent of OMAGIC.

## Over-aligned segment

It is possible to increase the `p_align` value of one single `PT_LOAD` segment using an `aligned` attribute.
When this value exceeds the page size, the question arises: should the kernel loader or the dynamic loader determine a suitable base address to meet this alignment requirement?

In 2020, the Linux kernel loader made the decision to [align the base address according to the maximum `p_align`](https://git.kernel.org/linus/ce81bb256a224259ab686742a6284930cbe4f1fa).
This facilitates [transparent huge pages for mapped files](#transparent-huge-pages-for-mapped-files) at expense cost of reduced address randomization.

```
% cat align.c
#include <stdio.h>
__attribute__((aligned(A))) int aligned;
int main() { printf("%p\n", &aligned); }
% cc -DA=4096 align.c -o align && ./align
0x55e994c13000
% cc -DA=2097152 align.c -o align && ./align
0x55639a400000
```

Should a userspace dynamic loader do the same?
If it does, a variable with an alignment greater than the page size will indeed align accordingly.
As of glibc 2.35, it has [followed suit](https://sourceware.org/PR28676).

On the other hand, the traditional interpretation dictates that a variable with an alignment greater than the page size is invalid.
Most other dynamic loaders do not implement this particular logic, which has some overhead.

## `-z separate-loadable-segments`

In previous examples using `-z separate-loadable-segments`, the `p_vaddr` values of `PT_LOAD` segments are multiples of MAXPAGESIZE.
The generic ABI says "loadable process segments must have congruent values for p_vaddr and p_offset, modulo the page size."

> p_offset - This member gives the offset from the beginning of the file at which the first byte of the segment resides.
>
> p_vaddr - This member gives the virtual address at which the first byte of the segment resides in memory.

This alignment requirement aligns with the `mmap` documentation. For example, Linux man-pages specifies, "offset must be a multiple of the page size as returned by sysconf(_SC_PAGE_SIZE)."

The `p_offset` values are also multiples of MAXPAGESIZE.
After layouting out a `PT_LOAD` segment, the linker must pad the end by inserting zeros so that the next `PT_LOAD` segment starts at a multiple of MAXPAGESIZE.

However, the alignment padding is wasteful. Fortunately, we can link `a.o` using different MAXPAGESIZE and different alignment settings: `-z noseparate-code`,`-z separate-code`,`-z separate-loadable-segments`.

```sh
clang -pie -fuse-ld=lld -Wl,-z,noseparate-code a.o -o a0.4096
clang -pie -fuse-ld=lld -Wl,-z,noseparate-code,-z,max-page-size=65536 a.o -o a0.65536
clang -pie -fuse-ld=lld -Wl,-z,noseparate-code,-z,max-page-size=2097152 a.o -o a0.2097152

clang -pie -fuse-ld=lld -Wl,-z,separate-code a.o -o a1.4096
clang -pie -fuse-ld=lld -Wl,-z,separate-code,-z,max-page-size=65536 a.o -o a1.65536
clang -pie -fuse-ld=lld -Wl,-z,separate-code,-z,max-page-size=2097152 a.o -o a1.2097152

clang -pie -fuse-ld=lld -Wl,-z,separate-loadable-segments a.o -o a2.4096
clang -pie -fuse-ld=lld -Wl,-z,separate-loadable-segments,-z,max-page-size=65536 a.o -o a2.65536
clang -pie -fuse-ld=lld -Wl,-z,separate-loadable-segments,-z,max-page-size=2097152 a.o -o a2.2097152
```
```
% stat -c %s a0.4096 a0.65536 a0.2097152
6168
6168
6168
% stat -c %s a1.4096 a1.65536 a1.2097152
12392
135272
4198504
% stat -c %s a2.4096 a2.65536 a2.2097152
16120
200440
6295288
```

We can derive two properties:

* Under one MAXPAGESIZE, we have `size(noseparate-code) < size(separate-code) < size(separate-loadable-segments)`.
* For `-z noseparate-code`, increasing MAXPAGESIZE does not change the output size.

AArch64 and PowerPC64 have a default MAXPAGESIZE of 65536.
Staying with the `-z noseparate-code` default ensures that they will not experience unnecessary size increase.

## `-z noseparate-code`

How does `-z noseparate-code` work? Let's illustrate this with an example.

At the end of the read-only `PT_LOAD` segment, the address is 0x628.
Instead of starting the next segment at `alignUp(0x628, MAXPAGESIZE) = 0x1000`, we start at `alignUp(0x628, MAXPAGESIZE) + 0x628 % MAXPAGESIZE = 0x1628`.
Since the `.text` section has an alignment (`sh_addralign`) of 16, we start at 0x1630.
Although the address is advanced beyond necessity, the file offset (congruent to the address, modulo MAXPAGESIZE) can be decreased to 0x630, merely 8 bytes (due to alignment padding) after the previous section's end.

Moving forward, the end of the executable `PT_LOAD` segment has an address of 0x17b0.
Instead of starting the next segment at `alignUp(0x17b0, MAXPAGESIZE) = 0x2000`, we start at `alignUp(0x17b0, MAXPAGESIZE) + 0x17c0 % MAXPAGESIZE = 0x27b0`.
While we advance the address more than needed, the file offset can be decreased to 0x7b0, precisely at the previous section's end.

```
% readelf -WSl a0.4096
...
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        00000000000002a8 0002a8 00001c 00   A  0   0  1
  ...
  [12] .eh_frame         PROGBITS        00000000000005c0 0005c0 000068 00   A  0   0  8
  [13] .text             PROGBITS        0000000000001630 000630 00011e 00  AX  0   0 16
  ...
  [16] .plt              PROGBITS        0000000000001780 000780 000030 00  AX  0   0 16
  [17] .fini_array       FINI_ARRAY      00000000000027b0 0007b0 000008 08  WA  0   0  8
  ...
  [20] .dynamic          DYNAMIC         00000000000027c8 0007c8 0001a0 10  WA  7   0  8
  [21] .got              PROGBITS        0000000000002968 000968 000028 00  WA  0   0  8
  [22] .relro_padding    NOBITS          0000000000002990 000990 000670 00  WA  0   0  1
  [23] .data             PROGBITS        0000000000003990 000990 000014 00  WA  0   0  8
  ...
  [26] .bss              NOBITS          00000000000039d0 0009d0 000008 00  WA  0   0  4
...
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000628 0x000628 R   0x1000
  LOAD           0x000630 0x0000000000001630 0x0000000000001630 0x000180 0x000180 R E 0x1000
  LOAD           0x0007b0 0x00000000000027b0 0x00000000000027b0 0x0001e0 0x000850 RW  0x1000
  LOAD           0x000990 0x0000000000003990 0x0000000000003990 0x000040 0x000048 RW  0x1000
  DYNAMIC        0x0007c8 0x00000000000027c8 0x00000000000027c8 0x0001a0 0x0001a0 RW  0x8
  GNU_RELRO      0x0007b0 0x00000000000027b0 0x00000000000027b0 0x0001e0 0x000850 R   0x1
```

`-z separate-code` performs the trick when transiting from the first RW `PT_LOAD` segment to the second, whereas `-z separate-loadable-segments` doesn't.

## When MAXPAGESIZE is larger than the actual page size

Let's consider two adjacement `PT_LOAD` segments.
The memory area associated with the first segment ends at `alignUp(load[i].p_vaddr+load[i].p_memsz, pagesize)` while the memory area associated with the second one starts at `alignDown(load[i+1].p_vaddr, pagesize)`.
When the actual page size equals MAXPAGESIZE, the two addresses are identical.
However, if the actual page size is smaller, a gap emerges between these addresses.

A typical link unit generally presents three gaps.
These gaps might either be unmapped or mapped. When mapped, they necessitate `struct vm_area_struct` objects within the Linux kernel.
As of Linux 6.3.13, the size of `struct vm_area_struct` is 152 bytes.
For instance, 10000 mapped object files would require `10000 * 3 * sizeof(struct vm_area_struct) = 4,560,000 bytes`, signifying a considerable memory footprint.
You can refer to [Extra struct vm_area_struct with ---p created when PAGE_SIZE < max-page-size](https://sourceware.org/bugzilla/show_bug.cgi?id=31076).

Dynamic loaders typically invoke `mmap` using `PROT_READ`, encompassing the whole file, followed by multiple `mmap` calls using `MAP_FIXED` and the corresponding flags.
When dynamic loaders, like musl, don't process gaps, the gaps retain `r--p` permissions.
However, in glibc's `elf/dl-map-segments.h`, the `has_holes` code employs `mprotect` to transition permissions from `r--p` to `---p`.

While `---p` might be perceived as a security enhancement, personally, I don't believe it significantly impacts exploitability.
While there might be numerous gadgets in `r-xp` areas, reducing gadgets in `r--p` areas doesn't seem notably impactful. (<https://isopenbsdsecu.re/mitigations/rop_removal/>)

### Unmap the gap

Within Linux kernel loads the executable and its interpreter (it present) (`fs/binfmt_elf.c`), the gap gets unmapped, thereby freeing a `struct vm_area_struct` object.
Implementing a similar approach in dynamic loaders could yield comparable savings.

However, unmapping the gap carries the risk of an unrelated future `mmap` occupying the gap:

```
564d8e90f000-564d8e910000 r--p 00000000 08:05 2519504        /sample/build/main
   ================ an unrelated mmap may be placed in the gap
564d8e91f000-564d8e920000 r-xp 00010000 08:05 2519504        /sample/build/main
```

It is not clear whether the potential occurrence of an unrelated mmap considered a regression in security.
Personally, I don't think this poses a significant issue as the program does not access the gaps.
This property can be guaranteed for direct access when input relocations to the linker use symbols with in-bounds addends (e.g. when x is defined relative to an input section, we know `R_X86_64_PC32(x)` must be in-bounds).

However, some programs may expect contiguous maps areas of a file (such as when glibc `link_map::l_contiguous` is set to 1).
Does this choice render the program exploitable if an attacker can ensure a map within the gap instead of outside the file?
It seems to me that they could achieve everything with a map outside of the file.

Having said that, the presence of an unrelated map between maps associated with a single file descriptor remains odd, so it's preferable to avoid it if possible.

### Extend the memory area to cover the gap

This appears the best solution.

When creating a memory area, instead of setting the end to `alignUp(load[i].p_vaddr+load[i].p_memsz, pagesize)`, we can extend the end to `min(alignDown(min(load[i+1].p_vaddr), pagesize), alignUp(file_end_addr, pagesize))`.

```
564d8e90f000-**564d8e91f000** r--p 00000000 08:05 2519504        /sample/build/main  (the end is extended)
564d8e91f000-564d8e920000 r-xp 00010000 08:05 2519504        /sample/build/main
```

For the last `PT_LOAD` segment, we could also just use `alignDown(min(load[i+1].p_vaddr), pagesize)` and ignore `alignUp(file_end_addr, pagesize))`.
Accessing a byte beyond the backed file will result to a `SIGBUS` signal.

<!-- Android https://r.android.com/2902180 -->

### A new linker option?

Personally I favor the area end extending approach.
I've also pondered whether this falls under the purview of linkers. Such a change seems intrusive and unsightly.
If the linker extends the end of p_memsz to cover the gap, should it also extend p_filesz?

* If it doesn't, we create a PT_LOAD with p_filesz/p_memsz that is not for BSS, which is weird.
* If it does, we have an output file featuring overlapping file offset ranges, which is weird as well.

Moreover, a PT_LOAD whose end isn't backed by a section is unusual. I'm concerned that many binary manipulation tools may not handle this case correctly.
Utilizing a linker script can intentionally create discontiguous address ranges. I'm concerned that the linker might not discern such cases with intelligent logic regarding p_filesz/p_memsz.

This feature request seems to be within the realm of loaders and specific information, such as the page size, is only accessible to loaders.
I believe loaders are better equipped to handle this task.

## Transparent huge pages for mapped files

Some programs optimize their usage of the limited Translation Lookaside Buffer (TLB) by employing transparent huge pages.
When the Linux kernel loads an executable, it takes into account the `p_align` field to create a memory area.
If `p_align` is 4096, the memory area will commence at a multiple of 4096, but not necessarily at a multiple of a huge page.

Transparent huge pages for mapped files have several requirements including:

* the memory area's start address and the start file offset align with a huge page (`include/linux/huge_mm.h:transhuge_vma_suitable`).
* `CONFIG_READ_ONLY_THP_FOR_FS` is enabled (`scripts/config -e TRANSPARENT_HUGEPAGE -e TRANSPARENT_HUGEPAGE_MADVISE -e READ_ONLY_THP_FOR_FS`)
* <del>the VMA has the `VM_EXEC` flag</del> (I [removed this condition for v6.8](https://git.kernel.org/linus/7fbb5e188248c50f737720825da1864ce42536d1))
* the file is not opened for write

When `madvise(addr, len, MADV_HUGEPAGE)` is called, the kernel code path is `do_madvise -> madvise_vma_behavior -> hugepage_madvise -> khugepaged_enter_vma -> thp_vma_allowable_order+__khugepaged_enter`.

To ensure that `addr-fileoff` is a multiple of a huge page, we should link the executable using `-z max-page-size=` with the huge page size.

In kernels with the `VM_EXEC` requirement (before v6.8), if we want to remap the file as huge pages from the ELF header, we must specify `--no-rosegment` to ld.lld.

Build the following program with `c++ -fuse-ld=lld -Wl,-z,max-page-size=2097152` and run it. We do not define `COLLAPSE` for now.
```cpp
#include <err.h>
#include <errno.h>
#include <fcntl.h>
#include <linux/kernel-page-flags.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

// Adapted from https://mazzo.li/posts/check-huge-page.html
// normal page, 4KiB
#define PAGE_SIZE (1 << 12)
// huge page, 2MiB
#define HPAGE_SIZE (1 << 21)

// See <https://www.kernel.org/doc/Documentation/vm/pagemap.txt> for
// format which these bitmasks refer to
#define PAGEMAP_PRESENT(ent) (((ent) & (1ull << 63)) != 0)
#define PAGEMAP_PFN(ent) ((ent) & ((1ull << 55) - 1))

extern char __ehdr_start[];
__attribute__((used)) const char pad[HPAGE_SIZE] = {};

// Checks if the page pointed at by `ptr` is huge. Assumes that `ptr` has
// already been allocated.
static void check_huge_page(void *ptr) {
  if (getuid())
    return warnx("not root; skip KPF_THP check");
  int pagemap_fd = open("/proc/self/pagemap", O_RDONLY);
  if (pagemap_fd < 0)
    errx(1, "could not open /proc/self/pagemap: %s", strerror(errno));
  int kpageflags_fd = open("/proc/kpageflags", O_RDONLY);
  if (kpageflags_fd < 0)
    errx(1, "could not open /proc/kpageflags: %s", strerror(errno));

  // each entry is 8 bytes long
  uint64_t ent;
  if (pread(pagemap_fd, &ent, sizeof(ent), ((uintptr_t)ptr) / PAGE_SIZE * 8) != sizeof(ent))
    errx(1, "could not read from pagemap\n");

  if (!PAGEMAP_PRESENT(ent))
    errx(1, "page not present in /proc/self/pagemap, did you allocate it?\n");
  if (!PAGEMAP_PFN(ent))
    errx(1, "page frame number not present, run this program as root\n");

  uint64_t flags;
  if (pread(kpageflags_fd, &flags, sizeof(flags), PAGEMAP_PFN(ent) << 3) != sizeof(flags))
    errx(1, "could not read from kpageflags\n");
  if (!(flags & (1ull << KPF_THP)))
    errx(1, "could not allocate huge page\n");
  if (close(pagemap_fd) < 0)
    errx(1, "could not close /proc/self/pagemap: %s", strerror(errno));
  if (close(kpageflags_fd) < 0)
    errx(1, "could not close /proc/kpageflags: %s", strerror(errno));
}

int main() {
  printf("__ehdr_start: %p\n", __ehdr_start);
  int ret, tries = 2;
#ifdef COLLAPSE // use Linux 6.1 MADV_COLLAPSE
  do {
    ret = madvise(__ehdr_start, HPAGE_SIZE, MADV_COLLAPSE);
  } while (ret && errno == EAGAIN && --tries);
  printf("madvise(MADV_COLLAPSE): %d\n", ret);
  if (ret) {
    ret = madvise(__ehdr_start, HPAGE_SIZE, MADV_HUGEPAGE);
    if (ret)
      err(1, "madvise");
  }
#else
  ret = madvise(__ehdr_start, HPAGE_SIZE, MADV_HUGEPAGE);
  if (ret)
    err(1, "madvise");
#endif

  size_t size = HPAGE_SIZE;
  char *buf = (char *)aligned_alloc(HPAGE_SIZE, size);
  madvise(buf, 2 << 20, MADV_HUGEPAGE);
  *((volatile char *)buf);
  check_huge_page(buf);

  int fd = open("/proc/self/maps", O_RDONLY);
  read(fd, buf, HPAGE_SIZE);
  write(STDOUT_FILENO, buf, strstr(buf, "[stack]\n") - buf + 8);
  close(fd);

#ifndef COLLAPSE
  fd = open("/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs", O_RDONLY);
  read(fd, buf, 32);
  close(fd);
  usleep(atoi(buf) * 1000);
#endif

  memcpy(buf, __ehdr_start, HPAGE_SIZE);
  check_huge_page(__ehdr_start);
}
```

The output looks like:
```
% g++ test.cc -o ~/tmp/test -O2 -fuse-ld=lld -Wl,-z,max-page-size=2097152 && sudo ~/tmp/test
__ehdr_start: 0x55f3b1c00000
55f3b1c00000-55f3b1e00000 r--p 00000000 103:03 555277119                 /home/ray/tmp/test
55f3b1e00000-55f3b1e01000 r--p 00200000 103:03 555277119                 /home/ray/tmp/test
55f3b2000000-55f3b2002000 r-xp 00200000 103:03 555277119                 /home/ray/tmp/test
55f3b2201000-55f3b2202000 r--p 00201000 103:03 555277119                 /home/ray/tmp/test
55f3b2401000-55f3b2402000 rw-p 00201000 103:03 555277119                 /home/ray/tmp/test
55f3b3a9a000-55f3b3abb000 rw-p 00000000 00:00 0                          [heap]
```

Thanks to 周洲仪 for helping me figure out the khugepaged behavior.

`usleep` gives khugepaged an opportunity to collapse pages (`hpage_collapse_scan_file => collapse_file => retract_page_tables => pmdp_collapse_flush`).
In the fortunate scenario when this collapse occurs, and the next page fault is triggered (`memcpy(buf, __ehdr_start, HPAGE_SIZE)`), the kernel will populate the `pmd` with a huge page (`handle_page_fault ...=> handle_pte_fault ...=> do_fault_around => filemap_map_pages ...=> do_set_pmd => set_pmd_at`).

However, in an unfortunate case, `check_huge_page(__ehdr_start)` will fail with `could not allocate huge page`.
`scan_sleep_millisecs` defaults to 10000 (10 seconds). Reducing the value increases the likelihood of the fortunate case.

Linux 6.1 introduces `MADV_COLLAPSE` to attempt a synchronous collapse of the native pages mapped by the memory range into Transparent Huge Pages (THPs).
While success is not guaranteed, a successful collapse eliminates the need to wait for the khugepaged daemon (`madvise_collapse => hpage_collapse_scan_file => collapse_file => retract_page_tables => pmdp_collapse_flush`).
In the event of repeated `MADV_COLLAPSE` failures, a fallback mechanism using `MADV_HUGEPAGE` can be employed.
```
% g++ -static -DCOLLAPSE test.cc -o test -O2 -fuse-ld=lld -Wl,-z,max-page-size=2097152
% sudo ./test
__ehdr_start: 0x200000
madvise(MADV_COLLAPSE): -1
...
test: could not allocate huge page
% sudo ./test
__ehdr_start: 0x55f3b1c00000
madvise(MADV_COLLAPSE): 0
00200000-00429000 r--p 00000000 fd:03 260                                /root/test
00628000-0069f000 r-xp 00228000 fd:03 260                                /root/test
0089e000-008a3000 r--p 0029e000 fd:03 260                                /root/test
00aa2000-00aa5000 rw-p 002a2000 fd:03 260                                /root/test
00aa5000-00aab000 rw-p 00000000 00:00 0
01800000-01822000 rw-p 00000000 00:00 0                                  [heap]
7fd141600000-7fd141800000 rw-p 00000000 00:00 0
7fd141800000-7fd141a00000 rw-p 00000000 00:00 0
7fd141a00000-7fd141a01000 rw-p 00000000 00:00 0
7ffe69edf000-7ffe69f00000 rw-p 00000000 00:00 0                          [stack]
```

In `-z noseparate-code` layouts, the file content starts somewhere at the first page, potentially wasting half a huge page on unrelated content.
Switching to `-z separate-code` allows reclaiming the benefits of the half huge page but increases the file size.
Balancing these aspects poses a challenge.
One potential solution is using `fallocate(FALLOC_FL_PUNCH_HOLE)`, which introduces complexity into the linker.
However, this approach feels like a workaround to address a kernel limitation.
It would be preferable if a file-backed huge page didn't necessitate a file offset aligned to a huge page boundary.

### Cost of RELRO

To accommodate `PT_GNU_RELRO`, the `RW` region will possess two permissions after the runtime linker maps the program.
While GNU ld provides one RW segment split by the dynamic loader, lld employs two explicit RW `PT_LOAD` segments.
After relocation resolving, the effects of lld and GNU ld are similar.

For those curious, explore my notes on GNU ld's [file size increase due to RELRO](/blog/2020-11-15-explain-gnu-linker-options#z-relro).

Due to RELRO, covering the two RW `PT_LOAD` segments necessitates a minimum of 2 (huge) pages.
In contrast, without RELRO, only one (huge) page is required at minimum.
This means potentially wasting up to MAXPAGESIZE-1 bytes, which could otherwise be utilized to cover more data.

Nowadays, RELRO is considered a security baseline and removing it might unsettle security-minded individuals.
