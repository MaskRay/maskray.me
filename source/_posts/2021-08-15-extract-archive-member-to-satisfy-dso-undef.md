---
layout: post
title: Extract an archive member to satisfy a DSO undef
author: MaskRay
tags: [binutils,linker,llvm]
---

In ELF linkers, undefined symbols from DSO participate in symbol resolution, just like undef from .o.
The differences with undef from .o are:

* undef purely from DSO do not need .symtab entries
* --no-allow-shlib-undefined can error for such undef (-rpath-link ("load dependent libraries") behavior can suppress some errors.

ELF linkers extract an archive member to satisfy an undefined symbol from a shared object.
Here is an example:

<!-- more -->

```sh
echo '.globl _start; _start: call bb' > a.s
echo '.globl bb; bb: call cc' > b.s
echo '.globl cc; cc: nop' > c.s
cc -c a.s b.s c.s
rm -f c.a && ar rc c.a c.o
ld -shared -z defs c.o -o c.so
ld -shared -z defs b.o ./c.so -o b.so

ld -rpath=. a.o b.so c.a
```

ld extracts `c.a(c.o)` and exports `cc` into the dynamic symbol table.
ld.bfd, gold, and ld.lld have the same behavior.

This archive extraction behavior intrigued me because link.exe (PE/COFF) and ld64 (Mach-O) don't inspect undefined symbols in DLL/dylib for symbol resolution.

```sh
# On macOS
echo '.globl _main; _main: call bb' > a.s
echo '.globl bb; bb: call cc' > b.s
echo '.globl cc; cc: nop' > c.s
clang --target=x86_64-apple-darwin -c a.s b.s c.s
rm -f c.a && ar rc c.a c.o

clang -dynamiclib c.o -o c.dylib
clang -dynamiclib b.o c.dylib -o b.dylib
clang a.o b.dylib c.a
```

`a.out` doesn't define or reference `cc`.

## Thoughts

On ELF, an undefined symbol from a shared object can interact with a .o definition.
The .o definition needs to be exported to `.dynsym` in case the shared object relies on symbols in the executable.

The rationale is that a shared object may have incomplete `DT_NEEDED` entries.
An undefined symbol may need to be resolved to the executable or another shared object.
we can imagine that not extracting archive members to satisfy an undefined symbol from a shared object can cause ld.so "unresolved symbol" errors.
(`--no-allow-shlib-undefined` may catch such fragile links.)

However, if every shared object is linked with `-z defs` (aka `--no-undefined`), we can make sure a shared object won't have unresolved symbols (provided that the runtime shared objects have the same symbol set, which is a reasonable request.)
Not extracting archive members will be very robust.

Due to circular dependency or other reasons, it can be difficult to make every shared object `-z defs` happy.
Unfortunately `--no-allow-shlib-undefined` doesn't provide a way to allow some undefined symbols.

The above reasoning might make not extracting an archive member plausible.
However, a more important factor is at play here: indirect dependency should not affect symbol resolution.
A shared object built with `-z defs` is not a good justification because it involves indirect dependency.
The linker can only reasonably check direct dependencies for symbol resolution.

### One definition rule violation

One definition rule is C++'s, but we can reuse the concept: multiple definitions have more or less unreliability.

In the `ld -rpath=. a.o b.so c.a` ELF example, if we add the `DF_SYMBOLIC` flag to `c.so`
`a.out` will have definitions trying to interpose `c.so` definitions in vain.
Linkers don't seem to have any ability to detect such fragile links.

### Did "dynamic linking should be similar to static linking" affect the design?

The ELF pioneers had "dynamic linking should be similar to static linking" in mind when designing dynamic linking.
I think it means two things: (a) no source-level annotation, (b) emulating archive member extraction.

For `ld a.o -lb c.a`, `-lb` may refer to either `libb.a` or `libb.so`.
One might argue that the member in `c.a` should be extracted to satisfy `b`, in case `b` refers to `libb.a`.

However, an archive and a shared object have a major difference: a shared object tracks dependencies while an archive does not.
If we let `b.a` track dependencies, the executable linker command line may probably be `ld a.o -lb -lb0 c.a` where `libb0.a` is another archive satisfying undefined symbols in `libb.a`.
In this case, not extracting `c.a` members is still fine.

---

Related discussions:

* <https://bugs.llvm.org/show_bug.cgi?id=43554>
* <https://reviews.llvm.org/D108006> (propose `--no-search-static-libs-for-shlib-undefined` to ld.lld)
* <https://groups.google.com/g/generic-abi/c/NyH6f470Cuc>
