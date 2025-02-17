layout: post
title: All about symbol versioning
author: MaskRay
tags: [linker,llvm,binutils]
---

[中文版](#中文版)

Updated in 2023-08.

Many people just want to know how to define or reference versioned symbols properly. You may jump to [Recommended usage](#recommended-usage) below.

In 1995, Solaris' link editor and ld.so introduced the symbol versioning mechanism.
Ulrich Drepper and Eric Youngdale borrowed Solaris' symbol versioning in 1997 and designed the GNU style symbol versioning for glibc.

<!-- more -->

When a shared object is updated and the behavior of a symbol changes, a `DT_SONAME` version bump is traditionally required to indicate ABI incompatibility (such as changing the type of parameters or return values).
The `DT_SONAME` version bump can be inconvenient when there are many dependent applications.
If we don't bump `DT_SONAME`, a dependent application/shared object built with the old version may run abnormally at run-time.

Symbol versioning provides a way to maintain backward compatibility without changing `DT_SONAME`.

The following part describes the representation, and then describes the behaviors from the perspectives of assembler, linker, and ld.so.
One may wish to skip the representation part when reading for the first time.

## Representation

In a shared object or executable file that uses symbol versioning, there are up to three sections related to symbol versioning. `.gnu.version_r` and `.gnu.version_d` among them are optional:

* `.gnu.version` (version symbol section of type `SHT_GNU_versym`). The `DT_VERSYM` tag in the dynamic table points to the section. Assuming there are N entries in `.dynsym`, `.gnu.version` contains N `uint16_t` values, with the i-th entry indicating the version ID of the i-th symbol. Put it another way, `.gnu.version` is a parallel table to `.dynsym`.
* `.gnu.version_r` (version requirement section of type `SHT_GNU_verneed`). The `DT_VERNEED`/`DT_VERNEEDNUM` tags in the dynamic table delimiter this section. This section describes the version information used by the undefined versioned symbol in the module.
* `.gnu.version_d` (version definition section of type `SHT_GNU_verdef`). The `DT_VERDEF`/`DT_VERDEFNUM` tags in the dynamic table delimiter this section. This section describes the version information used by the defined versioned symbols in the module.

```c
// Version definitions
typedef struct {
  Elf64_Half    vd_version;  // version: 1
  Elf64_Half    vd_flags;    // VER_FLG_BASE (index 1) or 0 (index != 1)
  Elf64_Half    vd_ndx;      // version index
  Elf64_Half    vd_cnt;      // number of associated aux entries, one plus number of children version definitions
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

Currently GNU ld sets the `VER_FLG_WEAK` flag when a version node has no symbol associated with it.
This behavior matches Solaris.
[BZ24718#c15](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c15) proposed "set VER_FLG_WEAK on version reference if all symbols are weak" and was rejected.

`vd_cnt` is one plus the number of children version definitions. `vd_cnt` is not used by glibc or FreeBSD rtld.
ld.lld just always sets the field to 1.

The advantage of using a parallel table for `.gnu.version` is that symbol versioning is optional. ld.so implementations which do not support symbol versioning can freely assume no symbol has a version.
The behavior is that all references as if bind to the default version definitions.
musl ld.so falls into this category.

### Version index values

Index 0 is called `VER_NDX_LOCAL`. The binding of the symbol will be changed to `STB_LOCAL`.
Index 1 is called `VER_NDX_GLOBAL`. It has no special effect and is used for unversioned symbols.
Index 2 to 0xffef are used for user defined versions.

Defined versioned symbols have two forms:

* `foo@@v2`, the default version.
* `foo@v2`, a non-default version (hidden version). The `VERSYM_HIDDEN` bit of the version ID is set.

Undefined versioned symbols have only the `foo@v2` form.

There is a special case: a version symbol referenced by a copy relocation in an executable.
The symbol acts as a definition in runtime relocation processing but its version ID references `.gnu.version_r` instead of `.gnu.version_d`.
The resolution of [PR28158](https://sourceware.org/bugzilla/show_bug.cgi?id=28158) picks the form `@`.

Usually versioned symbols are only defined in shared objects, but executables can have defined versioned symbols as well.
(When a shared object is updated, the old symbols are retained so that other shared objects do not need to be relinked, and executable files usually do not provide versioned symbols for other shared objects to reference.)

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
The output will have both `foo` and `foo@v1 with gold and older ld.lld.

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

In glibc, when a versioned referenced is bound to a shared object without symbol versioning, `elf/dl-lookup.c:check_match` asserts that the filename does not match the `vn_file` filename.
If the filename matches, glibc thinks that the runtime shared object is older than the link-time shared object, and will report an assertion error.
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
% bmake && ./a
./a: /tmp/d/c.so: no version information available (required by ./a)
Inconsistency detected by ld.so: dl-lookup.c: 107: check_match: Assertion `version->filename == NULL || ! _dl_name_match_p (version->filename, map)' failed!
```

However, this check is pretty dumb, as most shared objects have `DT_VERSYM` due to versioned references to libc like `__cxa_finalize@GLIBC_2.2.5` (from GCC `crtbeginS.o`).

Aside from this assertion, `vn_file` is essentially ignored for symbol search since glibc 2.30 [BZ24741](https://sourceware.org/PR24741).
Previously during relocation resolving, after an object failed to provide a match, if it matched `vn_file`, rtld would report an error `symbol %s version %s not defined in file %s with link time reference`.

[ld.so: Support moving versioned symbols between sonames [BZ #24741]](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=f0b2132b35248c1f4a80f62a2c38cddcc802aa8c) has a side benefit.
Previously, if `b.so` has a versioned weak reference `foo@v1` where `v1` references `c.so`, rtld would report an error `symbol %s version %s not defined in file %s with link time reference` if `c.so` does not define `foo@@v1` or `foo@v1`.
This behavior did not match the expectation of weak references.
Newer rtld no longer report the error.
```sh
echo 'void fb(); int main() { fb(); }' > a.c
echo '__attribute__((weak)) void foo(); void fb() { if (foo) foo(); }' > b.c
echo 'v1 { foo; };' > c0.ver
echo 'void foo() {}' > c.c
echo 'v1 { };' > c.ver
sed 's/^        /\t/' > Makefile <<'eof'
.MAKE.MODE := meta curdirOk=1
CFLAGS := -fpic

a: a.c b.so c.so c0.so
	$(LINK.c) a.c b.so -Wl,-rpath=$$PWD -o $@
b.so: b.c c0.so
	$(LINK.c) -shared $> -Wl,-rpath=$$PWD -o $@
c0.so: c.c c.ver
	$(LINK.c) -shared -Wl,-soname=c.so,--version-script=c0.ver c.c -o $@
c.so: c.c c.ver
	$(LINK.c) -shared -Wl,-soname=c.so,--version-script=c.ver -Dfoo=foo1 c.c -o $@
clean:
	rm -f a *.so *.o *.meta
eof
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

# 中文版

1995年Solaris的link editor和ld.so引入了symbol versioning機制。
Ulrich Drepper和Eric Youngdale在1997年借鑑Solaris symbol versioning，設計了用於glibc的GNU風格symbol versioning。

一個shared object更新，某個符號的行爲變更(ABI改變(如變更參數或返回值的類型)或行爲變化)時，傳統上可以bump `DT_SONAME`：依賴的shared objects必須重新編譯、鏈接才能繼續使用；如果不改變`DT_SONAME`，依賴的shared objects可能悄悄地產生異常行爲。
使用symbol versioning可以提供不改變`DT_SONAME`的backward compatibility。

下面描述表示方式，然後從assembler、鏈接器、ld.so幾個角度描述symbol versioning行爲。初次閱讀時不妨跳過表示方式部分。

## 表示方式

在使用symbol versioning的shared object或可執行檔中，有至多三個symbol versioning相關的sections，其中`.gnu.version_r`和`.gnu.version_d`是可選的：

* `.gnu.version`(version symbol section)。dynamic table中的`DT_VERSYM` tag指向該section。假設`.dynsym`有N個entries，那麼`.gnu.version`包含N個uint16_t。第i個entry描述第i個dynamic symbol table所屬的version
* `.gnu.version_r`(version requirement section)。dynamic table中的`DT_VERNEED`/`DT_VERNEEDNUM` tags標記該section。描述該模塊的未定義的versioned符號用到的version信息
* `.gnu.version_d`(version definition section)。dynamic table中的`DT_VERDEF`/`DT_VERDEFNUM` tags標記該section。記錄該模塊定義的versioned符號用到的version信息

```c
// Version definitions
typedef struct {
  Elf64_Half    vd_version;  // version: 1
  Elf64_Half    vd_flags;    // VER_FLG_BASE (index 1) or 0 (index != 1)
  Elf64_Half    vd_ndx;      // version index
  Elf64_Half    vd_cnt;      // number of associated aux entries, one plus number of children version definitions
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
  Elf64_Half    vna_other;   // unused
  Elf64_Word    vna_name;    // .dynstr offset of the version name
  Elf64_Word    vna_next;    // offset in bytes to next vernaux entry
} Elf64_Vernaux;
```

目前GNU ld不會設置`VER_FLG_WEAK`。[BZ24718#c15](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c15)提議"set VER_FLG_WEAK on version reference if all symbols are weak"。

使用一個parallel table的好處是：不支持(忽略`DT_VERSYM,DT_VERDEF,DT_VERNEED`)symbol versioning的ld.so也能繼續工作，就好像所有符號都沒有version一樣。
musl ld.so就屬於此類。

### Version index values

Index 0稱爲`VER_NDX_LOCAL`。Version id爲0的符號的binding將會更改爲`STB_LOCAL`。
Index 1稱爲`VER_NDX_GLOBAL`。沒有特殊作用，用於unversioned符號。
Index 2到0xffef用於其他versions。

定義的versioned符號有兩種形式：

* `foo@@v2`，稱爲default version
* `foo@v2`，稱爲non-default version，也叫hidden version，其version id設置了`VERSYM_HIDDEN` bit

未定義符號只有`foo@v2`這一種形式。

通常只在shared object中定義versioned符號，但可執行檔也是可以獲得versioned符號的。
(一個shared object更新時保留舊符號，可以使其他shared objects不須重新鏈接，而可執行檔通常不提供versioned符號供其他shared objects引用。)

### 例子

`readelf -V`可以導出symbol versioning表。

下面輸出的`.gnu.version_d` section裏：

* Version index 1 (`VER_NDX_GLOBAL`) is the filename (soname if shared object). The `VER_FLG_BASE` flag is set.
* Version index 2 is a user defined version. Its name is `LUA_5.3`.

下面輸出的`.gnu.version_r` section裏，version index 3~10每一個都表示了一個依賴的shared object。名字`GLIBC_2.2.5`出現了三次，每一次給一個不同的shared object。

`.gnu.version`表給每個`.dynsym`符號分配了一個version index。

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

這套GNU symbol versioning允許`.symver` directive在.o裏標註符號的version。在.o裏符號名字面包含`@`或`@@`。

## Assembler行爲

GNU as和LLVM integrated assembler提供實現。

* 對於`.symver foo, foo@v1`
  + 如果foo未定義，.o中有一個名爲`foo@v1`的符號
  + 如果foo被定義，.o中有兩個符號：`foo`和`foo@v1`，兩者的binding一致(均爲`STB_LOCAL`，或均爲`STB_WEAK`，或均爲`STB_GLOBAL`)，`st_other`一致(visibility一致)。個人認爲這個行爲是設計缺陷<a name="gas-copy">{gas-copy}</a>
* 對於`.symver foo, foo@@v1`
  + 如果foo未定義，assembler報錯
  + 如果foo被定義，.o中有兩個符號：`foo`和`foo@@v1`，兩者的binding和`st_other`一致
* 對於`.symver foo, foo@@@v1`
  + 如果foo未定義，.o中有一個名爲`foo@v1`的符號
  + 如果foo被定義，.o中有一個名爲`foo@@v1`的符號

With GNU as 2.35 ([PR25295](https://sourceware.org/bugzilla/show_bug.cgi?id=25295)) or Clang 13:

* `.symver foo, foo@v1, remove`
  + 如果foo未定義，.o中有一個名爲`foo@v1`的符號
  + 如果foo被定義，.o中有一個名爲`foo@v1`的符號
  + 我推薦用這種方式定義non-default符號
  + Unfortunately, in GNU as, `foo` cannot be used in a relocation ([PR28157](https://sourceware.org/bugzilla/show_bug.cgi?id=28157)).

## 鏈接器行爲

鏈接器在讀入object files、archive files、shared objects、LTO files、linker scripts等後就進入符號解析階段。

GNU ld用indirect symbol表示versioned符號，在很多階段都有複雜的規則，這些規則都沒有文檔。
我個人得出的符號解析規則：

* 定義的`foo`可以滿足未定義的`foo`(傳統unversioned符號規則)
* 定義的`foo@v1`可以滿足未定義的`foo@v1`
* 定義的`foo@@v1`可以同時滿足未定義的`foo`和`foo@v1`

若存在多個default version的定義(如`foo@@v1 foo@@v2`)，觸發duplicate definition error。通常一個符號有零或一個default version(`@@`)定義，任意個non-default version(`@`)定義。

ld.lld的實現中，看到shared object中的`foo@@v1`則在符號表中同時插入`foo`和`foo@v1`，因此可以滿足未定義的`foo`和`foo@v1`。

鏈接器如果先看到未定義`foo`和`foo@v1`，會把它們當作兩個符號。之後看到定義的`foo@@v1`時，概念上應該合併`foo`和`foo@@v1`。若看到的是定義的`foo@@v2`，應該用`foo@@v2`滿足`foo`，而`foo@v1`仍是一個不同的符號。

* [Combining Versions](https://www.airs.com/blog/archives/220)描述了這個問題
* `gold/symtab.cc Symbol_table::define_default_version`用一個啓發式規則處理這個問題。它特殊判斷了visibility，但我感覺這個規則可能不需要也行
* Before 2.36, GNU ld reported a bogus multiple definition error for defined weak `foo@@v1` and defined global `foo@v1` [PR ld/26978](https://sourceware.org/bugzilla/show_bug.cgi?id=26978)
* Before 2.36, GNU ld had a bug that the visibility of undefined `foo@v1` does not affect the output visibility of `foo@@v1`: [PR ld/26979](https://sourceware.org/bugzilla/show_bug.cgi?id=26979)
* I fixed the object file side problem of ld.lld 12.0 in <https://reviews.llvm.org/D92259>
  `foo`
  Archive files and lazy object files may still have incompatibility issues.

When ld.lld sees a defined `foo@@v`, it adds both `foo` and `foo@v1` into the symbol table, thus `foo@@v1` can resolve both undefined `foo` and `foo@v1`.
After processing all input files, a pass iterates symbols and redirects `foo@v1` to `foo@@v1`.
Because ld.lld treats them as separate symbols during input processing, a defined `foo@v` cannot suppress the extraction of an archive member defining `foo@@v1`, leading to a behavior incompatible with GNU ld.
This probably does not matter, though.

GNU ld has another strange behavior: if both `foo` and `foo@v1` are defined, `foo` will be removed.
I strongly believe it is an issue in GNU ld but the maintainer rejected [PR ld/27210](https://sourceware.org/bugzilla/show_bug.cgi?id=27210).

### Version script

在輸出的shared object或可執行檔中定義version必須指定version script。若所有versioned符號均爲未定義狀態則無需version script。

```
# Make all symbols other than foo and bar local.
{ global: foo; bar; local: *; };

# Assign version FBSD_1.0 to malloc and version FBSD_1.3 to mallocx,
# and make internal local.
FBSD_1.0 { malloc; local: internal; };
FBSD_1.3 { mallocx; };
```

Version script有三個用途：

* 定義versions
* 指定一些模式，使得匹配的、定義的、unversioned的符號具有指定的version
* Scope reduction
  + 對於一個被`local:`模式匹配的符號，如果它是定義的、unversioned的，那麼它的binding會被更改爲`STB_LOCAL`，不會導出到dynamic symbol table
  + 對一個定義的unversioned符號，它可以被任何version node內的`local:`模式匹配
  + 對一個定義的versioned符號，它可以被相應的version node內的`local:`模式匹配。例如，`foo@@v1`和`foo@v1`都能被`v1 { local: foo; };`匹配

一個version script由一個anonymous version tag (`{...};`)，或若干named version tags (`v1 {...};`)組成。
如果一個anonymous version tag和其他version tag一起使用，GNU ld會報錯`anonymous version tag cannot be combined with other version tags`。
`local:`可以放在任意version tag裏。

如果一個定義的符號被多個version tags匹配，如下的優先級規則適用(`binutils-gdb/bfd/linker.c:find_version_for_sym`)：

* 第一個exact pattern的version tag
* 最後一個非`*`的wildcard pattern的version tag
* 第一個含`*`的version tag

`**`規則雖然也能匹配所有符號，但它的優先級高於`*`。

大多數patterns不含wildcard，所以gold和ld.lld迭代patterns而不是符號來改善性能。

### Versioned symbol產生方式

一個未定義符號獲得version的方式：

* 名字不包含`@`(沒有使用`.symver`)：某個shared object定義了default version符號
* 名字包含`@`：該符號須要被某個shared object定義，否則GNU ld會報錯；<https://reviews.llvm.org/D92260>之後ld.lld也會報錯

一個定義的符號獲得version的方式：

* 名字不包含`@`：被version script的一個named version tag的某個pattern匹配而獲得version
* 名字包含`@`
  + `-shared`：version `v1`須要被version script定義，否則GNU ld會報錯(`version node not found for symbol `)
  + `-no-pie`或`-pie`：GNU ld不需要version script即會生成version定義`v1`。這個行爲奇怪。

## 推薦用法

個人推薦：

定義default-version符號時不要用`.symver`。在version script的相應version node裏指定這個符號即可。
如果你確實想用`.symver`，使用`.symver foo, foo@@@v2`，在.o中只產生`foo@@v2`，不產生`foo`。

定義non-default符號時在原符號名後加後綴(`.symver foo_v1, foo@v1`)防止和`foo`衝突。在.o中會同時有`foo_v1`和`foo@v1`。目前沒有便捷方法去除(通常不想要的)`foo_v1`，一般在指定version script時注意把`foo_v1`設置爲local

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

未定義的versioned符號通常是鏈接時綁定的，object files不須要指定符號。如果確實要引用，推薦`.symver foo, foo@@@v1`，即使能`.symver foo, foo@v1`達到相同效果

多數時候，你希望未定義在鏈接時綁定到default version。通常沒有必要指定`.symver`。
如果確實要引用，`.symver foo_v1, foo@@@v1`或`.symver foo_v1, foo@v1`均可。

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

未定義符號被綁定到non-default version `foo@v1`：
```sh
% readelf -W --dyn-syms b.so | grep foo
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND foo@v1 (2)
```

如果省略`.symver`，則會被綁定到default version `foo@@v2`。

## rtld行爲

_Linux Standard Base Core Specification, Generic Part_ 描述了ld.so行爲。
kan在2005年給FreeBSD rtld添加了symbol versioning支持。

Dynamic table中的`DT_VERNEED`和`DT_VERNEEDNUM`標識了一個shared object/可執行檔需要的外部version定義，及該定義須由哪個shared object(`Vernaux::vna_name`)提供。
如果該Vernaux項(附屬於Verneed)沒有`VER_FLG_WEAK`標誌，且目標shared object中`DT_VERDEF`表存在但沒有定義需要的version，報錯。

通常minor releases不會bump soname。假設libB.so依賴1.3版本的libA(soname爲`libA.so.1`)，用了1.2版本不存在的API(函數)。假如使用PLT lazy binding，libB.so在安裝了1.2版本的libA的系統上似乎還能工作，直到1.3版本的函數的PLT被調用了爲止
若不使用symbol versioning，如果想要解決這個問題，就得在soname裏記錄minor version number(`libA.so.1.3`)
若使用symbol versioning，可以繼續使用`libA.so.1`。ld.so會在安裝了libA 1.2的系統上報錯，因爲libB.so的`DT_VERNEED`需要的1.3 version不存在

爲`foo`搜索定義時：

* 對於不含`DT_VERDEF`的object
  + 可以綁定到定義`foo`
* 對於含`DT_VERDEF`的object
  + 可以綁定到定義version `VER_NDX_GLOBAL`的`foo`
  + 可以綁定到定義任一default version的`foo`
  + 在relocation resolving階段(非dlvsym)可以綁定到定義non-default version index 2的`foo`

注意(未定義`foo`解析到定義version index 2的`foo@v1`)這種情況是ld.so允許而鏈接器不允許的<a name="reject-non-default">{reject-non-default}</a>。
rtld的行爲使得給.so庫添加version可以保持兼容性。
假如一個新版本想廢棄unversioned `bar`，可以去除`bar`而定義`bar@compat`。依賴該.so的庫中的未定義`bar`仍可以解析，但該庫無法重新用於鏈接其他程序。

爲`foo@v1`搜索定義時：

* 對於不含`DT_VERDEF`的object
  + 可以綁定到定義`foo`
* 對於含`DT_VERDEF`的object
  + 可以綁定到定義`foo@v1`或`foo@@v1`
  + 在relocation resolving階段(非dlvsym)可以綁定到定義version `VER_NDX_GLOBAL`的`foo`

假設`b.so`引用`malloc@GLIBC_2.2.5`，可執行檔因爲包含了一個malloc實現而定義`malloc`。
運行時`b.so`中的`malloc@GLIBC_2.2.5`會綁定到可執行檔。
address/memory/thread sanitizers利用了這個行爲：shared objects不需要鏈接interceptors，只有可執行檔需要鏈接interceptor。

## glibc中升級過的符號

注意，binutils 2.35之前的`nm -D`不顯示`@`和`@@`。

```sh
nm -D /lib/x86_64-linux-gnu/libc.so.6 | \
  awk '$2!="U" {i=index($3,"@"); if(i){v=substr($3,i); $3=substr($3,1,i-1); m[$3]=m[$3]" "v}} \
  END {for(f in m)if(m[f]~/@.+@/)print f, m[f]}'
```

在我的x86-64系統上的輸出：
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

一個符號發生ABI變更時(如變更參數或返回值的類型)必須要加新version，而API行爲變更時有時也會加，比如：

* `realpath@@GLIBC_2.3`: 之前的realpath在第二個參數爲NULL時返回EINVAL
* `memcpy@@GLIBC_2.14` [BZ12518](https://sourceware.org/bugzilla/show_bug.cgi?id=12518): 之前的memcpy copies forward。當年的Shockwave Flash有個memcpy downward的bug因爲memcpy採用了複雜的copy策略而觸發
* `quick_exit@@GLIBC_2.24` [BZ20198](https://sourceware.org/bugzilla/show_bug.cgi?id=20198): 之前的quick_exit會調用thread_local objects的destructors
* `glob64@@GLIBC_2.27`: 之前的glob不follow dangling symlinks

## 去除symbol versioning

因爲前文提到的[reject]：如果`in.so`引用`foo@v1`，而該符號只有unversioned `foo`定義。用`in.so`鏈接可執行檔時會因爲預設的`--no-allow-shlib-undefined`行爲報錯。ld.lld的錯誤信息：
```
ld.lld: error: undefined reference to foo@v1 [--no-allow-shlib-undefined]
```

如果因爲各種原因非得復用`in.so`，一種比較hack的解決方法是：
```sh
cp in.so out.so
r2 -wqc '/x feffff6f00000000 @ section..dynamic; w0 16 @ hit0_0' out.so
llvm-objcopy -R .gnu.version out.so
```

刪除`.gnu.version`後，鏈接器就會認爲`out.so`引用的是`foo`而非`foo@v1`。

llvm-objcopy在刪除sections時會把對應的區域清零。
這樣得到的`out.so`若用於運行時，glibc會報錯`unsupported version 0 of Verneed record`。
若須用於運行時，可以把dynamic table中的`DT_VER*`刪除。上面用了r2命令定位到`DT_VERNEED`(0x6ffffffe)並把它改寫爲`DT_NULL`(解析dynamic table時遇到`DT_NULL`停止)。`readelf -d`的輸出大致是：

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

## 評價

GCC/Clang支持asm specifier和`#pragma redefine_extname`重命名一個符號。比如聲明`int foo() asm("foo_v1");`再引用`foo`，.o中的符號會是`foo_v1`。

舉個例子，musl v.1.2.0最大的變化是32-bit架構的time64支持。musl採取了一種使用asm specifier的方案：

```c
// include/features.h
#define __REDIR(x,y) __typeof__(x) x __asm__(#y)

// API header include/sys/time.h
int utimes(cosnt char *, const struct timeval [2]);
__REDIR(utimes, __utimes_time64);

// Implementation src/linux/utimes.c
int utimes(const char *path, const struct timeval times[2]) { ... }

// Internal header compat/time32/time32.h
int __utimes_time32() __asm__("utimes");

// Compat implementation compat/time32/utimes_time32.c
int __utimes_time32(const char *path, const struct timeval32 times32[2]) { ... }
```

* .o中，time32定義仍爲`utimes`，提供ABI兼容舊程序；time64定義則爲`__utimes_time64`
* Public header用asm specifier重定向`utimes`到`__utimes_time64`
  + 缺點是倘若用戶自行聲明`utimes`而不include public header，會得到deprecated time32定義。這種自行聲明的方式是不推薦的
* 內部實現中“好看的”名字`utimes`表示time64定義；“難看的”名字`__utimes_time32`表示deprecated time32定義
  + 假如time32實現被其他函數調用，那麼用“難看的”名字能清晰地標識出來“此處和deprecated time32定義有關”

對於上述的例子，用symbol versioning來實作大概是這樣：

```c
// API header include/sys/time.h
int utimes(cosnt char *, const struct timeval [2]);

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

注意`.symver`不可用`@@@`。這個header被定義的translation unit使用，`@@@`會產生一個default version定義，而我們想要一個non-default version。
根據前文對Assembler行爲的討論，不如意的地方是：定義的translation unit中，`__utimes_time32`這個符號也存在。鏈接時注意用version script localize它。

那麼symbol versioning還有什麼意義呢？我細細琢磨，有如下優點：

* 在不阻礙運行時符號解析的情況下拒絕鏈接舊的符號[(reject-non-default)](#reject-non-default)
* 不需要標註declarations
* version定義可以延遲決定到鏈接時。version script提供靈活的pattern matching機制指定versions
* Scope reduction。然而，另一個類似`--dynamic-list`的機制可能會被發明用於localize符號
* 對編譯器認識的builtin functions，在GCC/Clang的實現裏重命名有一些語義上的問題(符號foo含有內建語義X)[2020-10-15-intra-call-and-libc-symbol-renaming](2020-10-15-intra-call-and-libc-symbol-renaming)
* [verneed-check]

對於第一條，asm specifier的方案用約定來避免意外鏈接(用戶應該include header)；而symbol versioning可以用ld強制。

設計缺點：

* `.symver foo, foo@v1`在`foo`被定義時的行爲[gas-copy]：保留符號`foo`(鏈接時有個多餘的符號)、binding/`st_other`保持同步(不方便設置不同的binding/visibility)
* Verdaux有點多餘。實踐中一個Verdef只有一個Verdaux
* Verneed/Vernaux的結構綁定了提供version定義的shared object的soname，ld.so要求"a versioned symbol is implemented in the same shared object in which it was found at link time"，這給在不同shared objects間移動定義的符號造成了不便。所幸glibc 2.30 [BZ24741](https://sourceware.org/PR24741)放寬了該要求，實質上忽略了`Vernaux::vna_name`

在此之前，glibc把`clock_*`函數從librt.so移動到libc.so用的方法是：

```c
// rt/clock-compat.c
__typeof(clock_getres) *clock_getres_ifunc(void) asm("clock_getres");
__typeof(clock_getres) *clock_getres_ifunc(void) { return &__clock_getres; }
```

libc.so中定義`__clock_getres`和`clock_getres`。librt.so中用一個名爲`clock_getres`的ifunc引導到libc.so中的`__clock_getres`。

## 相關鏈接

* [Combining Versions](http://www.airs.com/blog/archives/220)
* [Version Scripts](http://www.airs.com/blog/archives/300)
* <https://invisible-island.net/ncurses/ncurses-mapsyms.html>
