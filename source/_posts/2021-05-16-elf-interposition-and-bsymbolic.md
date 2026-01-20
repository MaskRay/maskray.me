---
layout: post
title: ELF interposition and -Bsymbolic
author: MaskRay
tags: [gcc,llvm,linker]
---

This article describes ELF interposition, the linker option `-Bsymbolic`, and its friends. In the end, it will discuss an ambitious plan which I dubbed "the Last Alliance of ELF and Men".

Motivated by [a great post by Daniel Colascione](https://www.facebook.com/dan.colascione/posts/10107358290728348) ("Python is 1.3x faster when compiled in a way that re-examines shitty technical decisions from the 1990s.") and a recent [rant from Linus Torvalds on shared objects' performance issues](https://lore.kernel.org/lkml/CAHk-=whs8QZf3YnifdLv57+FhBi5_WeNTG1B-suOES=RcUSmQg@mail.gmail.com/),
I have summarized the current unfortunate ELF state and filed some GCC/binutils feature requests.
I believe the performance of our shared object oriented world will be no slower than one with mostly statically linked executables.

(I wrote [_-fno-semantic-interposition_](/blog/2021-05-09-fno-semantic-interposition) first but then realized reorganization would improve readability, so moved some parts and added some stuff to this new article.)

<!-- more -->

The ELF pioneers had "dynamic linking should be similar to static linking" in mind when designing dynamic linking.
I think it means two things: (a) no source-level annotation, (b) emulating archive member extraction.
Let's discuss both concepts in detail.

## No source-level annotation

Say, we have two default visibility functions f and g. g calls f. g is compiled and linked into a shared object.
There are 3 cases for f.

First case: f is defined in the same translation unit of g.
I will discuss this in depth in my next article [_-fno-semantic-interposition_](/blog/2021-05-09-fno-semantic-interposition).
One notable point: GCC `-fpic` suppresses interprocedural optimizations including inlining for non-vague-linkage external linkage functions.
```c
// a.c -> a.o -> a.so
void f() { ... }
void g() { f(); }
```

Second case: f is defined in a different object file which will be linked into the same shared object.
```c
// a.c -> a.o
void f() { ... }

// b.c -> b.o
void f();
void g() { f(); }

// a.o b.o -> a.so
```

We will see when linking a shared object, that the first two cases need a PLT entry for `f` even if there is a local definition.

Third case: f is defined in a different shared object or the executable.
The symbol search on f cannot be prevented.
```c
// a.c -> a.o - a.so
void f() { ... }

// b.c -> b.o
void f();
void g() { f(); }

// b.o a.so -> b.so
```

You can see that in all three cases no annotation is required on f and g.

## ELF interposition

A component is an executable or shared object, sometimes called a module.
A dynamic symbol is `STB_GLOBAL` or `STB_WEAK`.

ELF interposition means the following properties. The next chapter will give a rigid definition.

* If a dynamic symbol is defined by multiple components, they don't conflict.
* For a symbol lookup (due to a relocation like `R_*_JUMP_SLOT`/`R_*_GLOB_DAT`/absolute relocation/etc), the definition from the first component wins.
* Definitions from subsequent components are overridden.

Let's see an example about duplicate definitions in two components.

```c
// a.c -> a.o -> a.so
// a.o -> a.a
void f() { ... }

// b.c -> b.o -> b.so
// b.o -> b.a
void f() { ... }
void g() { f(); }
```

In the static linking case, we have `ld ... a.a b.a`.
In the dynamic linking case, assuming `a.so` and `b.so` will be linked into the executable, we have `ld ... a.so b.so`.

So, is `ld ... a.so b.so` similar to `ld ... a.a b.a`?
I.e. Can we think of ELF interposition as an emulation of archive member extraction?

If `b.a(b.o)` is not extracted (i.e. the archive member does not define a symbol which is referenced previously and not provided by a preceding object file), `ld ... a.so b.so` is somewhat similar to `ld ... a.a b.a`.
The `g` definition in `b.so` is shadowed, just like the `g` definition in `b.a(b.o)` is not used.
(There is still a difference, though: `g` is defined in the dynamic linking case while undefined (because the archive member is not extracted) in the static linking case.)

However, if `b.a(b.o)` is extracted, we will get a duplicate definition error from `ld ... a.a b.a`.
Well, `ld ... a.so b.so` does not cause an error.

Therefore, I think "dynamic linking should be similar to static linking" as justification for interposition is lame.
Interposition is a simple scheme, and convenient in some scenarios, but inefficient, error-prone, and less secure.

## Specification

Since 2000-07-17, the ELF specification says the following for the `STV_DEFAULT` visibility (this is the default visibility. You get this unless you do thing like `-fvisibility=` or `__attribute__((visibility(...)))`):

> Global and weak symbols are also preemptable, that is, they may by preempted by (typo: be) definitions of the same name in another component."

In Chapter 5 Dynamic Linking, the specification says:

> When the dynamic linker creates the memory segments for an object file, the dependencies (recorded in DT_NEEDED entries of the dynamic structure) tell what shared objects are needed to supply the program's services. By repeatedly connecting referenced shared objects and their dependencies, the dynamic linker builds a complete process image. When resolving symbolic references, the dynamic linker examines the symbol tables with a breadth-first search. That is, it first looks at the symbol table of the executable program itself, then at the symbol tables of the DT_NEEDED entries (in order), and then at the second level DT_NEEDED entries, and so on. Shared object files must be readable by the process; other permissions are not required.

The wording remains unchanged since then, i.e. the evolution of dynamic linking has not contributed back to the specification.

This paragraph is probably difficult to follow. Let me rephrase it with some additions of dynamic loader behaviors.
The dynamic loader does one critical job: resolving dynamic relocations and binding symbol references from one component to another.
There is a flat namespace for symbol search.
The dynamic loader computes a breadth-first search list (`executable, needed0, needed1, needed2, needed0_of_needed0, needed1_of_needed0, ...`).
For each symbol reference, the dynamic loader iterates over the list and finds the first component which provides a definition.
(For `dlsym` with an explicit handle, the symbol search uses the dependency order, a breadth-first search rooted at the handle.)

The implication is that `STB_GLOBAL` and `STB_WEAK` definitions are equivalent in terms of symbol search. A `STB_WEAK` definition can preempt a `STB_GLOBAL` definition.

While not mentioned in the ELF specification, many dynamic loader implementations allow the environment variable `LD_PRELOAD` to inject shared objects.
The effect is like the `LD_PRELOAD` list is inserted at the beginning of the executable's `DT_NEEDED` list.
The search list may look like `executable, preload0, preload1, needed0, needed1, needed2, needed0_of_preload0, ..., needed0_of_needed0, needed1_of_needed0, ...`
(If the program calls `dlopen` with `RTLD_GLOBAL`, the newly loaded component and its dependencies (if not loaded) will be appended to the list.)
Here is the algorithm:

```text
fn load(c) {
  ...
  if tail != null { tail->next = &c }
  tail = &c
}

head = &exe
tail = null
load(exe)
for c in LD_PRELOAD {
  load(c)
}
for (c = head; c; c = c->next) {
  if c.loaded { continue; }
  for c1 in c.needed
    load(c1)
}
```

### Preemptibility in the linker

The executable is always the first element of the search list, so a defined symbol of any binding in the executable cannot be preempted (interposed).

For a shared object, a default visibility `STB_GLOBAL` or `STB_WEAK` symbol can be preempted (interposed) because a preceding component may define a symbol of the same name.

In the following example, `f` is preemptible even if it is locally defined.

```c
// At runtime a preceding component may also define f.
void f() { ... }
void g() { f(); }
```

To make it possible for the branch instruction to jump to a different definition, the linker resolves the branch target to a PLT entry.
The PLT entry has a code sequence loading the resolved address from an associated GOT entry.
The GOT entry is relocated by `R_*_JUMP_SLOT`.

Two places have costs:

* The dynamic loader performs a symbol search for `R_*_JUMP_SLOT`, and possibly for `R_*_GLOB_DAT`/absolute relocations.
* Every call site goes through a PLT indirection.

## Alternative symbol search models

Solaris named the above the default search model and introduced an alternative model: direct bindings.
With `-z defs`, one can ensure the dependencies are provided as part of the link and all symbol references are satisfied.
The linker can record the bound component for each symbol reference.

Here is an example from Solaris's Linkers and Libraries Guide:
```text
$ elfdump -y W.so.2
[6] [ DEPEND DIRECT ]          <self>  a
[7] [ DEPEND LAZY DIRECT ] [1] w.so.1  b
```

With the information about the component name, the dynamic loader can speed up its symbol search by just looking at one component.
In particular, frequently the bound component is the component itself.

Actually, with symbol versioning, the `Elf64_Verneed::vn_file` field carries the required information as well, but it isn't used to speed up the symbol search.

In Mac OS X, the two-level namespace introduced in 10.1 (default unless you use `ld -flat_namespace`) is a similar model.

Prelink can be conceived as a direct binding model without great ergonomics.

The ELF specification defines `DF_SYMBOLIC` which can be conceived as a special case of direct bindings.
When a shared object is marked as `DF_SYMBOLIC` (set by `ld -Bsymbolic`), the symbol search checks the shared object itself before starting the linear search from the executable.
It is quite common for a shared object to call `STV_DEFAULT` definitions in itself. `DF_SYMBOLIC` can improve the performance greatly.

## `-Bsymbolic`

The linker option `-Bsymbolic` can be used together with `-shared`. `ld -shared -Bsymbolic` is very similar to `-pie`.

`-Bsymbolic` follows ELF `DF_SYMBOLIC` semantics: all defined symbols are non-preemptible. This can optimize relocation processing:

* function calls: a branch instruction (e.g. `call foo@PLT`) will not create a PLT entry. The associated `R_*_JUMP_SLOT` dynamic relocation will be suppressed.
* variable access and function addresses: the GOT entry will not cause a `R_*_GLOB_DAT` dynamic relocation. On x86-64, with `R_X86_64_GOTPCRELX`/`R_X86_64_REX_GOTPCRELX`, the GOT indirection code sequence can be rewritten. However, the code sequence is still longer than that without GOT. On PowerPC64, there is a similar TOC optimization. On other architectures, there is no difference in code sequences.

`-fno-semantic-interposition` can save a PLT if all call sites are in the defining translation unit.
`-Bsymbolic` can address cross-translation-unit pessimization which cannot be optimized with `-fno-semantic-interposition`.
Personally I think `-Bsymbolic` claims most of the direct binding benefits.

As a data point, when building the Linux kernel's x86_64 defconfig with a `clang -fPIC` built clang, my build is 15% faster if I add `-Bsymbolic-functions` to `libLLVM.so` and `libclang-cpp.so`.
I cannot tell the performance difference with a mostly statically linked PIE clang.
From llvm-project 13.0.0 onwards, the build system uses `-Bsymbolic-functions` by default.

However, in practice, deployment of `-Bsymbolic` may run into pointer equality problems.
We will discuss variables and functions separately.
In practice the most serious problem is copy relocations.

### Pointer equality for variables

An inline variable with external linkage and a local static variable defined in an inline function with external linkage are required to be unique.
The address of such a variable seen by a `-Bsymbolic` linked shared object may be different from the address seen from outside the shared object.

Fortunately it is uncommon to export such a vague linkage variable to more than one component.
Some projects roll their own `typeid` mechanism and need caution, e.g. [`llvm::Any::TypeID`](https://reviews.llvm.org/D108943), [`mlir::TypeID`](https://reviews.llvm.org/D89153).

```cpp
// a.h
inline int *addr() {
  static int data;
  return &data;
}

// a.cc -> a.o -> a.so (-Bsymbolic)
#include "a.h"
int *addr0 = addr();

// b.cc -> b.o -> exe
#include "a.h"
int *addr1 = addr();
```

(ELF specific) In addition, a regular non-inline variable with external linkage can cause incompatibility problems due to copy relocations.
GCC/Clang `-fno-pic` emits direct access relocations referencing a global variable.
If the global variable turns out to be defined in a shared object, there will be a copy relocation in the executable.
The object the shared object sees and the executable sees will be different.

```cpp
// a.h
extern int var;

// a.cc -> a.o -> a.so (-Bsymbolic)
int var;
// clang -fpic (indirect):  movq var@GOTPCREL(%rip), %rax
// ld -Bsymbolic:           (non-GOTPCRELX) R_X86_64_RELATIVE or (GOTPCRELX) PC-relative
int *addr0() { return &var; }

// b.cc -> b.o -> exe
int main() {
  // clang -fno-pic (direct):  movl $var, %esi
  // clang -fpie/-fpic (indirect):  movq var@GOTPCREL(%rip), %rsi
  int *addr1 = &var;
}
```

For Clang `-fno-pic`, the direct access relocation can be avoided with `-fno-direct-access-external-data`.
GCC feature request: [PR98112](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98112).

<a name="HAVE_LD_PIE_COPYRELOC"></a>
Since GCC 5, on x86-64, `-fpie` can cause copy relocations as well due to `HAVE_LD_PIE_COPYRELOC`.
We should fix it. [Pending GCC patch](https://gcc.gnu.org/pipermail/gcc-patches/2021-May/570139.html).

(In C++, `typeid()` on an incomplete class can define a typeinfo name object.
A `-Bsymbolic` linked shared object may see a different copy, but the address can hardly cause a problem.).

### Pointer equality for functions

Many objects in C++ are not clearly part of a single object file, but are required by the ODR to have a single definition.
For example, C++ [dcl.inline]: "An inline function or variable with external or module linkage can be defined in multiple translation units ([basic.def.odr]), but is one entity with one address. A type or static variable defined in the body of such a function is therefore a single entity."

The address of an inline function seen by a `-Bsymbolic` linked shared object may be different from the address seen from outside the shared object.
Fortunately such cases are rare.
ELF/Mach-O programs may use `-fvisibility-inlines-hidden` to break such pointer equality.
On Windows, correct dllexport and dllimport are needed to make pointer equality work.
On the other hand, Windows link.exe enables identical COMDAT folding (`/OPT:ICF`) by default so different inline functions may have the same address.

On Mach-O, such symbols are placed into `__LINKEDIT,__weak_binding` so that dyld can coalesce the definitions across dylibs.

On Windows, you need to compile the DLL and the executable differently: the defining DLL needs `__declspec(dllexport)` on the inline function and the executable needs `__declspec(dllimport)`.
This is tricky.

```cpp
// a.cc -> a.obj -> a.dll
__declspec(dllexport) inline void f() {}
__declspec(dllexport) void *g() { return (void *)&f; }

// b.cc -> b.obj -> b.exe
// The address will be different if dllimport is omitted or dllexport is used.
__declspec(dllimport) inline void f() {}
```

(ELF specific) In addition, a regular non-inline function with external linkage can cause incompatibility problems due to canonical PLT entries.
GCC/Clang `-fno-pic` emits direct access relocations when taking the address of an external function.
If the global variable turns out to be defined in a shared object, there will be a canonical PLT entry in the executable.
The function address the shared object sees and the executable sees will be different.

```cpp
// a.h
void fun();

// a.cc -> a.o -> a.so (-Bsymbolic-functions)
void fun() {}
void *addr0() { return (void *)&fun; }

// b.cc -> b.o -> exe
// addr1() != addr0()
void *addr1() { return (void *)&fun; }
```

### `-Bsymbolic-functions`

The function incompatibility problems are uncommon.
It is often benign when the function address seen by a shared object is different from outside the shared object.
However, the variable case is usually severe: the executable and a shared object may act on different copies of a variable supposed to be the same entity.

In practice, we can usually use the linker option `-Bsymbolic-functions`.
The option applies to `STT_FUNC` symbols in ld.lld and non-`STT_OBJECT` symbols in GNU ld and gold, avoiding variable incompatibility problems.

### `-Bsymbolic-non-weak-functions`

The previous paragraph "Pointer equality for functions" has mentioned two problems. To address the pointer equality problem for vague linkage functions, <a name="-Bsymbolic-non-weak-functions"></a>we can introduce a linker option `-Bsymbolic-non-weak-functions` which applies to `STT_FUNC` `STB_GLOBAL` symbols to bypass vague linkage `STB_WEAK` symbols.
GNU ld feature request: [PR27871](https://sourceware.org/bugzilla/show_bug.cgi?id=27871).

`-Bsymbolic-non-weak-functions` provides an escape hatch for rare cases where definition interposition is needed: such declarations can be annotated as `__attribute__((weak))`.

<a name="-fno-pic_got"></a>
To address the canonical PLT entry problem, I think the best is to change the compiler's `-fno-pic` behavior: use GOT to take the address of an external default visibility function.
GCC feature request: [PR100593](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100593).
A stage 2 clang is byte identical when I apply the change.

Most pieces of software will just work, because using GOT is how `-fpie`/`-fpic` work.
The Linux kernel is an unfortunate outlier. Its x86 port has something like `asm("call %P[new2]" :: [new2]"i"(clear_page_rep));` and such `"i"` usage only works if the non-definition `clear_page_rep` is directly accessed.
In addition, more ports start to assume .got is empty.
I think the Linux kernel should use an explicit option to enable absolute relocations for function declaration addresses.

Technically we could add an alternative linker option `-Bsymbolic-plt` which resolves PLT-generating relocations at link time if the referenced symbol is defined and keeps the previous behavior for other relocation types (GOT-generating, absolute, PC-relative, etc).
However, I don't like making preemptibility different for different relocation types. In addition, once we fix the canonical PLT entry problem (we should, as it is evil like copy relocations), such an option will provide no additional usage (its effect on weak definitions can be solved with `-fvisibility-inlines=hidden`).

## Visibility

### `-fvisibility=protected`

A non-default visibility symbol cannot be preempted, even if the binding is `STB_WEAK`.
`-fvisibility=protected` can make all definitions protected and thus non-preemptible, nullifying the performance benefit of `-fno-semantic-interposition` and `-Bsymbolic`.
Note: if you want a definition to be preemptible, you will need a default visibility attribute, even if it is weak (e.g. `__attribute__((weak,visibility("default")))`).

However, `-fvisibility=protected` shares the same problem with `-Bsymbolic`: too coarse-grained.
It can cause the same sets of problems as discussed above in Pointer equality for variables and functions.
Notably, a shared object built with `-fvisibility=protected` is incompatible with `-fno-pic` global variable access.

In Clang, `-Xclang=-fapply-global-visibility-to-externs` applies the global visibility to `extern` declarations.

In GCC/binutils's x86 port, there were some changes in the wrong direction during the GCC 5 era.
As a result, there are some `STT_OBJECT` issues resulting in poor Clang interoperability (and also gold).

```text
% cat a.s
leaq foo(%rip), %rax

.data
.global foo
.protected foo
foo:
% gcc -fuse-ld=bfd -shared a.s
/usr/bin/ld.bfd: /tmp/ccchu3Xo.o: relocation R_X86_64_PC32 against protected symbol `foo' can not be used when making a shared object
/usr/bin/ld.bfd: final link failed: bad value
collect2: error: ld returned 1 exit status
```

See [Copy relocations, canonical PLT entries and protected visibility](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected) for details.
There is no problem when you only use Clang and ld.lld.

### `-fvisibility=hidden`

`-fvisibility=hidden` can make all definitions hidden and thus non-preemptible, nullifying the performance benefit of `-fno-semantic-interposition`.

`-fvisibility=hidden` requires annotation of exported symbols (`__attribute__((visibility("default")))`).
The explicit annotation sometimes makes it inconvenient to split and join libraries.

However, projects with Windows portability in mind will define macros to dispatch to either the visibility attribute or `__declspec(dllexport)`.

Personally I think that explicit annotation is the correct way of exposing API.
However, the adoption may not be that high.

### `-fvisibility-inlines-hidden`

The C++ specific `-fvisibility-inlines-hidden` is a safer subset of `-fvisibility=hidden`.
The option just violates pointer equality for inline function definitions.
As discussed above, this is usually safe.

## Interaction with `LD_PRELOAD`

There are several types of `LD_PRELOAD` usage.

First, use `LD_PRELOAD=same_soname.so` to replace a `DT_NEEDED` entry with the same SONAME. Both `-fno-semantic-interposition` and `-Bsymbolic` are compatible with such usage.

Second, use `LD_PRELOAD=malloc.so` to intercept some functions not defined in the application or any of its shared object dependencies.
Both `-fno-semantic-interposition` and `-Bsymbolic` are compatible.
Common examples include malloc replacement and fakeroot, both interposing some libc.so functions.
```c
void *f() { return malloc(0xb612); }
```

Third, use `LD_PRELOAD=different_soname.so` to replace a non-vague-linkage function defined in a shared object dependency and the SONAME is different.
(This usage is unlikely compatible with C++'s one definition rule.)
Such usage is incompatible with `-Bsymbolic`, `-Bsymbolic-functions`, and (if non-weak) `-Bsymbolic-non-weak-functions`.
If `-fno-semantic-interposition` causes an inlining and the call site is not intercepted, there is an incompatibility issue.

Note: it is unfair and usually incorrect to just state "it breaks LD_PRELOAD".
Please categorize your `LD_PRELOAD` use cases.

## The Last Alliance of ELF and Men

You may want to read my [_-fno-semantic-interposition_](/blog/2021-05-09-fno-semantic-interposition) first.
This section formulates an ambitious plan "the Last Alliance of ELF and Men" and also serves as a summary.
For easy navigation, You can click the `{}`-style links to jump back to the anchors defined in previous paragraphs.

### Choice (a) Compiler option oriented

We need a variant of `-fvisibility=protected` which applies to `STB_GLOBAL STT_FUNC` symbols.

The GNU ld protected symbol bugs described in [Copy relocations, canonical PLT entries and protected visibility](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected) must be fixed.

### Choice (b) Linker option oriented

We need `-Bsymbolic-non-weak-functions` (a variant of `-Bsymbolic-functions`) which only applies to `STB_GLOBAL` symbols (i.e. `STB_WEAK` symbols are excluded). [{-Bsymbolic-non-weak-functions}](#-Bsymbolic-non-weak-functions)

I wish that distributions default to `-fno-semantic-interposition` and (in the long term) `-Wl,-Bsymbolic-non-weak-functions`, bringing back the lost performance for decades.
macOS (Mach-O), Windows (PE-COFF), and Solaris (ELF) direct bindings have set up precedent so there is a good chance that most pieces of portable software are already in a good state.

We need a linker option to cancel default `-Bsymbolic-non-weak-functions`. I have added `-Bno-symbolic` to GNU ld and gold ([binutils 2.37; PR27834](https://sourceware.org/bugzilla/show_bug.cgi?id=27834)) and ld.lld 13.

(From Peter Smith) The linker can introduce a debugging option for executables to catch accidental interposition, say, `--warn-interposition`:
"Warning symbol S of type STT_FUNC is defined in executable A and shared objects B and C, using definition in A."

### Next steps

There is some amount of work to annotate software which cannot be built with the non-weak function `-fvisibility=protected` or `-Wl,-Bsymbolic-non-weak-functions`.

In return, I estimate that many pieces of software may be 5% to 20% faster (CPython is 1.3x faster) and a few percent smaller in size.

It turns out that some GCC folks don't like a configure-time option making `-fno-semantic-interposition` the default: [PR100937](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100937).

There is a trade-off and the downside is that `LD_PRELOAD` replacing a function definition in a shared object will be a non-default choice.
The users can build the software by themselves.

GCC `-fno-pic` should use GOT to take the address of an external default visibility function. [PR100593](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=100593). [{-fno-pic_got}](#-fno-pic_got)
Clang's behavior just emulates GCC.

The first stage of the plan is heavily throttled at GCC folks' mercy.
The second stage of the plan is throttled at distributions' interest.

### Copy relocations

With our function oriented non-interposition-by-default plan, we will not be blocked by copy relocation elimination.
Given the long historical issues, I don't expect that copy relocations can be addressed any time soon (say: 10 years).
Nevertheless, here is the plan.

We should fix the GCC 5 x86-64 `-fpie` `HAVE_LD_PIE_COPYRELOC` mistake. [PATCH](https://gcc.gnu.org/pipermail/gcc-patches/2021-May/570139.html). [{HAVE_LD_PIE_COPYRELOC}](#HAVE_LD_PIE_COPYRELOC)

GCC can introduce `-fno-direct-access-external-data` to avoid `-fno-pic` copy relocations. [PR98112](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98112).
In the x86-64 world, there are actually more problems with copy relocations and protected data symbols in GCC and GNU ld. I will recommend that interested readers read the summary of [Copy relocations, canonical PLT entries and protected visibility](/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected#summary).

### Trivia

When Android used GCC, [GCC passed `-Bsymbolic` to the linker by default](https://android.googlesource.com/platform/ndk/+/master/docs/ClangMigration.md).
