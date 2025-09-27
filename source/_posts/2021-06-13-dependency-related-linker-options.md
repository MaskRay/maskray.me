layout: post
title: Dependency related linker options
author: MaskRay
tags: [gcc,llvm,linker]
---

This article describes the dependency related linker options `-z defs`, `--no-allow-shlib-undefined`, and `--warn-backrefs`.
Deploying them in a build system can improve build health.

## `-z defs`

<!-- more -->

`-z defs` (alias `--no-undefined`) tells the linker to report an error for an unresolved undefined non-weak symbol from a relocatable object file.
`-z nodefs` is the opposite option that suppresses such errrors.
Executables links (`-no-pie` and `-pie`) default to `-z defs`, while shared object links (`-shared`) default to `-z undefs`.

"Unresolved" means that (a) no other relocatable object file linked into the component provides a definition, and (b) no shared object linked into the component provides a definition.

When an undefined symbol in X is provided by a link-time shared object Y, we can say that X depends on Y.
The linker will record this fact by adding a `DT_NEEDED` dynamic tag.
If Y has a `DT_SONAME` tag, the `DT_NEEDED` value is the SONAME; otherwise, the `DT_NEEDED` value is the path of Y (either absolute or relative).

When an undefined symbol in X is not provided by any link-time shared objects, the situation is usually referred to as underlinking.

I can think of several reasons why `-shared` links have the loose default `-z undefs`:

(a) ELF supports interposition.
For example, if a shared object has an undefined symbol, the symbol may be provided by an arbitrary shared object or by the executable at runtime.
By not requiring the dependencies to be fully specified, the runtime can have flexibility.

(b) There may be mutual references between two shared objects, X and Y.
Breaking the tie with a regular approach would be difficult if full dependencies needed to be specified.
A variant is that the executable has X as its dependency while X also references symbols from the executable.

For (a), such unbounded flexibility does not quite fit into a build system.
With a modular design, the libraries should have well-defined roles and dependencies.
We do not substitute an arbitrary shared object for a link-time shared object.
When we need such flexibility, we can define an interface (e.g., C++ virtual functions) and have several shared objects implement the interface.

Having fully specified dependencies makes the shipped shared object X convenient to use.
It is likely that an executable link does not use X's dependency Y.
It would feel awkward if the executable link needs to additionally link against Y when linking against X.

If you have read my article about ELF interposition, you should know that having the dependency information makes direct bindings possible, which can improve symbol lookup time for the dynamic loader.
No ELF system other than Solaris has implemented direct bindings, though.

(b) indicates a bad layering of libraries.
X and Y are no longer isolated components.
A change in X may affect Y and vice versa. The unit testing for X needs to involve Y.
With archives, you will need `--start-group X.a Y.a --end-group` with GNU ld and gold.
Actually merging X and Y is often a better strategy.

If we don't merge X and Y, <http://blog.darlinghq.org/2018/07/mach-o-linking-and-loading-tricks.html> mentions that the Mach-O approach for such circular dependencies is usually to link the libraries twice.

```sh
ld -o libfoo.dylib foo.o -flat_namespace -undefined suppress
ld -o libbar.dylib bar.o -flat_namespace -undefined suppress
ld -o libfoo.dylib foo.o libbar.dylib
ld -o libbar.dylib bar.o libfoo.dylib
```

The ELF counterpart is:
```sh
ld -shared foo.o -o foo.so
ld -shared bar.o -o bar.so
ld -shared -z defs foo.o bar.so -o foo.so
ld -shared -z defs bar.o foo.so -o bar.so
```

A build system may support archives as well as shared objects.
An archive is a collection of regular object files, with special archive member selection semantics in the linker.
(We will discuss the archive member selection later.)
If A.a needs definitions from B.a and you do not supply B.a when linking a dependent executable (`ld ... A.a` instead of `ld ... A.a b.a`), you know that you may be welcomed by `undefined reference to` (GNU ld) or `undefined symbol: ` (ld.lld).
In practice a build system needs to track dependencies for archives.

Mach-O linkers' default `-undefined error` is similar to ELF `-z defs`.
Mach-O linkers' `-U` allows an individual symbol to be undefined. ELF linkers don't have a counterpart.

## `-z now`

`-z now` tells the linker to set the `DF_1_NOW` dynamic tag.
ld.so will resolve `R_*_JUMP_SLOT` relocations eagerly and report an error if there is an unresolved symbol.

`-z now` can be seen as a relaxed `-z defs`. The full depencency requirement is moved from link-time to runtime.

## `--no-allow-shlib-undefined`

`--no-allow-shlib-undefined` tells the linker to report an error for an unresolved undefined symbol from a shared object.
Executable links (`-no-pie` and `-pie`) default to `-z defs` while shared object links (`-shared`) default to `-z undefs`.

E.g. for `ld -shared a.o b.so -o a.so`, if `b.so` has an undefined symbol not defined by `a.o` or `b.so`'s dependencies, the linker will report an error.

gold and ld.lld do not recursively load `DT_NEEDED` tags. Instead, they report an error only when `b.so`'s `DT_NEEDED` list is all seen as input shared objects.
Say, `b.so` depends on `c.so` and `d.so`. `ld.lld -shared a.o b.so c.so -o a.so` will not error for an unresolved undefined symbol in `b.so` because `b.so`'s dependency `d.so` is not on the linker command line.
`ld.lld -shared a.so b.so c.so d.so -o a.so` may error.

If a build system is `-z defs` clean, it will also be `--no-allow-shlib-undefined` clean.
If a build system cannot use `-z defs`, `--no-allow-shlib-undefined` can catch some propagated problems.

```cpp
// a.cc => a.o => a.so (not linked with -z defs)
void f(); // f is undefined
void g() { f(); }

// b.cc => b.o
void g();
int main() { g(); }
```

`ld b.o a.so` will report an error for the undefined symbol `f` in `a.so`.

In glibc, there will be a runtime error `symbol lookup error: ... undefined symbol: ` when glibc tries to resolve `f`.

The symbol resolution happens either eagerly (`ld -z now` or `LD_BIND_NOW=1`) or lazily (when the PLT entry is executed for the first time).
Therefore, with lazy PLT binding, if we arrange for `g` to not be called, the `symbol lookup error: ... undefined symbol: ` error will go away.

If we use Bazel layering check features as an analogy, `-z defs` (link what you use) is like `layering_check` (include what you use) while `--no-allow-shlib-undefined` is a bit like `hdrs_check`.
See [Layering check with Clang](/blog/2022-09-25-layering-check-with-clang).

### Whether `--no-allow-shlib-undefined` checks shared objects recursively

GNU ld checks shared objects recursively while gold and ld.lld don't.
There are several reasons that ld.lld doesn't.

First, complexity.
In the GNU ld manpage, `-rpath-link=dir` lists 12 rules, which are just overwhelming.
Several points don't play well with cross linking.
Eliminating difference between native link and a cross link is an important design choice of ld.lld.
In addition, I know `--no-copy-dt-needed-entries` can affect the behavior subtly, which isn't even documented.

Second, performance.
Recursive loading needs to parse additional shared objects which may slow down the link a bit.

Third, practicality.
`--allow-shlib-undefined` is an unfortunate default for `-shared`. Changing it may be disruptive today.
Mach-O and PE/COFF have many problems but this may be a place where they got right.
I have several paragraphs describing it in https://maskray.me/blog/2021-06-13-dependency-related-linker-options
Recursive loading of --allow-shlib-undefined gives me a feeling of working around the wrong default.

### What if the symbol is defined in a relocatable object file but is not exported?

This is a quite complex topic. Please refer to [DSO undef and non-exported definition](/blog/2023-10-31-dso-undef-and-non-exported-definition).

---

Gentoo Linux has a quality assurance project to enable `-Wl,--no-allow-shlib-undefined`: <https://wiki.gentoo.org/wiki/Project:Quality_Assurance/-Wl,-z,defs_and_-Wl,--no-allow-shlib-undefined>.

## Archive processing

This concept is required to understand `--warn-backrefs`.
Please check out [Symbol processing#Archive processing](/blog/2021-06-20-symbol-processing#archive-processing).

## `--warn-backrefs`

VMS (now OpenVMS), Mach-O ld64, Windows link.exe, and ld.lld use a different design.

For `ld.lld ... definition.a reference.o`, the archive index of `definition.a` lists the defined symbols.
When processing `definition.a`, ld.lld uses "lazy symbols" to represent the lazy definitions.
Each lazy symbol has an associated archive(member) name.
When processing `reference.o` (or a lazy object file which is now extracted), an undefined symbol can cause the lazy symbol to be fetched, i.e. a previous `definition.a` member will be extracted.

This archive processing strategy is nice because in the absence of duplicate definitions, `ld.lld ... definition.a reference.o` and `ld.lld ... reference.o definition.a` cause no ordering difference.

Moreover, `--start-group` is a no-op in ld.lld. The traditional approach may iterate over the archive members more than once, and `--start-group` can exacerbate the problem.
The ld.lld approach turns out to be easier to implement and can improve archive processing performance.

Let's analyze a case involving 4 libraries: `a`, `b`, `c`, and `d`.
The dependency edges are: `a->b, a->c, b->d, c->d`. There is an unspecified edge: `b->c`.
When the build system orders the libraries in a dependency order, there are two possibilities.

* `ld ... a.a c.a b.a d.a` may lead to an undefined symbol error. Some members of `c.a` may be dropped if they do not resolve a previously undefined symbol. When `b.a` members are extracted, their references cannot be satisfied by the dropped `c.a` members.
* `ld ... a.a b.a c.a d.a` is fine, because after `b.a` is processed, we should have seen all the symbol requirements on `c.a`'s members. `a, b, c, d` is a topological order of the full dependency list.

The layering check is loose because it only checks one particular topological order.
Nevertheless, a build system using this option can catch many missing dependency edges.

Working at the binary level, the option can catch some problems not detected by modules based layering check `clang -fmodule-name=X -fmodules-strict-decluse` (`error: module X does not depend on a module exporting 'string.h'`),
e.g. using a declaration without including a header.

For ld.lld, `--warn-backrefs` was added to check archive processing compatibility problems with GNU ld.
In ld.lld 11.0.0, I significantly improved `--warn-backrefs` to make the compatibility checking reliable.

Because the archive name is remembered, the diagnostic (e.g. `warning: backward reference detected: foo in a1.o refers to a2.o`) is better than GNU ld's (no filename).

We can use `--warn-backrefs-exclude=a2.o` to suppress the warning.

### Linking sandwich problem

Let's discuss `ld def1.a ref.o def2.a` when both `def1.a` and `def2.a` define a symbol which is referenced by `ref.o`.
I call this a linking sandwich.

The traditional approach succeeds: the linker simply ignores `def1.a` and then extracts a member of `def2.a`.
ld.lld extracts a member of `def1.a` when the undefined symbol in `ref.o` is scanned.
The `def2.a` member may or may not be extracted.

If extracted, the `def1.a` member and `def2.a` member have a duplicate symbol. If both definitions are `STB_GLOBAL`, ld.lld will report a duplicate symbol error.

If not extracted, then GNU ld and ld.lld extract different archive members.
The most common incarnation is that `def1.a` and `def2.a` have the same path, e.g. `liblldCommon.a liblldCOFF.a liblldCommon.a`.
This case is benign. I submitted <https://reviews.llvm.org/D77522> to suppress the `--warn-backrefs` diagnostic.
If `def1.a` and `def2.a` have different paths, this is like an one-definition-rule violation.
I think if we ever report a warning, we may need a new option.

---

The following example demonstrates another scenario where GNU ld and LLD exhibit different behavior.
The linker first extracts `b.a(b0.o)` to resolve the undefined `foo`.

- GNU ld proceeds to extract `b.a(b1.o)` to resolve the `dup` symbol
- LLD instead extracts `a.a(a.o)`. Notably, the `--warn-backrefs` flag fails to detect this behavioral difference as a potential issue.

```
# RUN: rm -rf %t && split-file %s %t && cd %t
# RUN: as main.s -o main.o
# RUN: as a.s -o a.o
# RUN: as b0.s -o b0.o
# RUN: as b1.s -o b1.o
# RUN: ar rc a.a a.o
# RUN: ar rc b.a b0.o b1.o

# RUN: ld.bfd main.o a.a b.a
# RUN: ld.lld --fatal-warnings --warn-backrefs main.o a.a b.a

#--- main.s
.globl _start
_start:
  call foo

#--- a.s
.globl dup; dup: hlt

#--- b0.s
.globl foo; foo: call dup
#--- b1.s
.globl dup; dup: ret
```
