layout: post
title: All about symbol versioning
author: MaskRay
tags: [linker,llvm,binutils]
---

[中文版](/blog/zh/2020-11-26-all-about-symbol-versioning)

Updated in 2026-01.

Many people just want to know how to define or reference versioned symbols properly. You may jump to [Recommended usage](#recommended-usage) below.

In 1995, Solaris' link editor and ld.so introduced the symbol versioning mechanism.
Ulrich Drepper and Eric Youngdale borrowed Solaris' symbol versioning in 1997 and designed the GNU style symbol versioning for glibc.

gnu-gabi specification: <https://sourceware.org/gnu-gabi/program-loading-and-dynamic-linking.txt#:~:text=versioning>

<!-- more -->

When a shared object is updated and the behavior of a symbol changes, a `DT_SONAME` version bump is traditionally required to indicate ABI incompatibility (such as changing the type of parameters or return values).
The `DT_SONAME` version bump can be inconvenient when there are many dependent applications.
If we don't bump `DT_SONAME`, a dependent application/shared object built with the old version may run abnormally at run-time.

Symbol versioning provides a way to maintain backward compatibility without changing `DT_SONAME`.

The following part describes the representation, and then describes the behaviors from the perspectives of assembler, linker, and ld.so.
One may wish to skip the representation part when reading for the first time.

## Representation

In a shared object or executable file that uses symbol versioning, there are up to three sections. `.gnu.version_r` and `.gnu.version_d` are optional:

* `.gnu.version` (type `SHT_GNU_versym`, pointed to by `DT_VERSYM`): A parallel table to `.dynsym` containing N `uint16_t` version IDs, one per dynamic symbol.
* `.gnu.version_r` (type `SHT_GNU_verneed`, delimited by `DT_VERNEED`/`DT_VERNEEDNUM`): Describes versions required by undefined symbols.
* `.gnu.version_d` (type `SHT_GNU_verdef`, delimited by `DT_VERDEF`/`DT_VERDEFNUM`): Describes versions of defined symbols.

The parallel table design makes symbol versioning optional - ld.so implementations that ignore it (like musl) treat all references as binding to default versions.

```c
// Version definitions
typedef struct {
  Elf64_Half    vd_version;  // version: 1
  Elf64_Half    vd_flags;    // VER_FLG_BASE (index 1) or 0 (index != 1)
  Elf64_Half    vd_ndx;      // version index
  Elf64_Half    vd_cnt;      // number of associated aux entries, one plus number of parent versions
  Elf64_Word    vd_hash;     // SysV hash of the version name
  Elf64_Word    vd_aux;      // offset in bytes to the verdaux array
  Elf64_Word    vd_next;     // offset in bytes to the next verdef entry
} Elf64_Verdef;

typedef struct {
  Elf64_Word    vda_name;    // version name
  Elf64_Word    vda_next;    // offset in bytes to the next verdaux entry
} Elf64_Verdaux;

// Version needs
typedef struct {
  Elf64_Half    vn_version;  // version: 1
  Elf64_Half    vn_cnt;      // number of associated aux entries
  Elf64_Word    vn_file;     // .dynstr offset of the needed filename
  Elf64_Word    vn_aux;      // offset in bytes to vernaux array
  Elf64_Word    vn_next;     // offset in bytes to next verneed entry
} Elf64_Verneed;

typedef struct {
  Elf64_Word    vna_hash;    // SysV hash of vna_name
  Elf64_Half    vna_flags;   // usually 0; copied from vd_flags of the needed so
  Elf64_Half    vna_other;   // `Version:` in readelf -V output
  Elf64_Word    vna_name;    // .dynstr offset of the version name
  Elf64_Word    vna_next;    // offset in bytes to next vernaux entry
} Elf64_Vernaux;
```

`Verdef` entries describe version definitions in the module.
Each `Verdef` has one or more `Verdaux` entries (`vd_aux`): the first gives the version name, and subsequent entries (if any, `vda_next`) name parent versions for version inheritance (e.g., `v2 {} v1;`).
`vd_cnt` equals one plus the number of parent version definitions. However, it is not useful in practice—glibc and FreeBSD rtld ignore it, and ld.lld simply hard-codes it to 1.

`Verneed` entries describe version requirements from other shared objects (`vn_file`).
Each `Verneed` has one or more `Vernaux` entries specifying the required version names (`vna_name`).

Version names are not globally unique. The following cases are valid:

* A `Verdef` entry defines version `v1` while a `Verneed` entry requires version `v1` from `a.so`.
* A `Verneed` entry requires version `v1` from `a.so` while another `Verneed` entry requires version `v1` from `b.so`.

### `VER_FLG_WEAK`

GNU ld sets `VER_FLG_WEAK` in `Verdef::vd_flags` when a version node has no associated symbol, matching Solaris behavior.
[BZ24718#c15](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c15) proposed "set VER_FLG_WEAK on version reference if all symbols are weak" but was rejected.
A [2026 comment](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c19) requires to revisit whether `VER_FLG_WEAK` version references should support optional dependencies—e.g., an executable with a weak reference to `foo` links against `libA.so.1` (which provides `foo@@v1`), but at runtime runs with a `libA.so.1` that lacks the version definition for `v1`. Currently this causes an rtld error.

### Version index values

- Index 0 is called `VER_NDX_LOCAL` (misleading name, should have been `VER_NDX_NONE`). For defined symbols, the binding of the symbol will be changed to `STB_LOCAL`.
- Index 1 is called `VER_NDX_GLOBAL`. It has no special effect and is used for unversioned symbols.
- Index 2 to 0xffef are used for user defined versions.

Defined versioned symbols have two forms:

* `foo@@v2`, the default version.
* `foo@v2`, a non-default version (hidden version). The `VERSYM_HIDDEN` bit of the version ID is set.

Undefined versioned symbols have only the `foo@v2` form.

There is a special case: a version symbol referenced by a copy relocation in an executable.
The symbol acts as a definition in runtime relocation processing but its version ID references `.gnu.version_r` instead of `.gnu.version_d`.
The resolution of [PR28158](https://sourceware.org/bugzilla/show_bug.cgi?id=28158) picks the form `@`.

Usually versioned symbols are only defined in shared objects, but executables can have defined versioned symbols as well.
(When a shared object is updated, the old symbols are retained so that other shared objects do not need to be relinked, and executable files usually do not provide versioned symbols for other shared objects to reference.)

In the `.gnu.version` section (described by the `DT_VERSYM` tag), each symbol is assigned a version index:

- Unversioned undefined symbols use index 0. However, LLD before 22 and GNU ld versions between 2.35 and 2.45 use index 1. See <https://sourceware.org/bugzilla/show_bug.cgi?id=33577#c34>
- Versioned undefined symbols use index >= 2
- Unversioned defined symbols use index 1
- Versioned defined symbols use index >= 2

Defined symbols of index 0 are local and should not have an entry in `.dynsym` or `.gnu.version`.

### Example

`readelf -V` can dump the symbol versioning tables.

In the `.gnu.version_d` output below:

* Version index 1 (`VER_NDX_GLOBAL`) is the filename (soname if shared object). The `VER_FLG_BASE` flag is set.
* Version index 2 is a user defined version. Its name is `LUA_5.3`.

In the `.gnu.version_r` output below, each of version indexes 3~10 represents a version in a needed shared object. The name `GLIBC_2.2.5` appears thrice, each for a different shared object.

The `.gnu.version` table assigns a version index to each `.dynsym` entry.
An entry (version ID) corresponds to a `Index:` entry in `.gnu.version_d` or a `Version:` entry in `.gnu.version_r`.

```
% readelf -V /usr/bin/lua5.3

Version symbols section '.gnu.version' contains 248 entries:
 Addr: 0x0000000000002af4  Offset: 0x002af4  Link: 5 (.dynsym)
  000:   0 (*local*)       3 (GLIBC_2.3)     4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)
  004:   5 (GLIBC_2.3.4)   4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)
  ...

Version definition section '.gnu.version_d' contains 2 entries:
 Addr: 0x0000000000002ce8  Offset: 0x002ce8  Link: 6 (.dynstr)
  000000: Rev: 1  Flags: BASE  Index: 1  Cnt: 1  Name: lua5.3
  0x001c: Rev: 1  Flags: none  Index: 2  Cnt: 1  Name: LUA_5.3

Version needs section '.gnu.version_r' contains 3 entries:
 Addr: 0x0000000000002d20  Offset: 0x002d20  Link: 6 (.dynstr)
  000000: Version: 1  File: libdl.so.2  Cnt: 1
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 9
  0x0020: Version: 1  File: libm.so.6  Cnt: 1
  0x0030:   Name: GLIBC_2.2.5  Flags: none  Version: 6
  0x0040: Version: 1  File: libc.so.6  Cnt: 6
  0x0050:   Name: GLIBC_2.11  Flags: none  Version: 10
  0x0060:   Name: GLIBC_2.14  Flags: none  Version: 8
  0x0070:   Name: GLIBC_2.4  Flags: none  Version: 7
  0x0080:   Name: GLIBC_2.3.4  Flags: none  Version: 5
  0x0090:   Name: GLIBC_2.2.5  Flags: none  Version: 4
  0x00a0:   Name: GLIBC_2.3  Flags: none  Version: 3
```

### Symbol versioning in object files

The GNU scheme allows `.symver` directives to label the versions of the symbols in relocatable object files. The symbol names residing in .o contain `@` or `@@`.

## Assembler behavior

GNU as and LLVM integrated assembler provide implementation.

* `.symver foo, foo@v1`
  + If foo is undefined, produce `foo@v1`
  + If foo is defined, produce `foo` and `foo@v1` with the same binding (`STB_LOCAL`, `STB_WEAK`, or `STB_GLOBAL`) and `st_other` value (i.e. the same visibility). Personally I think this behavior is a design flaw <a name="gas-copy">{gas-copy}</a>. The proposed [V4 PATCH gas: Extend .symver directive](https://sourceware.org/pipermail/binutils/2020-April/110622.html) can address this problem.
* `.symver foo, foo@@v1`
  + If foo is undefined, error
  + If foo is defined, produce `foo` and `foo@v1` with the same binding and `st_other` value.
* `.symver foo, foo@@@v1`
  + If foo is undefined, produce `foo@v1`
  + If foo is defined, produce `foo@@v1`

With GNU as 2.35 ([PR25295](https://sourceware.org/bugzilla/show_bug.cgi?id=25295)) or Clang 13:

* `.symver foo, foo@v1, remove`
  + If foo is undefined, produce `foo@v1`
  + If foo is defined, produce `foo@v1`
  + This is a recommended way to define a non-default version symbol.
  + Unfortunately, in GNU as, `foo` cannot be used in a relocation ([PR28157](https://sourceware.org/bugzilla/show_bug.cgi?id=28157)).

## Linker behavior

The linker enters the symbol resolution stage after reading in object files, archive files, shared objects, LTO files, linker scripts, etc.

GNU ld uses indirect symbols to represent versioned symbols. There are complicated rules, and these rules are not documented.
The symbol resolution rules that I personally derived:

* Defined `foo` resolves undefined `foo` (traditional unversioned rule)
* Defined `foo@v1` resolves undefined `foo@v1` (a non-default version symbol is like a separate symbol)
* Defined `foo@@v1` (default version) resolves both undefined `foo` and `foo@v1`

If there are multiple default version definitions (such as `foo@@v1 foo@@v2`), a duplicate definition error should be issued even if one is weak. Usually a symbol has zero or one default version (`@@`) definition, and an arbitrary number of non-default version (`@`) definitions.

If the linker sees undefined `foo` and `foo@v1` first, it will treat them as two symbols. When the linker sees the definition `foo@@v1`, conceptually `foo` and `foo@@v1` should be combined. If the linker sees `foo@@v2` instead, `foo@@v2` should resolve `foo` and `foo@v1` should be a separate symbol.

* [Combining Versions](https://www.airs.com/blog/archives/220) describes the problem.
* `gold/symtab.cc Symbol_table::define_default_version` uses a heuristic rule to solve this problem. It special cases on visibility, but I feel that this rule is unneeded.
* Before 2.36, GNU ld reported a bogus multiple definition error for defined weak `foo@@v1` and defined global `foo@v1` [PR ld/26978](https://sourceware.org/bugzilla/show_bug.cgi?id=26978)
* Before 2.36, GNU ld had a bug that the visibility of undefined `foo@v1` does not affect the output visibility of `foo@@v1`: [PR ld/26979](https://sourceware.org/bugzilla/show_bug.cgi?id=26979)
* I fixed the object file side problem of ld.lld 12.0 in <https://reviews.llvm.org/D92259>
  `foo`
  Archive files and lazy object files may still have incompatibility issues.

When ld.lld sees a defined `foo@@v`, it adds both `foo` and `foo@v1` into the symbol table, thus `foo@@v1` can resolve both undefined `foo` and `foo@v1`.
After processing all input files, a pass iterates symbols and redirects `foo@v1` to `foo@@v1`.
Because ld.lld treats them as separate symbols during input processing, a defined `foo@v` cannot suppress the extraction of an archive member defining `foo@@v1`, leading to a behavior incompatible with GNU ld.
This probably does not matter, though.

If both `foo` and `foo@v1` are defined (at the same position), `foo` will be removed.
GNU ld has another strange behavior: if both `foo` and `foo@v1` are defined, `foo` will be removed.
I strongly believe it is an issue in GNU ld but the maintainer rejected [PR ld/27210](https://sourceware.org/bugzilla/show_bug.cgi?id=27210).
I implemented a similar hack in ld.lld 13.0.0 (<https://reviews.llvm.org/D107235>) but hoped binutils can fix the assembler issue (<https://sourceware.org/pipermail/binutils/2021-August/117677.html>).

ld.lld assigns version indexes as follows: first, each `Verdef` entry gets an index starting from 2; then, for each dynamic symbol resolving to a shared object definition, a new `Verneed`/`Vernaux` index is assigned if the (file, version) pair hasn't been seen.

### Version script

To define a versioned symbol in a shared object or an executable, a version script must be specified. If there is no defined versioned symbol, the version script can be omitted.

```
# Make all symbols other than foo and bar local.
{ global: foo; bar; local: *; };

# Assign version FBSD_1.0 to malloc and version FBSD_1.3 to mallocx,
# and make internal local.
FBSD_1.0 { malloc; local: internal; };
FBSD_1.3 { mallocx; };
```

A version script has three purposes:

* Define versions.
* Specify some patterns so that matched defined non-local symbols (which do not have `@` in the name) are tied to the specified version.
* Scope reduction
  + for a matched symbol, its binding will be changed to `STB_LOCAL` and will not be exported to the dynamic symbol table.
  + for a defined unversioned symbol, it can be matched by a `local:` pattern in any version node. E.g. `foo` can be matched by `v1 { local: foo; };`
  + for a defined versioned symbol, it can be matched by a `local:` pattern in the associated version node. E.g. both `foo@@v1` and `foo@v1` can be matched by `v1 { local: foo; };`.

A version script consists of one anonymous version tag (`{...};`) or a list of named version tags (`v1 {...};`).
If you use an anonymous version tag with other version tags, GNU ld will error: `anonymous version tag cannot be combined with other version tags`.
A `local:` part can be placed in any version tag. Which version tag is used does not matter.

If a defined symbol is matched by multiple version tags, the following precedence rules apply (`binutils-gdb/bfd/linker.c:find_version_for_sym`):

1. The first version tag with an exact pattern (i.e. there is no wildcard) wins.
2. Otherwise, the last version tag with a non-`*` wildcard pattern in `global:` wins.
3. Otherwise, the last version tag with a non-`*` wildcard pattern in `local:` wins.
4. Otherwise, the last version tag with a `*` pattern wins.

In gold and ld.lld, the rules are like:

1. The first version tag with an exact pattern (i.e. there is no wildcard) wins.
2. Otherwise, the last version tag with a non-`*` wildcard pattern wins. If the version tag has non-`*` wildcard patterns in both `global:` and `local:`, the `global:` one wins.
4. Otherwise, the last version tag with a `*` pattern wins. (Prior to LLD 18, [the first instead of the last](https://github.com/llvm/llvm-project/commit/21ac457f3f8f707b6eb79f15f52d366a31ee4385))

For example, given `v1 { local: p*;}; v2 { global: pq*;}; v3 { local: pqr*;};`, `local: pqr*` is selected for a defined non-local symbol `pqrs` in gold and ld.lld while `global: pq*` is slected in GNU ld.

`**` is also a catch-all pattern, but its precedence is higher than `*`.

GNU ld reports an error when a pattern appears in both `global:` and `local:`.

Most patterns are exact so gold and ld.lld iterate patterns instead of symbols to improve performance.

GNU ld and gold add an absolute symbol (`st_shndx=SHN_ABS`) for each defined version to `.symtab` and `.dynsym`.
ld.so does not need the symbol, so this behavior looks strange to me.

In a `-r` link, `--version-script` is ignored. Technically `local:` version nodes may be useful together with `-r`, but GNU ld and ld.lld just ignore `--version-script`.

### How a versioned symbol is produced

An undefined symbol can be assigned a version if:

* its name does not contain `@` (`.symver` is unused) and a shared object provides a default version definition.
* its name contains `@` and a shared object defines the symbol. GNU ld errors if there is no such a shared object. After <https://reviews.llvm.org/D92260>, ld.lld will report an error as well.

A defined symbol can be assigned a version if:

* its name does not contain `@` and it is matched by a pattern in a named version tag in a version script.
* its name contains `@`
  + If `-shared`, the version should be defined by a version script, otherwise GNU ld errors `version node not found for symbol`. This exception looks strange to me so I have filed [PR ld/26980](https://sourceware.org/bugzilla/show_bug.cgi?id=26980).
  + If `-no-pie` or `-pie`, a version definition is unneeded in GNU ld. This behavior is strange.

## Recommended usage

Personal recommendation:

To define a default version symbol, don't use `.symver`. Just list the symbol name in a version node in the version script.
If you really want to use `.symver`, use `.symver foo, foo@@@v2` so that `foo` is not present. If you require binutils>=2.35 or Clang>=13, `.symver foo, foo@@v2, remove` works as well.

To define a non-default version symbol, add a suffix to the original symbol name (`.symver foo_v1, foo@v1`) to prevent conflicts with `foo`. This will however leave (usually undesirable) `foo_v1`. If you don't strip `foo_v1` from the object file, you may localize it with a `local:` pattern in the version script. With the newer toolchain, you can use `.symver foo_v1, foo@v1, remove`.

```sh
cat > a.c <<e
__asm__(".symver foo_v1, foo@v1, remove");
void foo_v1() {}

void foo() {}
e
cat > a.ver <<e
v1 {};
v2 { foo; };
e

cc -fpic a.c -shared -Wl,--version-script=a.ver -o a.so
```

```sh
% readelf -W --dyn-syms a.so | grep @
     5: 0000000000001630     7 FUNC    GLOBAL DEFAULT   11 foo@@v2
     6: 0000000000001629     7 FUNC    GLOBAL DEFAULT   11 foo@v1
```

Most of the time, you want an undefined symbol to be bound to the default version symbol at link time.
It is usually unnecessary to set the version with `.symver`.

If you really need to set a version, either `.symver foo_v1, foo@@@v1` or `.symver foo_v1, foo@v1` is fine.

```sh
cat > b.c <<e
__asm__(".symver foo_v1, foo@v1"); // foo@@@v1 and foo@v1, remove work as well
void foo_v1();

void bar() {
  foo_v1();
}
e

cc -fpic b.c ./a.so -shared -o b.so
```

The reference is bound to the non-default version `foo@v1`:
```sh
% readelf -W --dyn-syms b.so | grep foo
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND foo@v1 (2)
```

If you omit `.symver`, the reference will be bound to the default version `foo@@v2`.

### Why is `.symver xxx, foo@v1` bad with a defined symbol?

There are two cases.

First, xxx is not `foo` (the unadorned name of the versioned symbol).
This is the most common usage.
Without loss of generality, we use `.symver foo_v1, foo@v1` as the example.

If the version script does not localize `foo_v1`, we will get `foo_v1` in `.dynsym`.
The extra symbol is almost always undesired.

Second, xxx is `foo`.
The `foo` definition can satisfy unversioned references from other TUs.
If you think about it, it is very rare for a non-default version definition to be used outside the TU.

```sh
# a.s
.symver foo, foo@v1
.globl foo
foo:

# b.s
# should reference foo@v1 instead of foo, if intended to use the non-default version symbol
```

If the version script contains `v1 {};`, the output will have just `foo@v` with GNU ld and ld.lld>=13.0.0.
The output will have both `foo` and `foo@v1` with gold and older ld.lld.

If the version script contains `v1 { foo; };`, the output will have just `foo@v1` with GNU ld, gold, and ld.lld>=13.0.0.
The output will have both `foo` and `foo@v1 with older ld.lld.

If the version script contains `v2 { foo; };`, the patterm will be ignored.
Unfortunately no linker reports a warning for this error-prone case.

---

Having distinct behaviors is unfortunate.
And the second case requires complexity in the linker internals.

## rtld behavior

_Linux Standard Base Core Specification, Generic Part_ describes the behavior of rtld (ld.so).
Kan added symbol versioning support to FreeBSD rtld in 2005.

The `DT_VERNEED` and `DT_VERNEEDNUM` tags in the dynamic table delimiter the version requirement by a shared object/executable file: the required (needed) versions and required (needed) shared object names (`Vernaux::vna_name`).

When an object with `DT_VERNEEDED` is loaded, glibc rtld performs some checks (`_dl_check_all_versions`).
For each Vernaux entry (a Verneed's auxiliary entry), glibc rtld checks whether the referenced shared object has a `DT_VERDEF` table.
If no, ld.so handles the case as a graceful degradation and prints `no version information available (required by %s)`;
if yes and the table does not define the version, ld.so reports (if the Vernaux entry has the `VER_FLG_WEAK` bit) a warning or (otherwise) an error. [verneed-check]

Usually a minor release does not bump soname. Suppose that libB.so depends on libA 1.3 (soname is libA.so.1) and calls a function which does not exist in libA 1.2. If PLT lazy binding is used, libB.so may seem to work on a system with libA 1.2, until the PLT of the 1.3 symbol is called.
If symbol versioning is not used and you want to solve this problem, you have to record the minor version number (`libA.so.1.3`) in the soname. However, bumping soname is all-or-nothing: all the dependent shared objects need to be relinked.
If symbol versioning is used, you can continue to use the soname `libA.so.1`. ld.so will report an error if libA 1.2 is used, because the 1.3 version required by libB.so does not exist.

When searching a definition for `foo`,

* for an object without `DT_VERSYM`
  + it can be bound to `foo`
* for an object with `DT_VERSYM`
  + it can be bound to `foo` of version `VER_NDX_GLOBAL`. This takes precendence over the next two rules
  + it can be bound to `foo` of any default version
  + it can be bound to `foo` of non-default version index 2 in relocation resolving phase (not dlsym/dlvsym). The rule retains compatibility when a shared object becomes versioned.

Note (undefined `foo` binding to `foo@v1` with version index 2) is allowed by ld.so but not allowed by the linker <a name="reject-non-default">{reject-non-default}</a>.
The rtld behavior is to retains compatibility when a shared object becomes versioned: the symbols with the smallest version (index 2) indicate the previously unversioned symbols.
If a new version of a shared object needs to deprecate an unversioned `bar`, you can remove `bar` and define `bar@compat` instead. Libraries using `bar` are unaffected but new linking against `bar` is disallowed.

When there are multiple versions of `foo`, `dlsym(RTLD_DEFAULT, ...)` returns the default version.
On glibc before 2.36, `dlsym(RTLD_NEXT, ...)` [returned the first version (BZ14932)](https://sourceware.org/bugzilla/show_bug.cgi?id=14932).
This was because in `elf/dl-sym.c:do_sym`, the `RTLD_NEXT` branch did not pass the flags `DL_LOOKUP_RETURN_NEWEST` to `dl_lookup_symbol_x`.
FreeBSD does not have the issue.

When searching a definition for `foo@v1`,

* for an object without `DT_VERSYM`
  + it can be bound to `foo`. In glibc, `elf/dl-lookup.c:check_match` asserts that the filename does not match the `vn_file` filename
* for an object with `DT_VERSYM`
  + it can be bound to `foo@v1` or `foo@@v1`
  + it can be bound to `foo` of version `VER_NDX_GLOBAL` in relocation resolving phase (not dlsym/dlvsym)

Say `b.so` references `malloc@GLIBC_2.2.5`. The executable defines an unversioned `malloc` due to linking in a malloc implementation.
At run-time, `malloc@GLIBC_2.2.5` in `b.so` will bind to the executable.
For example, address/memory/thread sanitizers leverage this behavior: shared objects do not need to link in interceptors; having the interceptor in the executable is sufficient.
libxml2 relied on the behavior to [drop versioning](https://gitlab.gnome.org/GNOME/libxml2/-/commit/bbb2b8f1360ee75a306a4dd1dae9e055cfe20125) on symbols while retaining compatibility for objects linking against older versions of libxml2.

When a versioned referenced is bound to a shared object without symbol versioning, in glibc versions before 2.41, `elf/dl-lookup.c:check_match` [used to assert that the filename did not match the `vn_file` filename](https://sourceware.org/cgit/glibc/commit/?h=bdaf50035354407add60d080d68fabe127330563)
```sh
echo 'void foo(); int main() { foo(); }' > a.c
echo 'v1 { foo; };' > c0.ver
echo 'void foo() {}' > c.c
sed 's/^        /\t/' > Makefile <<'eof'
.MAKE.MODE := meta curdirOk=1
CFLAGS := -fpic
LDFLAGS := -Wl,--no-as-needed

a: a.c c.so c0.so
        $(LINK.c) a.c c0.so -Wl,-rpath=$$PWD -o $@
c0.so: c.c c0.ver
        $(LINK.c) -shared -Wl,-soname=c.so,--version-script=c0.ver c.c -o $@
c.so: c.c
        $(LINK.c) -shared -Wl,-soname=c.so -nostdlib c.c -o $@
clean:
        rm -f a *.so *.o *.meta
eof
```
```
% bmake && ./a   # glibc<2.41
./a: /tmp/d/c.so: no version information available (required by ./a)
Inconsistency detected by ld.so: dl-lookup.c: 107: check_match: Assertion `version->filename == NULL || ! _dl_name_match_p (version->filename, map)' failed!

% bmake && ./a   # glibc>=2.41
./a: /tmp/d/c.so: no version information available (required by ./a)
```

This glibc<2.41 check is pretty dumb, as most shared objects have `DT_VERSYM` due to versioned references to libc like `__cxa_finalize@GLIBC_2.2.5` (from GCC `crtbeginS.o`).

Aside from this assertion, `vn_file` is essentially ignored for symbol search since glibc 2.30 [BZ24741](https://sourceware.org/PR24741).
Previously during relocation resolving, after an object failed to provide a match, if it matched `vn_file`, rtld would report an error `symbol %s version %s not defined in file %s with link time reference`.

[glibc 2.30 ld.so: Support moving versioned symbols between sonames [BZ #24741]](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=f0b2132b35248c1f4a80f62a2c38cddcc802aa8c) has a side benefit for weak references.
Previously, if `b.so` had a versioned weak reference `foo@v1` (where `v1` referenced `c.so`), rtld would error with `symbol %s version %s not defined in file %s with link time reference` when `c.so` lacked `foo@@v1` or `foo@v1`—contrary to weak reference semantics.
Newer rtld tolerates this as long as the runtime `c.so` defines version `v1`:

```sh
echo '#include <stdio.h>\nvoid fb(); int main() { fb(); puts("a"); }' > a.c
echo '__attribute__((weak)) void foo(); void fb() { if (foo) foo(); }' > b.c
echo 'v1 { foo; };' > c-link.ver
echo '#include <stdio.h>\nvoid foo() { puts("foo"); }' > c.c
echo 'v1 { };' > c.ver
echo 'v2 { };' > c2.ver
sed 's/^        /\t/' > Makefile <<'eof'
.MAKE.MODE := meta curdirOk=1
CFLAGS := -fpic

a: a.c b.so c-link.so c.so c2.so
	$(LINK.c) a.c b.so -Wl,-rpath=$$PWD -o $@
b.so: b.c c-link.so
	$(LINK.c) -shared -Wl,--no-as-needed $> -Wl,-rpath=$$PWD -o $@
c-link.so: c.c c.ver
	$(LINK.c) -shared -Wl,-soname=c.so,--version-script=c-link.ver c.c -o $@
c.so: c.c c.ver
	$(LINK.c) -shared -Wl,-soname=c.so,--version-script=c.ver -Dfoo=foo1 c.c -o $@
c2.so: c.c c2.ver
	$(LINK.c) -shared -Wl,-soname=c.so,--version-script=c2.ver -Dfoo=foo1 c.c -o $@
clean:
	rm -f a *.so *.o *.meta
eof
```

If `c.so` lacks the required version entirely, rtld still reports a fatal error:
```
% bmake
...
% ./a
a
% LD_PRELOAD=c2.so ./a
./a: /tmp/t/v2/c2.so: version `v1' not found (required by /tmp/t/v2/b.so)
```

The [BZ24718#319 comment](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c19) in 2026 asked `VER_FLG_WEAK` in `Vernaux::vna_flags` extension, allowing rtld to continue execution (with a verbose-mode warning `weak version ... not found`) instead of failing.

If the `v1` Verneed in `a` has the `VER_FLG_WEAK` flag, we shall see the following instead:

```
% LD_PRELOAD=c2.so ./a
./a: /tmp/t/v2/c2.so: weak version `v1' not found (required by /tmp/t/v2/b.so)
a
```

### Example

Run the following code to create `a.c b.c b0.c Makefile`, then run `bmake`.

```sh
cat > ./a.c <<'eof'
#define _GNU_SOURCE  // workaround before glibc 2.36 (commit 748df8126ac69e68e0b94e236ea3c2e11b1176cb)
#include <dlfcn.h>
#include <stdio.h>

int foo(int a);

int main(void) {
   int res = foo(0);
   printf ("foo(0) = %d, %s\n", res, res == 0 ? "ok" : "wrong");

   /* Resolve to foo@@v3 in b.so, instead of foo@v1 or foo@v2.  */
   int (*fp)(int) = dlsym(RTLD_DEFAULT, "foo");
   res = fp (0);
   printf ("foo(0) = %d, %s\n", res, res == 3 ? "ok" : "wrong");

   /* Resolve to foo@@v3 in b.so, instead of foo@v1 or foo@v2.  */
   fp = dlsym(RTLD_NEXT, "foo");
   res = fp(0);
   printf ("foo(0) = %d, %s\n", res, res == 3 ? "ok" : "wrong");
}
eof
echo 'int foo(int a) { return -1; }' > ./b0.c
cat > ./b.c <<'eof'

int foo_va(int a) { return 0; } asm(".symver foo_va, foo@va, remove");
int foo_v1(int a) { return 1; } asm(".symver foo_v1, foo@v1, remove");
int foo_v2(int a) { return 2; } asm(".symver foo_v2, foo@v2, remove");
int foo(int a) { return 3; } asm(".symver foo, foo@@@v3");
eof
echo 'va {}; v1 {} va; v2 {} v1; v3 {} v2;' > ./b.ver
sed 's/^        /\t/' > ./Makefile <<'eof'
.MAKE.MODE := meta curdirOk=1
CFLAGS = -fpic -g

a: a.o b0.so b.so
	$(CC) -Wl,--no-as-needed a.o b0.so -ldl -Wl,-rpath=$$PWD -o $@

b0.so: b0.o
	$(CC) $> -shared -Wl,--soname=b.so -o $@

b.so: b.o
	$(CC) $> -shared -Wl,--soname=b.so,--version-script=b.ver -o $@

clean:
	rm -f a *.so *.o *.meta
eof
```

Before glibc 2.36, the output is:
```text
% ./a
foo(0) = 0, ok
foo(0) = 3, ok
foo(0) = 0, wrong
```
Since 2.36, the last line is correct.

## Upgraded symbols in glibc

[When to prevent execution of new binaries with old glibc](https://sourceware.org/pipermail/libc-alpha/2021-October/132168.html) has a summary about when a new symbol version is introduced.

Note that GNU nm before binutils 2.35 does not display `@` or `@@`.

```sh
nm -D /lib/x86_64-linux-gnu/libc.so.6 | \
  awk '$2!="U" {i=index($3,"@"); if(i){v=substr($3,i); $3=substr($3,1,i-1); m[$3]=m[$3]" "v}} \
  END {for(f in m)if(m[f]~/@.+@/)print f, m[f]}'
```

The output on my x86-64 system:

```
pthread_cond_broadcast  @GLIBC_2.2.5 @@GLIBC_2.3.2
clock_nanosleep  @@GLIBC_2.17 @GLIBC_2.2.5
_sys_siglist  @@GLIBC_2.3.3 @GLIBC_2.2.5
sys_errlist  @@GLIBC_2.12 @GLIBC_2.2.5 @GLIBC_2.3 @GLIBC_2.4
quick_exit  @GLIBC_2.10 @@GLIBC_2.24
memcpy  @@GLIBC_2.14 @GLIBC_2.2.5
regexec  @GLIBC_2.2.5 @@GLIBC_2.3.4
pthread_cond_destroy  @GLIBC_2.2.5 @@GLIBC_2.3.2
nftw  @GLIBC_2.2.5 @@GLIBC_2.3.3
pthread_cond_timedwait  @@GLIBC_2.3.2 @GLIBC_2.2.5
clock_getres  @GLIBC_2.2.5 @@GLIBC_2.17
pthread_cond_signal  @@GLIBC_2.3.2 @GLIBC_2.2.5
fmemopen  @GLIBC_2.2.5 @@GLIBC_2.22
pthread_cond_init  @GLIBC_2.2.5 @@GLIBC_2.3.2
clock_gettime  @GLIBC_2.2.5 @@GLIBC_2.17
sched_setaffinity  @GLIBC_2.3.3 @@GLIBC_2.3.4
glob  @@GLIBC_2.27 @GLIBC_2.2.5
sys_nerr  @GLIBC_2.2.5 @GLIBC_2.4 @@GLIBC_2.12 @GLIBC_2.3
_sys_errlist  @GLIBC_2.3 @GLIBC_2.4 @@GLIBC_2.12 @GLIBC_2.2.5
sys_siglist  @GLIBC_2.2.5 @@GLIBC_2.3.3
clock_getcpuclockid  @GLIBC_2.2.5 @@GLIBC_2.17
realpath  @GLIBC_2.2.5 @@GLIBC_2.3
sys_sigabbrev  @GLIBC_2.2.5 @@GLIBC_2.3.3
posix_spawnp  @@GLIBC_2.15 @GLIBC_2.2.5
posix_spawn  @@GLIBC_2.15 @GLIBC_2.2.5
_sys_nerr  @@GLIBC_2.12 @GLIBC_2.4 @GLIBC_2.3 @GLIBC_2.2.5
nftw64  @GLIBC_2.2.5 @@GLIBC_2.3.3
pthread_cond_wait  @GLIBC_2.2.5 @@GLIBC_2.3.2
sched_getaffinity  @GLIBC_2.3.3 @@GLIBC_2.3.4
clock_settime  @GLIBC_2.2.5 @@GLIBC_2.17
glob64  @@GLIBC_2.27 @GLIBC_2.2.5
```

* `realpath@@GLIBC_2.3`: the previous version returns `EINVAL` when the second parameter is `NULL`
* `memcpy@@GLIBC_2.14` [BZ12518](https://sourceware.org/bugzilla/show_bug.cgi?id=12518): the previous version guarantees a forward copying behavior. Shockwave Flash at that time had a "memcpy downward" bug which required the workaround.
* `quick_exit@@GLIBC_2.24` [BZ20198](https://sourceware.org/bugzilla/show_bug.cgi?id=20198): the previous version copies the destructors of thread_local objects.
* `glob64@@GLIBC_2.27`: the previous version does not follow dangling symlinks.

## How to remove symbol versioning

Imagine that you want to build an application with a prebuilt shared object which has versioned references, but you can only find shared objects providing the unversioned definitions. The linker will helpfully error:
```
ld.lld: error: undefined reference to foo@v1 [--no-allow-shlib-undefined]
```

As the diagnostic suggests, you can add `--allow-shlib-undefined` to get rid of the error. It is not recommended but the built application may happen to work.

For this case, an alternative hacky solution is:

```sh
# 64-bit
cp in.so out.so
rizin -wqc '/x feffff6f00000000 @ section..dynamic; w0 16 @ hit0_0' out.so
llvm-objcopy -R .gnu.version out.so

# 32-bit
cp in.so out.so
rizin -wqc '/x feffff6f @ section..dynamic; w0 8 @ hit0_0' out.so
llvm-objcopy -R .gnu.version out.so
```

With the removal of `.gnu.version`, the linker will think that `out.so` references `foo` instead of `foo@v1`.
However, llvm-objcopy will zero out the section contents. At runtime, glibc ld.so will complain `unsupported version 0 of Verneed record`.
To make glibc happy, you can delete `DT_VER*` tags from the dynamic table. The above code snippet uses an r2 command to locate `DT_VERNEED`(0x6ffffffe) and rewrite it to `DT_NULL`(a `DT_NULL` entry stops the parsing of the dynamic table). The difference of the `readelf -d` output is roughly:

```diff
  0x000000006ffffffb (FLAGS_1)            Flags: NOW
- 0x000000006ffffffe (VERNEED)            0x8ef0
- 0x000000006fffffff (VERNEEDNUM)         5
- 0x000000006ffffff0 (VERSYM)             0x89c0
- 0x000000006ffffff9 (RELACOUNT)          1536
  0x0000000000000000 (NULL)               0x0
```

## ld.lld

* If an undefined symbol is not defined by a shared object, GNU ld will report an error. ld.lld before 12.0 did not error (I fixed it in <https://reviews.llvm.org/D92260>).

## GNU function attribute

There is a GNU function attribute which is lowers to a `.symver` assembly directive.
The attribute is implemented by GCC but not by Clang.

```cpp
extern "C" __attribute__((symver("foo@@v2"))) void foo() {}
extern "C" __attribute__((symver("foo@v1"))) void foo_v1() {}
```

Unfortunately, `@@@` and `,remove` are not supported.
Along with the reason that Clang does not implement the function attribute, I discourage using this feature.

## Remarks

GCC/Clang supports asm specifier and `#pragma redefine_extname` renaming a symbol. For example, if you declare `int foo() asm("foo_v1");` and then reference foo, the symbol in .o will be `foo_v1`.

For example, the biggest change in musl v1.2.0 is the time64 support for its supported 32-bit architectures.
musl adopted a scheme based on asm specifiers:

```c
// include/features.h
#define __REDIR(x,y) __typeof__(x) x __asm__(#y)

// API header include/sys/time.h
int utimes(const char *, const struct timeval [2]);
__REDIR(utimes, __utimes_time64);

// Implementation src/linux/utimes.c
int utimes(const char *path, const struct timeval times[2]) { ... }

// Internal header compat/time32/time32.h
int __utimes_time32() __asm__("utimes");

// Compat implementation compat/time32/utimes_time32.c
int __utimes_time32(const char *path, const struct timeval32 times32[2]) { ... }
```

* In .o, the time32 symbol remains `utimes` and is compatible with the ABI required by programs linked against old musl versions; the time64 symbol is `__utimes_time64`.
* The public header redirects `utimes` to `__utimes_time64`.
  + cons: if the user declares `utimes` by themself, they will not link against the correct `__utimes_time64`.
* The "good-looking" name `utimes` is used for the preferred time64 implementation internally and the "ugly" name `__utimes_time32` is used for the legacy time32 implementation.
  + If the time32 implementation is called elsewhere, the "ugly" name can make it stand out.

For the above example, here is an implementation with symbol versioning:

```c
// API header include/sys/time.h
int utimes(const char *, const struct timeval [2]);

// Implementation src/linux/utimes.c
int utimes(const char *path, const struct timeval times[2]) { ... }

// Internal header compat/time32/time32.h
// Probably __asm__(".symver __utimes_time32, utimes@time32, rename"); if supported
__asm__(".symver __utimes_time32, utimes@time32");

// Implementation compat/time32/utimes_time32.c
int __utimes_time32(const char *path, const struct timeval32 times32[2])
{
  ...
}
```

Note that `@@@` cannot be used. The header is included in a defining translation unit and `@@@` will lead to a default version definition while we want a non-default version definition.

According to [Assembler behavior](#Assembler-behavior), the undesirable `__utimes_time32` is present. Be careful to use a version script to localize it.

So what is the significance of symbol versioning? I think carefully:

* Refuse linking against old symbols while keeping compatibility with unversioned old libraries. [{reject-non-default}](#reject-non-default)
* No need to label declarations.
* The version definition can be delayed until link time. The version script provides a flexible pattern matching mechanism to assign versions.
* Scope reduction. Arguably another mechanism like `--dynamic-list` might have been developed if version scripts did not provide `local:`.
* There are some semantic issues in renaming builtin functions with asm specifiers in GCC and Clang (they do not know that the renamed symbol has built-in semantic). See [2020-10-15-intra-call-and-libc-symbol-renaming](2020-10-15-intra-call-and-libc-symbol-renaming)
* [verneed-check]

For the first item, the asm specifier scheme uses conventions to prevent problems (users should include the header); and symbol versioning can be forced by ld.

Design flaws:

* `.symver foo, foo@v1` <a name="gas-copy">{gas-copy}</a>
* Verdaux is a bit redundant. In practice, one Verdef has only one auxiliary Verdaux entry.
* This is arguably a minor problem but annoying for a framework providing multiple shared objects. ld.so requires "a versioned symbol is implemented in the same shared object in which it was found at link time", which disallows moving definitions between shared objects. Fortunately, glibc 2.30 [BZ24741](https://sourceware.org/PR24741) relaxes this requirement, essentially ignoring `Vernaux::vna_name`.

Before that, glibc used a forwarder to move `clock_*` functions from librt.so to libc.so:

```c
// rt/clock-compat.c
__typeof(clock_getres) *clock_getres_ifunc(void) asm("clock_getres");
__typeof(clock_getres) *clock_getres_ifunc(void) { return &__clock_getres; }
```

libc.so defines `__clock_getres` and `clock_getres`. librt.so defines an ifunc called `clock_getres` which forwards to libc.so `__clock_getres`.

## Related links

* [Combining Versions](http://www.airs.com/blog/archives/220)
* [Version Scripts](http://www.airs.com/blog/archives/300)
* <https://invisible-island.net/ncurses/ncurses-mapsyms.html>
