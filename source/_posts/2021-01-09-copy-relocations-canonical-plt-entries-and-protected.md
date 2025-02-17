layout: post
title: Copy relocations, canonical PLT entries and protected visibility
author: MaskRay
tags: [binutils,gcc,llvm]
---

Updated in 2022-09.

Background:

* `-fno-pic` can only be used by executables. On most platforms and architectures, direct access relocations are used to reference external data symbols.
* `-fpic` can be used by both executables and shared objects. Windows has `__declspec(dllimport)` but most other binary formats allow a default visibility external data to be resolved to a shared object, so generally direct access relocations are disallowed.
* `-fpie` was introduced as a mode similar to `-fpic` for ELF: the compiler can make the assumption that the produced object file can only be used by executables, thus all definitions are non-preemptible and thus interprocedural optimizations can apply on them.

For

```c
extern int a;
int *foo() { return &a; }
```

`-fno-pic` typically produces an absolute relocation (a PC-relative relocation can be used as well). On ELF x86-64 it is usually `R_X86_64_32` in the position dependent small code model.
If `a` is defined in the executable (by another translation unit), everything works fine.
If `a` turns out to be defined in a shared object, its real address will be non-constant at link time. Either action needs to be taken:

* Emit a dynamic relocation in every use site. Text sections are usually non-writable. A dynamic relocation applied on a non-writable section is called a text relocation.
* Emit a single copy relocation. Copy relocations only work for executables. The linker obtains the size of the symbol, allocates the bytes in .bss (this may make the object writable. On ld.lld a readonly area may be picked.), and emit an `R_*_COPY` relocation. All references resolve to the new location.

Multiple text relocations are even less acceptable, so on ELF a copy relocation is generally used.
Here is a nice description from [Rich Felker](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=55012): "Copy relocations are not a case of overriding the definition in the abstract machine, but an implementation detail used to support data objects in shared libraries when the main program is non-PIC."

Copy relocations have drawbacks:

* Break page sharing. The text segment cannot be shared by two processes using the same code.
* Make the symbol properties (e.g. size) part of ABI.
* If the shared object is linked with `-Bsymbolic` or `--dynamic-list` defines a data symbol copy relocated by the executable, the address of the symbol may be different in the shared object and in the executable. <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65888>

GNU ld and gold add copied data to `.bss` which makes read-only data writable.
ld.lld prefers `.bss.rel.ro` which is part of `PT_GNU_RELRO` to prevent permission escalation.

What went poorly was that `-fno-pic` code had no way to avoid copy relocations on ELF.
Traditionally copy relocations could only occur in `-fno-pic` code. A GCC 5 change made this possible for x86-64. Please read on.

## x86-64: copy relocations and `-fpie`

`-fpic` using GOT indirection for external data symbols has costs. Making `-fpie` similar to `-fpic` in this regard incurs costs if the data symbol turns out to be defined in the executable.
Having the data symbol defined in another translation unit linked into the executable is very common, especially if the vendor uses fully/mostly statically linking mode.

In GCC 5, ["x86-64: Optimize access to globals in PIE with copy reloc"](https://gcc.gnu.org/git/?p=gcc.git&a=commit;h=77ad54d911dd7cb88caf697ac213929f6132fdcf)
started to use direct access relocations for external data symbols on x86-64 in `-fpie` mode.

```c
extern int a;
int foo() { return a; }
```

* GCC<5: `movq a@GOTPCREL(%rip), %rax; movl (%rax), %eax` (8 bytes)
* GCC>=5: `movl a(%rip), %eax` (6 bytes)

This change is actually useful for architectures other than x86-64 but is never implemented for other architectures.
However, there was a serious flaw: the change was implemented as a configure-time choice (`HAVE_LD_PIE_COPYRELOC`), defaulting to such a behavior if ld supports PIE copy relocations (most binutils installations).
Many people would be fine if this were implemented as an opt-in.

Clang addressed the inflexible configure-time choice via an opt-in option `-mpie-copy-relocations` (<https://reviews.llvm.org/D19996>).

I noticed that:

* The option can be used for `-fno-pic` code as well to prevent copy relocations on ELF. This is occasionally what users want (if their shared objects use `-Bsymbolic` and export data symbols (usually undesired from API perspectives but can avoid costs at times)), and they switch from `-fno-pic` to `-fpic` just for this purpose.
* The option name should describe the code generation behavior, instead of the inferred behavior at the linking stage on a particular binary format.
* The option does not need to tie to ELF.
  + On COFF, the behavior is like always `-fdirect-access-external-data`. `__declspec(dllimport)` is needed to enable indirect access.
  + On Mach-O, the behavior is like `-fdirect-access-external-data` for `-fno-pic` (only available on arm) and the opposite for `-fpic`.
* H.J. Lu introduced `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` as GOT optimization to x86-64 psABI. This is great! With the optimization, GOT indirection can be optimized, so the incurred cost is very low now.

So I proposed an alternative option `-f[no-]direct-access-external-data`: <https://reviews.llvm.org/D92633> <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98112>.
My wish on the GCC side is to drop `HAVE_LD_PIE_COPYRELOC` and (x86-64) default to GOT indirection for external data symbols in `-fpie` mode.
I have a patch but as of November, I am quite frustrated that they don't listen after numerous pings [[PATCH] x86-64: Remove HAVE_LD_PIE_COPYRELOC](https://gcc.gnu.org/pipermail/gcc-patches/2021-November/582987.html).

Please keep in mind that `-f[no-]semantic-interposition` is for definitions while `-f[no-]direct-access-external-data` is for undefined data symbols.
GCC 5 introduced `-fno-semantic-interposition` to use local aliases for references to definitions in the same translation unit.

## `STV_PROTECTED`

Now let's consider how `STV_PROTECTED` comes into play. Here is the generic ABI definition:

> A symbol defined in the current component is protected if it is visible in other components but not preemptable, meaning that any reference to such a symbol from within the defining component must be resolved to the definition in that component, even if there is a definition in another component that would preempt by the default rules. A symbol with STB_LOCAL binding may not have STV_PROTECTED visibility. If a symbol definition with STV_PROTECTED visibility from a shared object is taken as resolving a reference from an executable or another shared object, the SHN_UNDEF symbol table entry created has STV_DEFAULT visibility.

A non-local `STV_DEFAULT` defined symbol is by default preemptible in a shared object on ELF. `STV_PROTECTED` can make the symbol non-preemptible.
You may have noticed that I use "preemptible" while the generic ABI uses "preemptable" and LLVM IR uses `dso_preemptable`.
Both forms work. "preemptible" is my preference because it is more common.

### Protected data symbols and copy relocations

Many folks consider that copy relocations are best-effort support provided by the toolchain.
`STV_PROTECTED` is intended as an optimization and the optimization can error out if it can't be done for whatever reason.
Since copy relocations are unacceptable in a non-small and ever-increasing number of environments, it is natural to think that we should just disallow copy relocations on protected data symbols.

However, GCC 5 and GNU ld 2.26 made changes to enable copy relocations on protected data symbols for i386 and x86-64 (<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65248> for GCC x86 change).
Let's review the requirement.

Direct accesses to a protected symbol in a shared object cannot be allowed, because the executable and the shared object will operate on different copies.
Therefore, in `-fpic` mode GOT-generating relocations have to be used, just in case the symbols may be copy relocated.
For the following code, the assembly for `b.c` uses a GOT-generating relocation since GCC 5. Previously a direct access was used.
```c
// b.c
__attribute__((visibility("protected"))) int var;
int foo() { return var; }
// -fpic: since GCC 5, GOT on at least aarch64, arm, i386, x86-64

// a.c
extern int var;
int main() { return var; }
```
```sh
gcc -fuse-ld=bfd -fpic -shared b.c -o b.so
gcc -fuse-ld=bfd -no-pie -fno-pic a.c ./b.so
```

When linking `b.o` into `b.so`, GNU ld considers a GOT-generating relocation to a protected data symbol non-local and emits an `R_*_GLOB_DAT` dynamic relocation (instead of `R_*_RELATIVE`).

At run-time, rtld binds the `R_*_GLOB_DAT` relocation in `b.so` to the executable, therefore allowing the executable and `b.so` to access the same copy.

In summary, this mechanism requires that (a) the compiler emits GOT-generating relocations (b) the linker produces `R_*_GLOB_DAT`.
If the linker produces `R_*_RELATIVE` or no dynamic relocation for `var`, the executable and `a.so` will access two different copies.

Say both `a.so` and `b.so` define a protected data symbol `var` and the executable copy relocates `var`.
It is unclear whether accesses in `b.so` should bind to the executable but the glibc change ["Add ELF_RTYPE_CLASS_EXTERN_PROTECTED_DATA to x86"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=62da1e3b00b51383ffa7efc89d8addda0502e107) made a decision.
When `ELF_RTYPE_CLASS_EXTERN_PROTECTED_DATA` is defined, the `R_*_GLOB_DAT` in `a.so` binds to the executable while `R_*_GLOB_DAT` in `b.so` binds to itself.

["[AArch64][BZ #17711] Fix extern protected data handling"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=0910702c4d2cf9e8302b35c9519548726e1ac489) and
["[ARM][BZ #17711] Fix extern protected data handling"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=3bcea719ddd6ce399d7bccb492c40af77d216e42) ported the behavior to arm and aarch64.
For glibc 2.36, I have reverted the changes and landed [elf: Remove ELF_RTYPE_CLASS_EXTERN_PROTECTED_DATA](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=de38b2a343e6d64b95c50004943d6107a9e380d0)

### Reflection on protected data symbols and copy relocations

On the GCC side, in `-fpic` mode, using GOT-generating relocations when accessing a protected variable subverts the point using the protected visibility.
The unneeded pessimization is the foremost complaint.
The pessimization applies to all ports with `#define TARGET_BINDS_LOCAL_P default_binds_local_p_2`.
aarch64 moved to `default_binds_local_p_2` accidentally by <https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=cbddf64c0243816b45e6680754a251c603245dbc>.

For GCC<5 (and all versions of Clang), direct accesses to protected variables are produced in `-fpic` code.
Mixing such object files can still silently break copy relocations on protected data symbols.
Therefore, GNU ld made the controversial change <https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;a=commit;h=ca3fe95e469b9daec153caa2c90665f5daaec2b5> to error in `-shared` mode.
```text
% cat a.s
leaq var(%rip), %rax

.data
.global var
.protected var
var:
% gcc -fuse-ld=bfd -shared a.s
/usr/bin/ld.bfd: /tmp/ccchu3Xo.o: relocation R_X86_64_PC32 against protected symbol `var' can not be used when making a shared object
/usr/bin/ld.bfd: final link failed: bad value
collect2: error: ld returned 1 exit status
```

This led to a heated discussion <https://sourceware.org/legacy-ml/binutils/2016-03/msg00312.html>.
Swift folks noticed this <https://bugs.swift.org/browse/SR-1023> and their reaction was to switch from GNU ld to gold.

(
GNU ld tried repairing this. ["x86: Clear extern_protected_data for GNU_PROPERTY_NO_COPY_ON_PROTECTED"](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=73784fa565bd66f1ac165816c03e5217b7d67bbc) introduced `GNU_PROPERTY_NO_COPY_ON_PROTECTED`.
With this property, `ld -shared` will not error for ``relocation R_X86_64_PC32 against protected symbol `var' can not be used when making a shared object``.
However, nobody ever uses `GNU_PROPERTY_NO_COPY_ON_PROTECTED`.
)

Thankfully the damange is limited to x86. GNU ld's aarch64 port does not have the diagnostic.

Now, let's move focus outside GCC and GNU ld.

Neither gold nor ld.lld allows copy relocations on protected data symbols.
(gold misses some cases: <https://sourceware.org/bugzilla/show_bug.cgi?id=19823>, though.)

Clang always treats protected similar to hidden/internal and doesn't use GOT-generating relocations, therefore the copy relocation on protected data symbol scheme never works with Clang.

powerpc64 ELFv2 is interesting: TOC indirection (TOC is a variant of GOT) is used everywhere, data symbols normally have no direct access relocations.

### Protected function symbols and canonical PLT entries

```c
// b.c
__attribute__((visibility("protected"))) void *foo() {
  return (void *)foo;
}
```

GNU ld's x86 ported has rejected this for a long time, at least since binutils 2.11.
On many other architectures including powerpc the code is supported.

```
% gcc -m32 -fpic -shared -fuse-ld=bfd b.c
/usr/bin/ld.bfd: /tmp/cc3Ay0Gh.o: relocation R_X86_64_PC32 against protected symbol `foo' can not be used when making a shared object
/usr/bin/ld.bfd: final link failed: bad value
collect2: error: ld returned 1 exit status
% aarch64-linux-gnu-gcc -fpic -shared -fuse-ld=bfd b.c
/usr/lib/gcc-cross/aarch64-linux-gnu/11/../../../../aarch64-linux-gnu/bin/ld.bfd: /tmp/ccJf24eh.o: relocation R_AARCH64_ADR_PREL_PG_HI21 against symbol `foo' which may bind externally can not be used when making a shared object; recompile with -fPIC
/tmp/ccJf24eh.o: in function `foo':
b.c:(.text+0x0): dangerous relocation: unsupported relocation
collect2: error: ld returned 1 exit status
```

The rejection is based on the assumption: if such a shared object is used with `-fno-pic` code (or GCC x86-64 `-fpie` code with `HAVE_LD_PIE_COPYRELOC`) taking the address of `foo`, pointer equality will break.

I think the error is placed at the wrong place. The error should be placed at the executable when there is direct access relocation referencing `foo`.
I.e. a canonical PLT entry is incompatible with a shared `STV_PROTECTED` definition (gold and ld.lld report errors), but there is no reason disallowing other cases (where nothing will break).

The aarch64 port has allowed this since my [aarch64: Allow PC-relative relocations against protected STT_FUNC for -shared](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=83c325007c5599fa9b60b8d5f7b84842160e1d1b) (milestone: binutils 2.39).
The x86 port has allowed this since my [x86: Make protected symbols local for -shared](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=d19a265487eda186b6977d9d15648cda9fad3298) (milestone: binutils 2.39).

### Branch instructions on x86

On many architectures, a branch instruction uses a branch specific relocation type (e.g. `R_AARCH64_CALL26`, `R_PPC64_REL24`, `R_RISCV_CALL_PLT`).
This is great: the relocation conveys the idea that the address is insignificant and the linker can arrange for a regular PLT if the symbol turns out to be external.

However, on i386, a branch in `-fno-pic` code emits an `R_386_PC32` relocation, which is indistinguishable from an address taken operation.
If the symbol turns out to be external, the linker has to employ a tricky called "canonical PLT entry" (`st_shndx=0, st_value!=0`).
The term is a parlance within a few ld.lld developers, but not broadly adopted.

```c
// a.c
extern void foo(void);
int main() { foo(); }
```

```text
% gcc -m32 -shared -fuse-ld=bfd -fpic b.c -o b.so
% gcc -m32 -fno-pic -no-pie -fuse-ld=lld a.c ./b.so

% gcc -m32 -fno-pic a.c ./b.so -fuse-ld=lld
ld.lld: error: cannot preempt symbol: foo
>>> defined in ./b.so
>>> referenced by a.c
>>>               /tmp/ccDGhzEy.o:(main)
collect2: error: ld returned 1 exit status

% gcc -m32 -fno-pic -no-pie a.c ./b.so -fuse-ld=bfd
# canonical PLT entry; foo has different addresses in a.out and b.so.
% gcc -m32 -fno-pic -pie a.c ./b.so -fuse-ld=bfd
/usr/bin/ld.bfd: /tmp/ccZ3Rl8Y.o: warning: relocation against `foo' in read-only section `.text'
/usr/bin/ld.bfd: warning: creating DT_TEXTREL in a PIE
% gcc -m32 -fno-pic -pie a.c ./b.so -fuse-ld=bfd -z text
/usr/bin/ld.bfd: /tmp/ccUv8wXc.o: warning: relocation against `foo' in read-only section `.text'
/usr/bin/ld.bfd: read-only segment has dynamic relocations
collect2: error: ld returned 1 exit status
```

This used to be a problem for x86-64 as well, until ["x86-64: Generate branch with PLT32 relocation"](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=bd7ab16b4537788ad53521c45469a1bdae84ad4a) changed `call/jmp foo` to emit `R_X86_64_PLT32` instead of `R_X86_64_PC32`.
Note: (`-fpie/-fpic`) `call/jmp foo@PLT` always emits `R_X86_64_PLT32`.

The relocation type name is a bit misleading, `_PLT32` does not mean that a PLT will always be created.
Rather, it is optional: the linker can resolve `_PLT32` to any place where the function will be called.
If the symbol is preemptible, the place is usually the PLT entry.
If the symbol is non-preemptible, the linker can convert `_PLT32` into `_PC32`.
A function symbol can be either branched or taken address.
For an address taken operation, the function symbol is used in a manner similar to a data symbol. `R_386_PLT32` cannot be used.
ld.lld and gold will just reject the link if text relocations are disabled.

There are two issues disallowing the switch from `R_386_PC32` to `R_386_PLT32` <https://sourceware.org/bugzilla/show_bug.cgi?id=27169>.
First is that one can do something like:
```
gcc -m32 -fno-pic -shared -z notext a.c -o a.so
```
The linker will produce a `R_386_PC32` dynamic relocation. According to H.J., some shared objects intentionally do this for optimization.
This is a dark art leveraging text relocations for an unusual PC-relative relocation type.

Second is related to a not-so-useful diagnostic. Please read on.

### Non-default visibility ifunc and `R_386_PC32`

For a call to a hidden function declaration, the compiler produces an `R_386_PC32` relocation.
The relocation is an indicator that EBX may not be set up.

If the declaration refers to an ifunc definition, the linker will resolve the `R_386_PC32` to an IPLT entry.
For `-pie` and `-shared` links, the IPLT entry references EBX.
If the call site does not set up EBX to be `_GLOBAL_OFFSET_TABLE_`, the IPLT call will be incorrect.

GNU ld has implemented a diagnostic (["i686 ifunc and non-default symbol visibility"](https://sourceware.org/bugzilla/show_bug.cgi?id=20515)) to catch the problem.
If we change `call/jmp foo` to always use `R_386_PLT32`, such a diagnostic will be lost.

Can we change the compiler to emit `call/jmp foo@PLT` for default visibility function declarations?
If the compiler emits such a modifier but does not set up EBX, the ifunc can still be non-preemptible
(e.g. hidden in another translation unit or -Bsymbolic) and we will still have a dilemma.

Personally, I think avoiding a canonical PLT entry is more useful than a ld ifunc diagnostic.
i386 ABI is legacy and the x86 maintainer will not make the change, though.

## Summary

I hope the above give an overview to interested readers. Symbol interposition is subtle.
One has to think about all the factors related to symbol interposition and the relevant toolchain fixes are like a whack-a-mole game.
I appreciate all the prior discussions and I believe many unsatisfactory things can be fixed in a quite backward-compatible way.

Some features are inherently incompatible. We make the trade-off in favor of more important features.
Here are two things that should not work.

* Copy relocations on protected data symbols.
* Canonical PLT entries on protected function symbols. With the `R_386_PLT32` change, this issue will only affect function pointers.

However, if `-fpie` or `-fno-direct-access-external-data` is specified, both limitations will be circumvented.
In [June 2021](https://gcc.gnu.org/pipermail/gcc-patches/2021-June/573434.html), I said that I am fine with a different option name, e.g. `-fno-direct-extern-access`.

People sometimes simply just say: "protected visibility does not work." I'd argue that Clang+gold/ld.lld works quite well.
The things on GCC+GNU ld side are inconsistent, though.

Some milestones to make the ELF world better:

* eliminate copy relocations for x86-64 `-fpie`
  + GCC x86-64: default to GOT indirection for external data symbols in `-fpie` mode. Patch: <https://gcc.gnu.org/pipermail/gcc-patches/2021-May/570139.html> As of 2021-11, I am quite frustrated that they don't listen after numerous pings.
* eliminate canonical PLT entries for `-fno-pic` (all architectures)
  + implement <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100593>. Optionally add `-fdirect-access-external-function` for users who want the original behavior.
  + make ld default to warn for canonical PLT entries (`st_shndx==0, st_value!=0`). On i386 this depends on `R_386_PLT32` (see below)
  + make glibc ld.so warn for canonical PLT entries
* eliminate copy relocations for `-fno-pic` (all architectures; if GOT indirection cost is a concern, opt out the legacy architectures (i386/ppc32))
  + implement `-fno-direct-access-external-data` [PR98112](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98112)
  + make `-fno-pic` default to `-fno-direct-access-external-data` for most architectures. Some users may prefer performance despite copy relocations without x86-64 GOTPCRELX/ppc64 TOC optimization/AArch64 GOT optimization. They can opt out.
  + make ld default to warn for `R_*_COPY`
  + make glibc ld.so warn for `R_*_COPY`
* GCC: treat `STV_PROTECTED` similar to `STV_HIDDEN`
  + GCC aarch64/arm/x86/...: allow direct access relocations on protected symbols in `-fpic` mode.
* GNU ld: treat `STV_PROTECTED` similar to `STV_DEFAULT` in `-Bsymbolic` mode
  + disallow direct access relocations on protected symbols, i.e. no copy relocation/canonical PLT
  + done for [aarch64](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=90b7a5df152a64d2bea20beb438e8b81049a5c30)
  + done for [x86](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=d19a265487eda186b6977d9d15648cda9fad3298)
* glibc: warn for direct access to a protected symbol
  + done in [elf: Refine direct extern access diagnostics to protected symbol](https://sourceware.org/git/?p=glibc.git;a=commit;h=7374c02b683b7110b853a32496a619410364d70b) (glibc 2.35)

After elimination of canonical PLT entries, we can safely enable distribution-wide default `ld -Bsemantic-non-weak-functions`.
This will improve performance for lots of software, especially for short-lived processes where relocation symbol lookup takes a significant portion.
See [ELF interposition and -Bsymbolic](/blog/2021-05-09-elf-interposition-and-bsymbolic) for details.

A weak action item

* GCC or GNU as i386: emit `R_386_PLT32` for branches to undefined function symbols. My preference on this is weak.
* Note, ld is not the only consumer of `R_386_PLT32`. The Linux kernel has code resolving relocations and has been fixed (patch merged in mainline: <https://github.com/ClangBuiltLinux/linux/issues/1210>).

The "copy relocations on protected data symbols" scheme has some fragile support with GNU ld and glibc on x86 (acked by multiple glibc maintainers).
It definitely did not work before circa 2016, and should not work in the future.
This scheme does not work with gold or ld.lld. GNU ld for non-x86 architectures don't work.
I think no software will break if we drop the fragile glibc support.

I'll conclude this article with random notes on other binary formats:

Windows/COFF `__declspec(dllimport)` gives us a different perspecitive how external references can be designed.
The annotation is verbose but differentiates the two cases (1) the symbol has to be defined in the same linkage unit (2) the symbol can be defined in another linkage unit.
If we lift the "the symbol visibility is decided by the most constrained visibility" requirement for protected->default,
a COFF undefined/defined symbol is quite like a protected undefined/defined symbol in ELF. `__declspec(dllimport)` gives the undefined symbol default visibility (i.e. the LLVM IR `dllimport` is redundant). `__declspec(dllexport)` is something which cannot be modeled with the existing ELF visibilities.

For an undefined variable, Mach-O uses `__attribute__((visibility("hidden")))` to say "a definition must be available in another translation unit in the same linkage unit" but does not actually mark the undefined symbol in the binary format.
COFF uses `__declspec(dllimport)` to convey this. In ELF, `__attribute__((visibility("hidden")))` additionally makes the undefined symbol unexportable.
The Mach-O notation actually resembles COFF: it can be exported by the definition in another translation unit.
I think its usage of hidden visibility is a misnomer: it would be more appropriately mapped to LLVM IR protected instead of hidden.

## Appendix

For a STB_GLOBAL/STB_WEAK symbol,

STV_DEFAULT: both compiler & linker need to assume such symbols can be preempted in -fpic mode. The compiler emits GOT indirection by default.
GCC -fno-semantic-interposition uses local aliases on defined non-weak function symbols for x86 (unimplemented in other architectures).
Clang -fno-semantic-interposition uses local aliases on defined non-weak symbols (both function and data) for x86.

STV_PROTECTED: GCC -fpic uses GOT indirection for data symbols, regardless of defined or undefined. This pessimization is to make a misfeature "copy relocation on protected data symbol" work. Clang code generation treats STV_PROTECTED the same way as STV_HIDDEN.

STV_HIDDEN: non-preemptible, regardless of defined or undefined. The compiler suppresses GOT indirection, unless undefined STB_WEAK.

For defined symbols, -fno-pic/-fpie can avoid GOT indirection for STV_DEFAULT (and GCC STV_PROTECTED).
-fvisibility=hidden can change visibility.

For undefined symbols, -fpie/-fpic use GOT indirection by default. Clang -fno-direct-access-external-data (discussed in my article) can avoid GOT indirection. If you -fpic -fno-direct-access-external-data & ld -shared, you'll need additional linker options to make the linker know defined non-STB_LOCAL STV_DEFAULT symbols are non-preemptible.
