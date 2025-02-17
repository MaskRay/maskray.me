layout: post
title: GNU indirect function
author: MaskRay
tags: [glibc,binutils]
---

Updated in 2024-04.

GNU indirect function (ifunc) is a mechanism making a direct function call resolve to an implementation picked by a resolver.
It is mainly used in glibc but has adoption in FreeBSD.

<!-- more -->

For some performance critical functions, e.g. memcpy/memset/strcpy, glibc provides multiple implementations optimized for different architecture levels.
The application just uses `memcpy(...)` which compiles to `call memcpy`.
The linker will create a PLT for `memcpy` and produce an associated special dynamic relocation referencing the resolver symbol/address.
During relocation resolving at runtime, the return value of the resolver will be placed in the GOT entry and the PLT entry will load the address.

On Mach-O, there is a similar feature called `N_SYMBOL_RESOLVER` (assembly directive `.symbol_resolver`).
Apple engineers [regretted](https://reviews.llvm.org/D139163#3993795) adding it.

## Representation

ifunc has a dedicated symbol type `STT_GNU_IFUNC` to mark it different from a regular function (`STT_FUNC`).
The value 10 is in the OS-specific range (10~12). `readelf -s` tell you that the symbol is ifunc if OSABI is `ELFOSABI_GNU` or `ELFOSABI_FREEBSD`.

On Linux, by default GNU as uses `ELFOSABI_NONE` (0). If ifunc is used, the OSABI will be changed to `ELFOSABI_GNU`.
Similarly, GNU ld sets the OSABI to `ELFOSABI_GNU` if ifunc is used.
gold does not do this [PR17735](https://sourceware.org/bugzilla/show_bug.cgi?id=17735).

Things are loose in LLVM. The integrated assembler and ld.lld do not set `ELFOSABI_GNU`.
(From 16.0 onawards, the integrate assembler will set `ELFOSABI_GNU`.)
Currently the only problem I know is the `readelf -s` display. Everything else works fine.

### Assembler behavior

In assembly, you can assign the type `STT_GNU_IFUNC` to a symbol via `.type foo, @gnu_indirect_function`.
An ifunc symbol is typically `STB_GLOBAL`.

In the object file, `st_shndx` and `st_value` of an `STT_GNU_IFUNC` symbol indicate the resolver.
After linking, if the symbol is still `STT_GNU_IFUNC`, its `st_value` field indicates the resolver address in the linked image.

Assemblers usually convert relocations referencing a local symbol to reference the section symbol, but this behavior needs to be inhibited for `STT_GNU_IFUNC`.

### Example

```sh
cat > b.s <<'eof'
.global ifunc
.type ifunc, @gnu_indirect_function
.set ifunc, resolver

resolver:
  leaq impl(%rip), %rax
  ret

impl:
  movq $42, %rax
  ret
eof

cat > a.c <<'eof'
int ifunc(void);
int main() { return ifunc(); }
eof

cc a.c b.s  # ifunc is non-preemptible
./a.out  # exit code 42
```

GNU as makes transitive aliases to an `STT_GNU_IFUNC` ifunc as well.

```asm
.type foo,@gnu_indirect_function
.set foo, foo_resolver

.set foo2, foo
.set foo3, foo2
```

GCC and Clang support a function attribute which emits `.type ifunc, @gnu_indirect_function; .set ifunc, resolver`:

```c
static int impl(void) { return 42; }
static void *resolver(void) { return impl; }
void *ifunc(void) __attribute__((ifunc("resolver")));
int main() { ifunc(); }
```

Static linking with glibc and `gcc -fprofile-generate` can lead to crashes on some ports.
On these ports the instrumented resolver attempts to access TLS before it's properly set up by `__libc_setup_tls`.
`__libc_setup_tls` calls ifunc functions `memcpy`, so we cannot fix the interaction by moving `ARCH_SETUP_TLS ()` before `ARCH_SETUP_IREL ()` in `csu/libc-start.c`.

```
% gcc -fprofile-generate -static a.c -o a; ./a
[1]    2657463 segmentation fault  ./a
```

This is a feature incompatibility issue <https://gcc.gnu.org/PR114115> reported by the [XZ Backdoor](https://www.openwall.com/lists/oss-security/2024/03/29/4) author.

As a workaround, annotate the resolver with the `no_profile_instrument_function` attribute.
```c
__attribute__((no_profile_instrument_function))
static void *resolver(void) { return impl; }
```

## Preemptible ifunc

A preemptible ifunc call is no different from a regular function call from the linker perspective.

The linker creates a PLT entry, reserves an associated GOT entry, and emits an `R_*_JUMP_SLOT` relocation resolving the address into the GOT entry.
The PLT code sequence is the same as a regular PLT for `STT_FUNC`.

If the ifunc is defined within the module, the symbol type in the linked image is `STT_GNU_IFUNC`, otherwise (defined in a DSO), the symbol type is `STT_FUNC`.

The difference resides in the loader.

At runtime, the relocation resolver checks whether the `R_*_JUMP_SLOT` relocation refers to an ifunc.
If it does, instead of filling the GOT entry with the target address, the resolver calls the target address as an indirect function, with ABI specified additional parameters (hwcap related), and places the return value into the GOT entry.

```sh
cc -shared b.s -o b.so
cc a.c ./b.so -o a  # the symbol `ifunc` is preemptible
```

## Non-preemptible ifunc

The non-preemptible ifunc case is where all sorts of complexity come from.

First, the `R_*_JUMP_SLOT` relocation type cannot be used in some cases:

* A non-preemptible ifunc may not have a dynamic symbol table entry. It can be local. It can be defined in the executable without the need to export.
* A non-local `STV_DEFAULT` symbol defined in a shared object is by default preemptible. Using `R_*_JUMP_SLOT` for such a case will make the ifunc look like preemptible.

Therefore a new relocation type `R_*_IRELATIVE` was introduced. There is no associated symbol and the address indicates the resolver.
```text
R_*_RELATIVE: B + A
R_*_IRELATIVE: call (B + A) as a function
R_*_JUMP_SLOT: S
```

When an `R_*_JUMP_SLOT` can be used, there is a trade-off between `R_*_JUMP_SLOT` and `R_*_IRELATIVE`: an `R_*_JUMP_SLOT` can be lazily resolved but needs a symbol lookup.
Currently powerpc can use `R_PPC64_JMP_SLOT` in some cases [PR27203](https://sourceware.org/bugzilla/show_bug.cgi?id=27203).

A PLT entry is needed for two reasons:

* The call sites emit instructions like `call foo`. We need to forward them to a place to perform the indirection. Text relocations are usually not an option (exception: <a name="ifunc-noplt">{ifunc-noplt}</a>).
* If the ifunc is exported, we need a place to mark its canonical address.

Such PLT entries are sometimes referred to as IPLT. They are placed in the synthetic section `.iplt`.
In GNU ld, `.iplt` will be placed in the output section `.plt`. In ld.lld, I decided that `.iplt` is better <https://reviews.llvm.org/D71520>.

On many architectures (e.g. AArch64/PowerPC/x86), the PLT code sequence is the same as a regular PLT, but it could be different.

On x86-64, the code sequence is:
```asm
jmp *got(%rip)
pushq $0
jmp .plt
```

Since there is no lazy binding, `pushq $0; jmp .plt` are not needed. However, to make all PLT entries of the same shape to simplify linker implementations and facilitate analyzers, it is find to keep it this way.

### PowerPC32 `-msecure-plt` IPLT

As a design to work around the lack of PC-relative instructions, PowerPC32 uses multiple GOT sections, one per file in `.got2`.
To support multiple GOT pointers, the addend on each `R_PPC_PLTREL24` reloc will have the offset within `.got2`.

`-msecure-plt` has small/large PIC differences.

* `-fpic/-fpie`: `R_PPC_PLTREL24 r_addend=0`. The call stub loads an address relative to `_GLOBAL_OFFSET_TABLE_`.
* `-fPIC/-fPIE`: `R_PPC_PLTREL24 r_addend=0x8000`. (A partial linked object file may have an addend larger than 0x8000.) The call stub loads an address relative to .got2+0x8000.

If a non-preemptible ifunc is referenced in two object files, in `-pie`/`-shared` mode, the two object files cannot share the same IPLT entry.
When I added non-preemptible ifunc support for PowerPC32 to ld.lld <https://reviews.llvm.org/D71621>, I did not handle this case.

### x86-32 EBX IPLT

The symbol cannot be `STV_HIDDEN` at a call site.
If `STV_HIDDEN`, the compiler may not set EBX before the call instruction, and the IPLT may crash.

### .rela.dyn vs .rela.plt

ld.lld placed `R_*_IRELATIVE` in the `.rela.plt` section because many ports of GNU ld behaved this way for function calls.
This turns out to be used to make PLT calls in an ifunc resolver work. See below.

While implementing ifunc for PowerPC, I noticed that GNU ld powerpc actually places `R_*_IRELATIVE` in `.rela.dyn` and glibc powerpc does not actually support `R_*_IRELATIVE` in `.rela.plt`.
This makes a lot of sense to me because `.rela.plt` normally just contains `R_*_JUMP_SLOT` which can be lazily resolved.
ifunc relocations need to be eagerly resolved so `.rela.plt` was a misplace.
Therefore I changed ld.lld to use `.rela.dyn` in <https://reviews.llvm.org/D65651>.

Within `.rela.dyn` or `.rela.plt, `R_*_IRELATIVE` relocations are placed at the end. When glibc rtld processes the `R_*_IRELATIVE` relocations, other relocations like `R_*_RELATIVE` and `R_*_GLOB_DAT` have been processed. This allows an ifunc resolver to take the address of the implementation function.

### `__rela_iplt_start` and `__rela_iplt_end`

[BZ #27164](https://sourceware.org/bugzilla/show_bug.cgi?id=27164)

A statically linked position dependent executable traditionally had no dynamic relocations.
However, ifunc may lead to `R_*_IRELATIVE` relocations.

In glibc, `csu/libc-start.c` has undefined weak references of `__rela_iplt_start`/`__rela_iplt_end`.
In static non-pie mode, the loader resolves ifunc relocations within the range `[__rela_iplt_start,__rela_iplt_end)`.
In static pie mode, however, static pie uses self-relocation (`_dl_relocate_static_pie`) to take care of `R_*_IRELATIVE`.
It is expected that `__rela_iplt_start==__rela_iplt_end`, otherwise some ifunc relocations may be repeatly applied, causing SIGSEGV from `ARCH_SETUP_IREL`.

GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in `-pie` mode.
For `{gcc,clang} -fuse-ld=bfd -static-pie -fpie` built static pie, the scheme above works because GNU ld leaves `__rela_iplt_start=__rela_iplt_end=0` as unresolved weak symbols.
Before 13.0.0, ld.lld defined `__rela_iplt_start`/`__rela_iplt_end` for `-pie` and therefore broke glibc loader's assumption, causing a program to crash.

I made a good argument and also pointed out that the `-no-pie` special case caused an [unneeded difference](https://sourceware.org/pipermail/libc-alpha/2021-January/121755.html) in the output of `diff -u =(ld.bfd --verbose) =(ld.bfd -pie --verbose)`.
My arguments were dismissed by "Ulrich and I designed/implemented IFUNC on x86 and for x86. I consider x86 implementation of IFUC as the gold standard."

In the end, I conceded and changed ld.lld to only define the two encapsulation symbols for `-no-pie` with grumbling.
Otherwise, `{gcc,clang} -fuse-ld=lld -static-pie` produced static pie would crash, even if the used glibc was built with GNU ld.

Interestingly, Android bionic turned out to rely on defined `__rela_iplt_start`/`__rela_iplt_end` for static pie.
<https://r.android.com/1809796> removed the reliance.

### Address significance

A non-GOT-generating non-PLT-generating relocation referencing a `STT_GNU_IFUNC` indicates a potential address-taken operation.
This is only allowed for non-preemptible ifunc.

With a function attribute, the compilers knows that a symbol indicates an ifunc and will avoid generating such relocations.
With assembly such relocations may be unavoidable.

In most cases the linker needs to convert the symbol type to `STT_FUNC` and create a special PLT entry, which is called a "canonical PLT entry" in ld.lld.
References from other modules will resolve to the PLT entry to keep pointer equality: the address taken from the defining module should match the address taken from another module.

This approach has pros and cons:

* With a canonical PLT entry, the resolver of a symbol is called only once. There is exactly one `R_*_IRELATIVE` relocation.
* If the relocation appears in a non-`SHF_WRITE` section, a text relocation can be avoided.
* Relocation types which are not valid dynamic relocation types are supported. GNU ld may error ``relocation R_X86_64_PC32 against STT_GNU_IFUNC symbol `ifunc' isn't supported``
* References will bind to the canonical PLT entry. A function call needs to jump to the PLT, loads the value from the GOT, then does an indirect call.

For a symbolic relocation type (a special case of absolute relocation types where the width matches the word size) like `R_X86_64_64`, when the addend is 0 and the section has the `SHF_WRITE` flag, the linker can emit an `R_X86_64_IRELATIVE`.
<https://reviews.llvm.org/D65995> dropped the case.

For the following example, GNU ld linked `a.out` calls `fff_resolver` three times while ld.lld calls it once.

```c
// RUN: split-file %s %t
// RUN: clang -fuse-ld=bfd -fpic %t/dso.c -o %t/dso.so --shared
// RUN: clang -fuse-ld=bfd %t/main.c %t/dso.so -o %t/a.out
// RUN: %t/a.out

//--- dso.c
typedef void fptr(void);
extern void fff(void);

fptr *global_fptr0 = &fff;
fptr *global_fptr1 = &fff;

//--- main.c
#include <stdio.h>

static void fff_impl() { printf("fff_impl()\n"); }
static int z;
void *fff_resolver() { return (char *)&fff_impl + z++; }

__attribute__((ifunc("fff_resolver"))) void fff();
typedef void fptr(void);
fptr *local_fptr = fff;
extern fptr *global_fptr0, *global_fptr1;

int main() {
  printf("local %p global0 %p global1 %p\n", local_fptr, global_fptr0, global_fptr1);
  return 0;
}
```

## `-z ifunc-noplt`

Mark Johnston introduced `-z ifunc-noplt` for FreeBSD <https://reviews.llvm.org/D61613>.
With this option, all relocations referencing `STT_GNU_IFUNC` will be emitted as dynamic relocations (if `.dynsym` is created).
The canonical PLT entry will not be used.

## Miscellaneous

GNU ld has implemented a diagnostic (["i686 ifunc and non-default symbol visibility"](https://sourceware.org/bugzilla/show_bug.cgi?id=20515)) to flag `R_386_PC32` referencing non-default visibility ifunc in `-pie` and `-shared` links.
This diagnostic looks like the most prominent reason blocking my proposal to use `R_386_PLT32` for `call/jump foo`.
See [Copy relocations, canonical PLT entries and protected visibility](2021-01-09-copy-relocations-canonical-plt-entries-and-protected) for details.

<https://sourceware.org/glibc/wiki/GNU_IFUNC> misses a lot of information.
There are quite a few arch differences.
I asked for clarification <https://sourceware.org/pipermail/libc-alpha/2021-January/121752.html>

glibc defines `__libc_ifunc_impl_list` to enumerate implementations for one ifunc.

## Dynamic loader

In glibc, `_dl_runtime_resolver` needs to save and restore vector and floating point registers.
ifunc resolvers add another reason that `_dl_runtime_resolver` cannot only use integer registers.
(The other reasons are that ld.so has string function calls which may use vectors and external calls to libc.so.)

For an architecture, `sysdeps/*/dl-irel.h` defines the signature of ifunc resolvers.

### Relocation resolving order

There are two parts: the order within a module and the order between two modules.

glibc rtld processes relocations in the reverse search order (reversed `l_initfini`) with a special case for the rtld itself.
The main executable needs to be processed the last to process `R_*_COPY`.
If A has an ifunc referencing B, generally B needs to be relocated before A.
Without ifunc, the resolving order of shared objects can be arbitrary.

Let's say we have the following dependency tree.
```
main
  dep1.so
    dep2.so
      dep3.so
        libc.so.6
      dep4.so
        dep3.so
        libc.so.6
    libc.so.6
  libc.so.6
```

`l_initfini` contains `main, dep1.so, dep2.so, dep4.so, dep3.so, libc.so.6, ld.so`.
The relocation resolving order is `ld.so (bootstrap), libc.so.6, dep3.so, dep4.so, dep2.so, dep1.so, main, ld.so`.

For ifunc, if the ifunc is defined in a processed module, it is fine. If the ifunc is defined in an unprocessed module, it may crash.

For an ifunc defined in an executable, calling it from a shared object can be problematic because the executable's relocations haven't been resolved.
The issue can be circumvented by converting the non-preemptible ifunc defined in the executable to `STT_FUNC`.
GNU ld's x86 port made the change [PR23169](https://sourceware.org/bugzilla/show_bug.cgi?id=23169).

Within a module, glibc rtld resolves relocations in order.
Assume that both `DT_RELA` (`.rela.dyn`) and `DT_PLTREL` (`.rela.plt`) are present, glibc logic is like the following:
```c
// Simplified from elf/dynamic-link.h
ranges[0] = {DT_RELA, DT_RELASZ, 0};
ranges[1] = {DT_JMPREL, DT_PLTRELSZ, do_lazy};
if (!do_lazy && ranges[0].start + ranges[0].size == ranges[1].start) { // the equality operator is always satisfied in practice
  ranges[0].size += size;
  ranges[1] = {};
}
for (int ranges_index = 0; ranges_index < 2; ++ranges_index)
  elf_dynamic_do_Rela (... ranges[ranges_index]);
```

`elf_dynamic_do_Rela`:
```c
// Simplified from elf/dl-rel.h
// Handle RELATIVE relocations.
for (; relative < r; ++relative)
  DO_ELF_MACHINE_REL_RELATIVE (map, l_addr, relative);
// Handle other relocations that are not IRELATIVE.
for (; r < end; ++r) {
  if (r is R_*_IRELATIVE) {
    if (r2 == NULL) r2 = r;
    end2 = r;
    continue;
  }
  elf_machine_rel (... r);
}
// Then handle IRELATIVE relocations.
if (r2 != NULL)
  for (; r2 <= end2; ++r2)
    if (ELFW(R_TYPE) (r2->r_info) == ELF_MACHINE_IRELATIVE)
     elf_machine_rel (... r2);
```

A `R_*_IRELATIVE` relocation or a symbolic relocation (e.g. `R_X86_64_64`) referencing an ifunc symbol requires rtld to call an ifunc resolver.
The ifunc resolver may access variables or functions that require relocations (usually `R_*_JUMP_SLOT`, `R_*_GLOB_DAT`, or `R_*_RELATIVE`).
`R_*_GLOB_DAT` and `R_*_RELATIVE` relocations are resolved before `R_*_IRELATIVE`, so reading a variable or taking an address of a function is fine.

However, when lazy binding is enabled (neither `ld -z now` or `LD_BIND_NOW=1`; `do_lazy == 1`), `R_*_IRELATIVE` relocations in `.rela.dyn` are resolved before `R_*_JUMP_SLOT` in `.rela.plt`.
Therefore, calling a preemptible function in an ifunc resolver will crash due to accessing an unresolved GOTPLT entry.
This may work with GNU ld's x86-64 port, which places certain `R_X86_64_IRELATIVE` relocations in `.rela.plt`. The exact rules are complex and I am not interested in figuring it out.

---

In glibc, there used to be a problem where ifunc resolvers ran before `GL(dl_hwcap)` and `GL(dl_hwcap2)` were set up for static pie <https://sourceware.org/bugzilla/show_bug.cgi?id=27072>.

---

Let's see a case where an ifunc resolver has PLT calls.

```sh
cat > a.c <<eof
  #include <stdio.h>

  int a_impl() { return 42; }
  void *a_resolver() {
    puts("a_resolver");
    return (void *)a_impl;
  }
  int a() __attribute__((ifunc("a_resolver")));

  // .rela.dyn.rel => R_X86_64_64 referencing STT_GNU_IFUNC in .rela.dyn
  int (*fptr_a)() = a;

  int main() { printf("%d\n", a()); }
eof

cc -fpie -c a.c
cc -fuse-ld=bfd -pie a.o -o a
```

GNU ld produces an `R_X86_64_IRELATIVE` in `.rela.dyn`.
In lazy PLT mode, glibc ld.so will call the ifunc resolver before the `R_X86_64_JUMP_SLOT` for `puts` is set up, and segfault.

---

FreeBSD rtld uses multiple phases:

* Resolve non-`COPY` non-`IRELATIVE` non-`STT_GNU_IFUNC` relocations in all objects. Record what ifunc relocations categories have appeared.
* Resolve `COPY` relocations
* Initialize states needed by ifunc resolvers
* Prepare a list used to call init functions (if A depends on B, B is ordered before A)
* Resolve relocations in the init order
  + Resolve `IRELATIVE` relocations
  + Resolve other `.rela.dyn` relocations referencing `STT_GNU_IFUNC` (mostly absolute and `GLOB_DAT` relocations)
  + If neither `LD_BIND_NOW` nor `DF_1_NOW`, resolve `JUMP_SLOT` relocations referencing `STT_GNU_IFUNC`

In the lazy binding mode, when a `JUMP_SLOT` relocation is called, the PLT
trampoline calls the ifunc resolver.

The approach turns out to be very robust. Many segfault examples on glibc work on FreeBSD.

```sh
cat > ./a.c <<eof
#include <stdio.h>

int a_impl() { return 42; }
void *a_resolver() {
  puts("a_resolver");
  return (void *)a_impl;
}
int a() __attribute__((ifunc("a_resolver")));

int (*fptr_a)() = a;
int b(); extern int (*fptr_b)();
int c(); extern int (*fptr_c)();

int main() {
  printf("%d\n", a());
  b();
  printf("b: %p %p\n", fptr_b, b);
  printf("c: %p %p\n", fptr_c, c);
}
eof
cat > ./b.c <<eof
#include <stdio.h>

void c();
int b_impl() { return 42; }
void *b_resolver() {
  puts("b_resolver");
  c();
  return (void *)b_impl;
}
int b() __attribute__((ifunc("b_resolver")));
int (*fptr_b)() = b;
eof
cat > ./c.c <<eof
#include <stdio.h>

int d();
int c_impl() { return 42; }
void *c_resolver() {
  puts("c_resolver");
  d();
  return (void *)c_impl;
}
int c() __attribute__((ifunc("c_resolver")));
int (*fptr_c)() = c;
eof
cat > ./d.c <<eof
#include <stdio.h>

int d_impl() { return 42; }
void *d_resolver() {
  puts("d_resolver");
  return (void *)d_impl;
}
int d() __attribute__((ifunc("d_resolver")));
int (*fptr_d)() = d;
eof
cat > ./Makefile <<'eof'
a: a.c b.so c.so d.so
	${CC} -g -fpie a.c -Wl,--no-as-needed ./b.so ./c.so ./d.so -pie -ldl -o $@

.SUFFIXES: .so
.c.so:
	${CC} -g -fpic $< -shared -o $@
eof
```

Note that in `b.so`, the ifunc resolver has a PLT call to `c.so`.
In `c.so`, the ifunc resolver has a PLT call to `d.so`.
The ifunc resolvers thus have dependencies.
In real world applications, `d` called by other ifunc resolvers may be some performance critical functions like `memset`.

```text
% ./a
# resolve d.so in init order
d_resolver  # R_X86_64_64

# resolve c.so in init order
c_resolver  # R_X86_64_64
d_resolver  #   triggered lazy JUMP_SLOT

# resolve b.so in init order
b_resolver  # R_X86_64_64
c_resolver  #   triggered lazy JUMP_SLOT

# resolve a in init order
a_resolver  # R_X86_64_IRELATIVE
b_resolver  # R_X86_64_GLOB_DAT
c_resolver  # R_X86_64_GLOB_DAT

42

b_resolver  # lazy JUMP_SLOT

b: 0x80106f670 0x80106f670
c: 0x801073670 0x801073670

% LD_BIND_NOW=1 ./a
# resolve d.so in init order
d_resolver

# resolve c.so in init order
d_resolver  # eager JUMP_SLOT
c_resolver  # R_X86_64_64

# resolve b.so in init order
c_resolver  # eager JUMP_SLOT
b_resolver  # R_X86_64_64

# resolve a in init order
a_resolver  # R_X86_64_IRELATIVE
b_resolver  # eager JUMP_SLOT
b_resolver  # R_X86_64_GLOB_DAT
c_resolver  # R_X86_64_GLOB_DAT

42
b: 0x80106f670 0x80106f670
c: 0x801073670 0x801073670
```

When `a` is built with `-no-pie -fno-pic`, copy relocataions and canonical PLT entries are used.
`R_X86_64_64` relocations in `b.so` and `c.so` are bound to canonical PLT entries, so there are fewer resolver calls.
```text
d_resolver
a_resolver
42
b_resolver
c_resolver
d_resolver
c: 0x201ca0 0x201ca0
b: 0x201c90 0x201c90
```

We can let `b.so` depend on `c.so`, or let `c.so` depend on `b.so`, or swap the link order of `./b.so` and `./c.so`. Still works.

On glibc, <https://sourceware.org/git/?p=glibc.git;a=shortlog;h=refs/heads/fw/bug21041> can improve robustness of ifunc resolvers.

#### ifunc callee in an ifunc resolver

For two non-preemptible ifuncs A and B, B can call A as long as A's IRELATIVE relocation appears before B's.

Consider code where a non-local ifunc appears after its callee (`memset`).
The assembler often ensures that `ifunc` comes after its callee in the symbol table.
The symbol table essentially establishes a topolical order.
If the linker respects this symbol table order and generates IRELATIVE relocations in the same order, ifunc resolvers with dependencies will function correctly.

```c
static int impl(void) { return 42; }
static void *resolver(void) { memset(...); return impl; }
// Non-local `ifunc` occurs after its callee `memset`.
void *ifunc(void) __attribute__((ifunc("resolver")));
```

```
// We are good if dependended resolvers appear first
IRELATIVE for memset
IRELATIVE for resolver
```

The ifunc `__gettimeofday` calls `_dl_vdso_vsym`, which invokes the ifunc `memset`.
Before <https://sourceware.org/git/?p=glibc.git;a=commit;h=4a39c34c4f85de57fb4e648cfa1e774437d69680>, `__gettimeofday` used to require an IRELATIVE relocation, leading to IRELATIVE dependencies.

#### ifunc resolver and `DT_INIT_ARRAY`

For the main executable and initially loaded shared objects, `DT_INIT_ARRAY` functions are called after all relocation resolving is done.
Therefore, when an ifunc resolver reads a global variable like `int var = initialize();`, it will read the initial value, possibly zero.
Instead, the ifunc resolver should call an initializer directly.

[Function multi-versioning](/blog/2023-02-05-function-multi-versioning) describes such an interaction example.

#### Guideline

Szabolcs Nagy has a nice list:

* may be called multiple times (for every dynamic reloc referencing it in the same or different module).
* must return the same result during the lifetime of a process and thus its argument must not change either between calls.
* must not access global data or call functions in other modules including the libc, so relevant parameters must be passed in the arguments or available via id registers (though the latter is trap-and-emulate so has linux entry overhead).
* may access local data and local functions but only if this does not depend on dynamic relocations (which may be processed later than the ifunc reloc).
* must not access tls (this is arch specific constraint on targets where it runs before tls setup with static linking).
* must be async-signal- and thread-safe (in case ld.so supports lazy binding).
* must not fail (may run late when crash/abort is unacceptable).

## Apple systems

Since Clang 18, `__attribute__((ifunc("resolver")))` is also [implemented for Apple systems](https://github.com/llvm/llvm-project/pull/73687).
The implementation uses a similar mechanism as ELF's JUMP_SLOT lazy binding. Let's break down how this works:

```asm
        .section        __DATA,__data
        .p2align        3, 0x0
_ifunc.lazy_pointer:
        .quad   _ifunc.stub_helper

        .section        __TEXT,__text,regular,pure_instructions
        .globl  _ifunc
        .p2align        2
_ifunc:
        adrp    x16, _ifunc.lazy_pointer@GOTPAGE
        ldr     x16, [x16, _ifunc.lazy_pointer@GOTPAGEOFF]
        ldr     x16, [x16]
        br      x16
        .p2align        2
_ifunc.stub_helper:
        stp     x29, x30, [sp, #-16]!
        mov     x29, sp
        stp     x1, x0, [sp, #-16]!
        stp     x3, x2, [sp, #-16]!
        stp     x5, x4, [sp, #-16]!
        stp     x7, x6, [sp, #-16]!
        stp     d1, d0, [sp, #-16]!
        stp     d3, d2, [sp, #-16]!
        stp     d5, d4, [sp, #-16]!
        stp     d7, d6, [sp, #-16]!
        bl      _resolver
        adrp    x16, _ifunc.lazy_pointer@GOTPAGE
        ldr     x16, [x16, _ifunc.lazy_pointer@GOTPAGEOFF]
        str     x0, [x16]
        add     x16, x0, #0
        ldp     d7, d6, [sp], #16
        ldp     d5, d4, [sp], #16
        ldp     d3, d2, [sp], #16
        ldp     d1, d0, [sp], #16
        ldp     x7, x6, [sp], #16
        ldp     x5, x4, [sp], #16
        ldp     x3, x2, [sp], #16
        ldp     x1, x0, [sp], #16
        ldp     x29, x30, [sp], #16
        br      x16
```

* Lazy pointer: In a data section, a pointer is designated for the function marked with the `ifunc` attribute. This pointer starts by pointing to a "stub_helper" function.
* First call: When the ifunc function is first used, control jumps to the stub_helper, which calls the resolver.
* Resolver: The resolver figures out the best function implementation.
* Lazy pointer update: The stub helper then rewrites the lazy pointer to directly target the chosen implementation. This is similar to glibc's `elf_machine_fixup_plt`.
* Subsequent calls: Subsequent call to the ifunc function bypasses stub_helper/resolver and jumps straight to the selected implementation.

The lazy pointer is like a [manually-crafted ELF `.got.plt` entry](/blog/2021-08-29-all-about-global-offset-table#ppc64-elfv2-table-of-contents-toc).
Its update is not atomic, just like glibc's `elf_machine_fixup_plt` does not use atomic instructions.

## Alternative

An ifunc can be emulated by a global variable of a function pointer type, with no difference at a call site (calling a function pointer can use the same syntax as a function call).
For example, the following ifunc use

```cpp
void impl();
extern "C" void (*resolver())() { return impl; }
void ifunc() __attribute__((ifunc("resolver")));

void use() {
  ifunc();
  ifunc();
}
```

can be replaced with:
```cpp
void impl();
void (*resolver())() { return impl; }

extern void (*const ifunc)() = resolver();

void use() {
  ifunc();
  ifunc();
  // ifunc = nullptr; // do not compile
}
```

If you don't require `ifunc` to be const, its initialization can be postponed, and optionally with some arguments.

You can add `__attribute__((weak))` to `resolver` to prevent the compiler from optimizing out the `resolver` call or redirecting `ifunc()` to `impl()` directly.
Let's compare the difference for x86-64.

The ifunc scheme calls `jmp _Z5ifuncv`. `_Z5ifuncv` is guaranteed to have a PLT entry (whether it is defined in the same linker unit or not) which loads an entry from the associated GOT entry.

The global function pointer scheme emits code like `callq *ifunc(%rip)`, which is similar to `-fno-plt` code generation with a normal function.
See [All about Procedure Linkage Table#-fno-plt](/blog/2021-09-19-all-about-procedure-linkage-table#fno-plt) for detail.

Using a global function pointer offers more flexibility:

* It can be initialized at any appropriate time. The relocation resolving time ifunc is subject to static initialization order fiasco.
* The resolver can have custom arguments, not the fixed ones provided by the dynamic loader.
