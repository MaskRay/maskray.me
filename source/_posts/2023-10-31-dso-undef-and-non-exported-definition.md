layout: post
title: DSO undef and non-exported definition
author: MaskRay
tags: [binutils,linker]
---

## DSO undef and non-exported def

If a DSO has an undefined `STB_GLOBAL` symbol that is defined in a relocatable object file but not exported, should the `--no-allow-shlib-undefined` feature report an error?
You may want to check out [Dependency related linker options](/blog/2021-06-13-dependency-related-linker-options) for a discussion of this option and the [symbol exporting rule](/blog/2020-11-15-explain-gnu-linker-options#export-dynamic).

<!-- more -->

For quite some time, the `--no-allow-shlib-undefined` feature has been implemented in lld/ELF as follows:

```cpp
for (SharedFile *file : ctx.sharedFiles) {
  bool allNeededIsKnown =
      llvm::all_of(file->dtNeeded, [&](StringRef needed) {
        return symtab.soNames.count(CachedHashStringRef(needed));
      });
  if (!allNeededIsKnown)
    continue;
  for (Symbol *sym : file->requiredSymbols)
    if (sym->isUndefined() && !sym->isWeak())
      diagnose("undefined reference due to --no-allow-shlib-undefined: " +
               toString(*sym) + "\n>>> referenced by " + toString(file));
}
```

Recently I noticed that GNU ld implemented a related error in [April 2003](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=1b1fe8feb35ab988cdaf8481fc52ebdbf2db3d93) ([discussion](https://sourceware.org/pipermail/binutils/2003-April/026568.html)).

```sh
echo '.globl _start; _start: call shared' > main.s && clang -c main.s
echo '.globl shared; shared: call foo' > a.s && clang -shared -fpic a.s -o a.so
echo '.globl foo; foo:' > def.s && clang -c def.s && clang -shared def.o -o def.so
echo '.globl foo; .hidden foo; foo:' > def-hidden.s && clang -c def-hidden.s
```
```
% ld.bfd main.o a.so def.o
% ld.bfd main.o a.so def-hidden.o
ld.bfd: a.out: hidden symbol `foo' in def-hidden.o is referenced by DSO
ld.bfd: final link failed: bad value
```

A non-local default or protected visibility symbol can satisfy a DSO reference. The linker will export the symbol to the dynamic symbol table.
Therefore, `ld.bfd main.o a.so def.o` succeeds as intended.

We encounter an error for `ld.bfd main.o a.so def-hidden.o` because a symbol with hidden visibility cannot be exported, and it's unable to satisfy the reference in `a.so` at run-time.

Here is another interesting case: we use a version script to change the binding of a defined symbol to `STB_LOCAL`, causing it to be unable to satisfy the reference in `a.so` at run-time.
GNU ld also reports an error in this case.
```
% ld.bfd --version-script=local.ver main.o a.so def.o
ld.bfd: a.out: local symbol `foo' in def.o is referenced by DSO
ld.bfd: final link failed: bad value
```

My recent commit <https://github.com/llvm/llvm-project/commit/1981b1b6b92f7579a30c9ed32dbdf3bc749c1b40> strengthened LLD's `--no-allow-shlib-undefined` to detect cases in which the non-exported definitions are garbage-collected.
I have landed <https://github.com/llvm/llvm-project/pull/70769> to cover non-garbage-collected cases for LLD 18.

## DSO undef, non-exported def, and DSO def

A variation of the scenario mentioned above occurs when a DSO definition is also present.
Even if the executable does not export `foo`, another DSO (`def.so`) may provide it.
GNU ld's check allows for this case.

```sh
ld.bfd main.o a.so def-hidden.o def.so  # succeeded
ld.lld main.o a.so def-hidden.o def.so  # failed after commit 1981b1b6b92f7579a30c9ed32dbdf3bc749c1b40
```

It turns out that <https://github.com/llvm/llvm-project/commit/1981b1b6b92f7579a30c9ed32dbdf3bc749c1b40> unexpectedly strengthened `--no-allow-shlib-undefined` to also catch this ODR violation.
More precisely, when all three conditions are met, the new `--no-allow-shlib-undefined` code reports an error.

* There is a DSO undef that can be satisfied by a definition from another DSO (referred to as `SharedSymbol` in lld/ELF).
* The `SharedSymbol` is overridden by a non-exported (usually of hidden visibility) definition in a relocatable object file (`Defined`).
* The section containing the `Defined` is garbage-collected (it is not part of `.dynsym` and is not marked as live).

An exported symbol is a GC root, making its section live.
A non-exported symbol, however, can be discarded when its section is discarded.

So, is this error legitimate?
At run-time, the undefined symbol `foo` in `a.so` will be bound to `def.so`, even if the executable does not export `foo`, so we are fine.
This suggests that the `--no-allow-shlib-undefined` code probably should not report an error.

However, both `def-hidden.o` and `def.so` define `foo`, and we know the definitions are different and less likely benign.
At the very least, they are not exactly the same due to different visibilities or one being localized by a version script.

A real-world report boils down to
```
% ld.lld @response.txt -y _Znam
...
libfdio.so: reference to _Znam
libclang_rt.asan.so: shared definition of _Znam
libc++.a(stdlib_new_delete.cpp.obj): definition of _Znam
ld.lld: error: undefined reference due to --no-allow-shlib-undefined: _Znam
>>> referenced by libfdio.so
```

How does `libfdio.so` obtain a reference to `_Znam`? Well, `libfdio.so` is linked against both `libclang_rt.asan.so` and `libc++.a`.
Due to symbol processing rules, the definition from `libclang_rt.asan.so` takes precedence. (See [Symbol processing#Shared object overriding archive](/blog/2021-06-20-symbol-processing).)

An appropriate solution is to replace `libc++a` with an AddressSanitizer-instrumented version that does not define `_Znam`.

I have also encountered issues stemming from the combination of multiple definitions from `libgcc.a` (with hidden visibility) and `libclang_rt.builtins.a` (with default visibility), relying on archive member extraction rules.
```
% ld.lld @response.txt -y __divti3
...
a.so: reference to __divti3
libgcc.a(_divdi3.o): definition of __divti3
libc++.so: shared definition of __divti3
# A lazy symbol in libclang_rt.builtins.a is not reported by -y
ld.lld: error: undefined reference due to --no-allow-shlib-undefined: __divti3
>>> referenced by a.so
```

`a.so` is linked against `libc++.so` and `libclang_rt.builtins.a` and obtains a reference to `__divti3` due to `libc++.so`.
For the executable link, the undesired situation arises as the definition in `libgcc.a` takes precedence. What we actually want is for `libgcc.a` to provide the missing components from `libclang_rt.builtins.a`.

Some users compile relocatable object files with `-fvisibility=hidden` to disallow dynamic linking.
However, when their system includes specific shared objects, it increases the risk of conflicting multiple definition symbols.

While this additional check introduced in <https://github.com/llvm/llvm-project/commit/1981b1b6b92f7579a30c9ed32dbdf3bc749c1b40> may not perfectly fit into `--no-allow-shlib-undefined`, I believe it has value.
As a result, I have proposed [`--[no-]allow-non-exported-symbols-shared-with-dso`](https://github.com/llvm/llvm-project/pull/70163).
However, I am also on the fence that we introduce a new option, as it may not get used.

Technically, the check can be extended to default visibility to catch all link-time symbol interposition.
However, I suspect that there are a lot of benign violations and in the absence of an ignore list mechanism, this extension will not be useful.
