layout: post
title: PI_STATIC_AND_HIDDEN/HIDDEN_VAR_NEEDS_DYNAMIC_RELOC in glibc rtld
author: MaskRay
tags: [glibc,m68k,powerpc]
---

Recently I have fixed two glibc rtld bugs related to early GOT relocation for retro-computing architectures: m68k and powerpc32.
They are related to the obscure `PI_STATIC_AND_HIDDEN` macro which I am going to demystify.

In 2002, [`PI_STATIC_AND_HIDDEN` was introduced](https://sourceware.org/git/?p=glibc.git;a=commit;h=5a47e7f2a85ecc103175095c0f4104065fd4d8db) into glibc rtld (runtime loader).
This macro indicates whether accesses to the following types of variables need dynamic relocations.

* static specifier: `static int a;` (`STB_LOCAL`)
* hidden visibility attribute: `__attribute__((visibility("hidden"))) int a;` (`STB_GLOBAL STV_HIDDEN`), `__attribute__((weak, visibility("hidden"))) int a;` (`STB_WEAK STV_HIDDEN`)

`PI` in the macro name is an abbreviation for "position independent". This is a misnomer: a code sequence using GOT is typically position-independent as well.

In `-fPIC` mode, the compiler assumes that all non-local `STV_DEFAULT` symbols may be preemptible at run time.
A GOT-generating relocation is used and the GOT is typically unavoidable at link time (on some architectures the linker can optimize out the GOT).
This case is not interesting to rtld as rtld does not need to export such variables.

Excluding these cases (non-local `STV_DEFAULT`), all other variables are known to be non-preemptible at compile time.
The compiler can generate code which is guaranteed to avoid dynamic relocations at link time.

```c
static int var;
int foo() { return ++var; }
```

On 2022-04-26, I replaced `PI_STATIC_AND_HIDDEN` with the opposite macro [`HIDDEN_VAR_NEEDS_DYNAMIC_RELOC`](https://sourceware.org/git/?p=glibc.git;a=commit;h=098a657fe449a217cf65c5270d5fbc8d40b5b4e6).

## Non-`HIDDEN_VAR_NEEDS_DYNAMIC_RELOC` architectures with PC-relative instructions

To avoid dynamic relocations, the most common approach is to generate PC-relative instructions, as most modern architectures (e.g. aarch64, riscv, and x86-64) provide.
Using PC-relative instructions to reference variables assumes that the distance from code to data is a link-time constant.
Nowadays this condition is satisfied everywhere except the rare FDPIC ABI.

Here are some assembly fragments from architectures using PC-relative instructions.
The instructions may not be familar to you, but that is fine. We can see that there is no GOT related marker.
I have added some comments indicating the relocation type and the referenced symbol.
`var` in the C code has internal linkage which lowers to the `STB_LOCAL` binding.
References to such local symbols are often redirected to the section symbol (`.bss`): the link-time behaviors are identical.
```asm
# aarch64
        adrp    x1, .LANCHOR0               # R_AARCH64_ADR_PREL_PG_HI21 .bss
        ldr     w0, [x1, #:lo12:.LANCHOR0]  # R_AARCH64_LDST32_ABS_LO12_NC .bss
        add     w0, w0, 1
        str     w0, [x1, #:lo12:.LANCHOR0]  # R_AARCH64_LDST32_ABS_LO12_NC .bss

# arm
        ldr     r3, .L3
.LPIC0:
        add     r3, pc, r3
        ldr     r0, [r3]
        add     r0, r0, #1
        str     r0, [r3]
.L3:
        .word   .LANCHOR0-(.LPIC0+8)  # R_ARM_REL32 .bss

# riscv64
        lla     a5,.LANCHOR0  # R_RISCV_PCREL_HI20+R_RISCV_PCREL_LO12_I
        lw      a0,0(a5)
        addiw   a0,a0,1
        sw      a0,0(a5)

# x86-64
        movl    var(%rip), %eax  # R_X86_64_PC32 .bss-0x4
        addl    $1, %eax
        movl    %eax, var(%rip)  # R_X86_64_PC32 .bss-0x4
```

## Non-`HIDDEN_VAR_NEEDS_DYNAMIC_RELOC` architectures without PC-relative instructions

Many older architectures do not have PC-relative instructions.

x86-32 does not have PC-relative instructions, but it provides a way to avoid a load from a GOT entry.
It achieves this with a detour: compute the address of `_GLOBAL_OFFSET_TABLE_` (GOT base symbol), then add an offset (`S-_GLOBAL_OFFSET_TABLE_`) to get the symbol address.
`_GLOBAL_OFFSET_TABLE_` is computed this way: compute the address of a location in code, then add an offset (`_GLOBAL_OFFSET_TABLE_ - PC`).

You probably see now how the x86-32 ABI was misdesigned: the involvement of `_GLOBAL_OFFSET_TABLE_` is unnecessary.
A relocation with the calculation of `S-_GLOBAL_OFFSET_TABLE_` would achieve the same net effect.

The relocations with `GOT` in their `names` just use the GOT as an anchor. They don't indicate a load from a GOT entry.
```asm
# x86-32
        call    __x86.get_pc_thunk.dx         # R_386_PC32   __x86.get_pc_thunk.dx
        addl    $_GLOBAL_OFFSET_TABLE_, %edx  # R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
        movl    var@GOTOFF(%edx), %eax        # R_386_GOTOFF .bss
        addl    $1, %eax
        movl    %eax, var@GOTOFF(%edx)        # R_386_GOTOFF .bss
```

powerpc64 does not have PC-relative instructions before POWER10.
Earlier microarchitectures use TOC-relative relocations to compute the symbol address.
```asm
        addis 10,2,.LANCHOR0@toc@ha  # R_PPC64_TOC16_HA
        lwz 9,.LANCHOR0@toc@l(10)    # R_PPC64_TOC16_LO
        addi 9,9,1
        extsw 3,9
        stw 9,.LANCHOR0@toc@l(10)    # R_PPC64_TOC16_LO
```

A pending patch [[PATCH v3] powerpc64: Enable static-pie](https://sourceware.org/pipermail/libc-alpha/2022-April/137987.html) will define `PI_STATIC_AND_HIDDEN`.

## `HIDDEN_VAR_NEEDS_DYNAMIC_RELOC` architectures

A few older architectures tend to use a load from a GOT entry.
The GOT entry needs a relative relocation (instead of `R_*_GLOB_DAT`: the symbol is non-preemptible, so no symbol search is needed).
See [All about Global Offset Table](/blog/2021-08-29-all-about-global-offset-table).
In glibc, these architecture define `HIDDEN_VAR_NEEDS_DYNAMIC_RELOC`.

Some architectures even assume the distance from code to data may not be a link-time constant (see [All about Procedure Linkage Table](/blog/2021-09-19-all-about-procedure-linkage-table)).
They do not provide a relocation with a calculation of `S-_GLOBAL_OFFSET_TABLE_` or `S-P`.
```asm
# m68k
        move.l var@GOT(%a5),%a0  # R_68K_GOT32O var
        move.l (%a0),%d0
        addq.l #1,%d0
        move.l %d0,(%a0)

# microblaze
        lwi     r4,r20,var@GOT  # R_MICROBLAZE_GOT_64 var
        lwi     r3,r4,0
        addik   r3,r3,1
        swi     r3,r4,0

# nios2: r22 is a callee-saved register which requires a spill and expensive setup
        ldw     r3, %got(var)(r22)  # R_NIOS2_GOT16 var
        ldw     r2, 0(r3)
        addi    r2, r2, 1
        stw     r2, 0(r3)

# powerpc32: r30 is a callee-saved register which requires a spill and expensive setup
        lwz 9,.LC0-.LCTOC1(30)
        lwz 3,0(9)
        addi 3,3,1
        stw 3,0(9)

        .section        ".got2","aw"  # Like a manual GOT section
        .align 2
.LCTOC1 = .+32768
.LC0:
        .long .LANCHOR0  # R_PPC_ADDR32 .bss; may become R_PPC_RELATIVE at link time

        .section        ".bss"
        .set    .LANCHOR0,. + 0
var:
        .zero   4
```

The first task of rtld is to relocate itself and bind all symbols to itself.
Afterward, non-preemptible functions and data can be freely accessed.

On architectures where a GOT entry is used to access a non-preemptible variable, rtld needs to be careful not to reference such variables before relative relocations are applied.
In `rtld.c`, `_dl_start` has the following code:

```c
  if (bootstrap_map.l_addr)
    {
      // Apply R_*_RELATIVE, R_*_GLOB_DAT, and R_*_JUMP_SLOT.
      ELF_DYNAMIC_RELOCATE (&bootstrap_map, NULL, 0, 0, 0);
    }

  __rtld_malloc_init_stubs ();

  // _rtld_local_ro.dl_find_object
  GLRO (dl_find_object) = &_dl_find_object;
```

`_rtld_local_ro` is a hidden global variable.
Taking its address may be reordered before `ELF_DYNAMIC_RELOCATE` by the compiler.
On an architecture using a GOT entry to load the address, the reordering will make the subsequent memory store (`_rtld_local_ro.dl_find_object`) to crash, since the GOT address is incorrect: it's zero or the link-time address instead of the run-time address.

## powerpc32

I recently cleaned up the bootstrap code a bit with [`elf: Move elf_dynamic_do_Rel RTLD_BOOTSTRAP branches outside`](https://sourceware.org/pipermail/libc-alpha/2022-April/137926.html).
Afterwards, GCC powerpc32 appears to reliably reorder `_rtld_local_ro`, causing `ld.so` to crash right away.

```sh
mkdir -p out/ppc; cd out/ppc
../../configure --prefix=/tmp/glibc/ppc --host=powerpc-linux-gnu CC=powerpc-linux-gnu-gcc CXX=powerpc-linux-gnu-g++ && make -j 50 && make -j 50 install && 'cp' -f /usr/powerpc-linux-gnu/lib/libgcc_s.so.1 /tmp/glibc/ppc/lib
```
```text
% elf/ld.so
qemu: uncaught target signal 11 (Segmentation fault) - core dumped
[1]    373503 segmentation fault  elf/ld.so
```

I was pretty sure there is a relocation bug but was not immediately clear which piece of code may be at fault.

Nowadays there aren't many choices for powerpc32 images. [Void Linux ppc](https://voidlinux-ppc.org/) still provides powerpc32 glibc and musl images.
I downloaded one and fed it into qemu, booted it with `qemu-system-ppc -machine mac99 -m 2047M -cdrom void-live-ppc-20210825.iso -net nic -net user,smb=$HOME/Dev -boot d`.
I booted into the 4.4.261 kernel because gdb aborts immediately with 5.13.12 kernel.
Daniel Kolesa mentioned this 5.x kernel incompatibility to me and nobody has looked into it yet.

The live CD provides free space of about 1GiB and I can install cifs-utils and gdb.
Then run ld.so under gdb.

```sh
xbps-install -S
xbps-install cifs-utils gdb cgdb
mkdir ~/Dev
mount -t cifs -o vers=3.0 //10.0.2.4/qemu ~/Dev
cd ~/Dev/glibc/out/ppc
gdb -ex 'directory ../../elf' -ex r elf/ld.so
```

gdb says `stw     r9,1168(r25)` triggers SIGSEGV.
```text
% powerpc-linux-gnu-objdump --disassemble=_dl_start -S elf/ld.so
...
  if (bootstrap_map.l_addr)
   1f468:       40 96 01 64     bne     cr5,1f5cc <_dl_start+0x3ac>
   1f46c:       83 3e ff c4     lwz     r25,-60(r30)  # load the address of _rtld_local_ro from GOT => 0
   1f470:       3a 61 00 10     addi    r19,r1,16
  bootstrap_map.l_relocated = 1;
   1f474:       a1 21 01 b0     lhz     r9,432(r1)
   1f478:       61 29 20 00     ori     r9,r9,8192
   1f47c:       b1 21 01 b0     sth     r9,432(r1)
  __rtld_malloc_init_stubs ();
   1f480:       4b ff d5 f1     bl      1ca70 <__rtld_malloc_init_stubs>
  GLRO (dl_find_object) = &_dl_find_object;
   1f484:       81 3e ff b8     lwz     r9,-72(r30)
    ElfW(Addr) entry = _dl_start_final (arg, &info);
   1f488:       7e 64 9b 78     mr      r4,r19
   1f48c:       7f 83 e3 78     mr      r3,r28
  GLRO (dl_find_object) = &_dl_find_object;
   1f490:       91 39 04 90     stw     r9,1168(r25)  # access 0+1168 => SIGSEGV
    ElfW(Addr) entry = _dl_start_final (arg, &info);
   1f494:       4b ff fb 9d     bl      1f030 <_dl_start_final>
```

Then I confirm that the GOT entry corresponds to `_rtld_local_ro`.
```text
% readelf -Ws elf/ld.so | grep 4ffb8
0004ffb8  00000016 R_PPC_RELATIVE                    4efc8
% readelf -Ws elf/ld.so | grep 4efc8
     7: 0004efc8  1192 OBJECT  GLOBAL DEFAULT   14 _rtld_global_ro@@GLIBC_PRIVATE
   583: 0004efc8  1192 OBJECT  LOCAL  DEFAULT   14 _rtld_local_ro
   726: 0004efc8  1192 OBJECT  GLOBAL DEFAULT   14 _rtld_global_ro
```

[elf: Move post-relocation code of _dl_start into _dl_start_final](https://sourceware.org/pipermail/libc-alpha/2022-April/138118.html) shall fix the bug.

Note: adding `asm volatile("" ::: "memory");` in between does not prevent reordering.

Note: in the absence of a powerpc32 system, `qemu-ppc-static -d in_asm elf/ld.so` may provide some clue about the faulty basic block.
```text
----------------
IN: _dl_start
0x4001f484:  813effb8  lwz      r9, -0x48(r30)
0x4001f488:  7e649b78  mr       r4, r19
0x4001f48c:  7f83e378  mr       r3, r28
0x4001f490:  91390490  stw      r9, 0x490(r25)
0x4001f494:  4bfffb9d  bl       0x4001f030

qemu: uncaught target signal 11 (Segmentation fault) - core dumped
[1]    383218 segmentation fault  qemu-ppc-static -d in_asm elf/ld.so
```

## m68k

Last week I fixed a similar bug for m68k: [m68k: Removal of ELF_DURING_STARTUP optimization broke ld.so](https://sourceware.org/bugzilla/show_bug.cgi?id=29071).

`ld.so` has 671 `R_68K_RELATIVE` relocations and one `R_68K_GLOB_DAT` for `__stack_chk_guard@@GLIBC_2.4`.
The following function is used to apply a relocation. It is shared by self-relocation and relocation for other modules.
The self-relocation code defines `RTLD_BOOTSTRAP` and needs just `R_68K_RELATIVE`, `R_68K_GLOB_DAT`, and `R_68K_JMP_SLOT`.

```c
// sysdeps/m68k/dl-machine.h
static inline void __attribute__ ((unused, always_inline))
elf_machine_rela (struct link_map *map, struct r_scope_elem *scope[],
                  const Elf32_Rela *reloc, const Elf32_Sym *sym,
                  const struct r_found_version *version,
                  void *const reloc_addr_arg, int skip_ifunc)
{
  Elf32_Addr *const reloc_addr = reloc_addr_arg;
  const unsigned int r_type = ELF32_R_TYPE (reloc->r_info);

  if (__builtin_expect (r_type == R_68K_RELATIVE, 0))
    *reloc_addr = map->l_addr + reloc->r_addend;
  else
    {
      ...
      switch (r_type)
        {
        case R_68K_COPY:
          ...
        case R_68K_GLOB_DAT:
        case R_68K_JMP_SLOT:
          *reloc_addr = value;
          break;
```

However, somehow many case labels were available for self-relocation.
GCC compiles the switch statement into a jump table which requires loading an address from GOT.
With some clean-up to generic relocation code, GCC decides to perform loop-invariant code motion and hoists the load of the jump table address.
The hoisted load is before relative relocations are applied, so the jump table address is incorrect.

The foolproof approach is to add an optimization barrier (e.g. calling an non-inlinable function after relative relocations are resolved).
That is non-trivial given the code structure. So Andreas Schwab suggested a simple approach by avoiding the jump table: handle just the essential relocations.

The faulty code concealed well and I could not have found it without a debugger.
It took me a while to set up a m68k image using q800.
The memory is limited to 1000MiB and the emulation is very slow.
Linux 5.19 is expected to gain the support for a virtual Motorola 68000 machine. With `qemu-system-m68k -M virt` things will become better.

```sh
# Installation

7z x debian-11.0.0-m68k-NETINST-1.iso install/kernels/vmlinux-5.16.0-5-m68k install/cdrom/initrd.gz
mv install/kernels/vmlinux-5.16.0-5-m68k install/cdrom/initrd.gz .

qemu-img create -f qcow2 debian-m68k.qcow2 8G
qemu-system-m68k -M q800 -m 1000m -serial none -serial mon:stdio -net nic,model=dp83932 -net user -kernel vmlinux-5.16.0-5-m68k -initrd initrd.gz -append 'console=ttyS0 vga=off' -drive file=debian-m68k.qcow2,format=qcow2 -drive file=debian-11.0.0-m68k-NETINST-1.iso,format=raw,media=cdrom -nographic -boot d

# After installation

sudo qemu-nbd -c /dev/nbd0 m68k-deb10.qcow2
# Extract vmlinux and initrd
sudo qemu-nbd -d /dev/nbd0 m68k-deb10.qcow2

qemu-system-m68k -M q800 -m 1000M -kernel vmlinux-5.16.0-6-m68k -initrd initrd.img-5.16.0-6-m68k -append 'root=/dev/sda2 console=tty' -hda debian-m68k.qcow2 -net nic,model=dp83932 -net user,smb=$HOME/Dev
```

## musl rtld

musl rtld has a clear separation of 3 stages.

* stage 1 (`ldso/dlstart.c`): only relative relocations are applied. This allows static variables can be accessed.
* stage 2 `__dls2`: This applies non-relative relocations.
* stage 2b `__dls2b`: Set up thread pointer with a TLS stub.
* stage 3 `__dls3`: Load the executable and immediately loaded shared objects. Apply relocations and possibly relocate rtld/libc itself again for possible symbol interposition (e.g. `R_*_COPY`, interposed malloc implementation).

Each stage uses a PC-relative code sequence to load the address of the next stage entry point, and then jump to it.
This serves as a strong compiler barrier preventing code reordering.

(In glibc, `elf/rtld.c` `ELF_DYNAMIC_RELOCATE (&bootstrap_map, NULL, 0, 0, 0);` is kinda like musl's stage 1 plus stage 2.)

Stage 1 computes the entry of stage 2 with `GETFUNCSYM(&dls2, __dls2, base+dyn[DT_PLTGOT]);` where `GETFUNCSYM` is defined for every port:

```c
// arch/m68k/reloc.h
#define GETFUNCSYM(fp, sym, got) __asm__ ( \
	".hidden " #sym "\n" \
	"lea " #sym "-.-8,%0 \n" \
	"lea (%%pc,%0),%0 \n" \
	: "=a"(*fp) : : "memory" )

// arch/powerpc/reloc.h
#define GETFUNCSYM(fp, sym, got) __asm__ ( \
	".hidden " #sym " \n" \
	"	bl 1f \n" \
	"	.long " #sym "-. \n" \
	"1:	mflr %1 \n" \
	"	lwz %0, 0(%1) \n" \
	"	add %0, %0, %1 \n" \
	: "=r"(*(fp)), "=r"((int){0}) : : "memory", "lr" )
```

This approach is elegant. It even allows a static or hidden function call with a dynamic relocation, though I haven't found such an architecture in my testing.
