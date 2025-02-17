layout: post
title: glibc and DT_GNU_HASH
author: MaskRay
tags: [glibc]
---

tl;dr "Easy Anti-Cheat"'s incompatibility with glibc 2.36 provides shared objects (`libc.so.6`, `ld-linux-x86_64.so.2`) is an instance of Hyrum's law.

* On 2022-08-02 glibc 2.36 was [released](https://sourceware.org/pipermail/libc-alpha/2022-August/141193.html).
* On the same day the x86-64 package was [moved to `[core]`](https://github.com/archlinux/svntogit-packages/commit/c49faa50a6b806cb67be5885b77d0304fffc7c7c) on Arch Linux.
* On 2022-08-03 Jelgnum reported that with the new glibc, "Easy Anti-Cheat" cannot load the anti-cheat module ([GLIBC update broke EAC for most games that use it](https://github.com/ValveSoftware/Proton/issues/6051)).
* Multiple Arch Linux game users confirmed the problem.
* Frogging101 bisected the problem to the glibc commit [Do not use --hash-style=both for building glibc shared objects](https://sourceware.org/git/?p=glibc.git;a=commit;h=e47de5cb2d4dbecb58f569ed241e8e95c568f03c).
* The problem led to heated discussions, some clickbait news, and claims such as "glibc breaks ABI" and "glibc does not prioritize compatibility with pre-existing applications".

I feel compelled to demystify the accident and wish that people can stop defamation to glibc.

<!-- more -->

## Root cause

Carlos O'Donell provided [a great summary](https://sourceware.org/pipermail/libc-alpha/2022-August/141304.html) in a reply to the libc-alpha thread "Should we make DT\_HASH dynamic section for glibc?" on 2022-08-08.

The [glibc commit](https://github.com/archlinux/svntogit-packages/commit/c49faa50a6b806cb67be5885b77d0304fffc7c7c) dropped a compiler driver option `-Wl,--hash-style=both` when linking glibc provided shared objects (e.g. `libc.so.6`, `libpthread.so.0`, `ld-linux-x86-64.so.2`).
Many Linux distributions have configured their GCC to pass `--hash-style=gnu` to the linker or configured GNU ld to default to `--hash-style=gnu`.
In the absence of `--hash-style=both`, the linker produces a `.gnu.hash` section and a `DT_GNU_HASH` tag and suppresses `.hash` and `DT_HASH`.
The glibc commit does not change how a user executable/shared object is linked.

I do not use the game software, so my reasoning about "Easy Anti-Cheat" is based on others' information.
Apparently "Easy Anti-Cheat" does something similar to a dynamic loader (rtld), likely that it does some symbol lookup using `DT_HASH`. There is no `DT_GNU_HASH` support.
When the software comes to a glibc `libc.so.6` or `ld-linux-x86-64.so.2` without `DT_HASH`, it reports an error.
A wild guess is that "Easy Anti-Cheat" tries to detect whether a function has been interposed (see [ELF interposition and -Bsymbolic](/blog/2021-05-16-elf-interposition-and-bsymbolic)): it needs to bypass the regular `dlsym`/`dlvsym` functions.

Note: "Easy Anti-Cheat"'s reliance on `DT_HASH` was noticed by Gentoo users back in 2022-04 (<https://github.com/anyc/steam-overlay/issues/309>).

## What is `DT_HASH`?

`DT_HASH` is a dynamic tag [specified](http://www.sco.com/developers/gabi/latest/ch5.dynamic.html#hash) by the System V Application Binary Interface (generic ABI).
For an output executable or shared object needing a dynamic symbol table (`.dynsym`), a linker produces a `.hash` section with type `SHT_HASH` holding a symbol hash table. A `DT_HASH` tag is produced to hold the address of `.hash`.

`DT_HASH` is used by a dynamic loader to perform symbol lookup (for dynamic relocations and `dlsym` family functions).
[ELF: symbol lookup via `DT_HASH`](https://flapenguin.me/elf-dt-hash) has a great description of the format.

In "Figure 5-10: Dynamic Array Tags, `d_tag`", the generic ABI says that `DT_HASH` is mandatory in an executable or shared object.
I will talk about this later.

## What is `DT_GNU_HASH`?

In 2006, glibc [commit 871b91589bf4f6dfe19d5987b0a05bd7cf936ecc](https://sourceware.org/git/?p=glibc.git;a=commit;h=871b91589bf4f6dfe19d5987b0a05bd7cf936ecc) added support for GNU hash table as a replacement for the generic ABI hash table.
(In the old days, GNU toolchain commit messages only said "what a commit did", not the motivation.)
When `--hash-style={gnu,both}` is in effect, for an output executable or shared object needing a dynamic symbol table (`.dynsym`), GNU ld produces a `.gnu.hash` section with type `SHT_GNU_HASH` holding a GNU hash table. A `DT_GNU_HASH` tag is produced to hold the address of `.gnu.hash`.

The 2006-06 thread [[PATCH] DT\_GNU\_HASH: ~ 50% dynamic linking improvement](https://sourceware.org/pipermail/libc-alpha/2006-June/020618.html) has some discussion.
A 2006-10 message [GNU\_HASH section format](https://sourceware.org/pipermail/binutils/2006-October/049450.html) describes the format.
Unfortunately, as of 2022-08, `DT_GNU_HASH` is not specified in a more official document.

For a curious reader who doesn't want to learn the history, just read [ELF: better symbol lookup via `DT_GNU_HASH`](https://flapenguin.me/elf-dt-gnu-hash).

Ali Bahrami's [The Cost Of ELF Symbol Hashing](http://www.linker-aliens.org/blogs/ali/entry/the_cost_of_elf_symbol/) has described the advantages of `DT_GNU_HASH` over `DT_HASH`:

> * An improved hash function is used, to better spread the hash keys and reduce hash chain length.
> * The dynamic symbol table is sorted into hash order, such that memory access tends to be adjacent and monotonically increasing, which can help cache behavior. (Note that the Solaris link-editor does a similar sort, although the specific details differ.)
> * The dynamic symbol table contains some symbols that are never looked up by via the hash table. These symbols are left out of the hash table, reducing its size and hash chain lengths.
> * Perhaps most significantly, the GNU hash section includes a Bloom filter. This filter is used prior to hash lookup to determine if the symbol is found in the object or not.

The bloom filter size is configurable. In [ld.lld's setting](https://reviews.llvm.org/D42204), the produced `DT_GNU_HASH` is almost always smaller than `DT_HASH`.
If something like Solaris direct bindings is leveraged which mostly eliminates unsuccessful symbol lookup, we can make the bloom filter size to 1 to remove the overhead.

Nowadays `DT_GNU_HASH` is pretty much universal among ELF operating systems.

* DragonFlyBSD <https://gitweb.dragonflybsd.org/dragonfly.git/commit/7629c6317998f850ebca23c296822ba08af09e5b> (2012-03)
* FreeBSD <https://cgit.freebsd.org/src/commit/?id=f62651920d526d8ef7a8ea66487e5bd1814a7a6f> (2012-04)
* musl <https://git.musl-libc.org/cgit/musl/commit/?id=2bd05a4fc26c297754f7ee5745a1c3b072a44b7d> (2012-08)
* Android bionic <https://android.googlesource.com/platform/bionic/+/ec18ce06f2d007be40ad6f043058f5a4c7236573> (2014-11)
* OpenBSD <https://github.com/openbsd/src/commit/ac51d06c6c4e6ed24fe575245f6450f3721e3842> (2018-11)
* NetBSD <https://github.com/NetBSD/src/commit/c01408307ce010883aac252f282645252f4c0a58> (2020-02)
* SerenityOS <https://github.com/SerenityOS/serenity/commit/b370ee3423d9570f097adcbddcd07fac4285da52> (2021-01)

## `DT_GNU_HASH` transition

`DT_GNU_HASH` is superior to `DT_HASH` in almost all aspects except the slight implementation complexity.
The nice thing is that the transition is mostly transparent. As long as ld and rtld support the format, we can use it.
(Well, I will soon talk about exceptions: programs may poke into the rtld/libc internal, reimplement symbol lookup but do not support `DT_GNU_HASH`, and therefore make the transition not smooth.)

GNU ld made a transition to a `--hash-style=both` default. The configure option `--enable-default-hash-style=gnu` can change the default.
Some Linux distributions carried local patches to make GCC pass `--hash-style=gnu` to ld, so that most pieces of software used the format.
E.g. [Fedora Core 6](https://fedoraproject.org/wiki/Releases/6/FC6ReleaseSummary) (released in 2006-10) made the switch. I saw a 2007 Gentoo post about using `--hash-style=gnu`.
I haven't made a thorough review but it appears that the majority of Linux distributions have switched to `--hash-style=gnu` in 201x.

In 2011, [install.texi (Configuration): Document --with-linker-hash-style.](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=79bec9233c4f17eebe157c21ebfbbdc5c733357d) added a configure option `--with-linker-hash-style=` which was then adopted by distributions.
If GCC is configured with `-–with-linker-hash-style=`, it passes the `-–hash-style= value` to ld; otherwise GCC doesn't pass `-–hash-style=` and the ld default is used.

Generally there are two categories of reasons that `--hash-style=gnu` cannot be used.

* ABI flaw
* Custom `dlsym` implementation which only supports `DT_HASH`

MIPS cannot use `DT_GNU_HASH` because it sorts `.dynsym` in a different way (for a technique called IRIX Quickstart, which AFAIK never has an implementation on other operating systems) which is incompatible with `DT_GNU_HASH`'s sorting requirement.
See [All about Global Offset Table](https://maskray.me/blog/2021-08-29-all-about-global-offset-table#dt_mips_local_gotno-and-dt_mips_symtabno-dt_mips_gotsym) for detail.

mumble used to rely on `DT_HASH`. `DT_GNU_HASH` support was added in <https://github.com/mumble-voip/mumble/commit/6f19d7ebfd7565843b3c56484af624afb5956c0f> and <https://github.com/mumble-voip/mumble/commit/9d3e53152a8df4059aeae9a00a3bbe438a4c56c0>.
libstrangle relied on `DT_HASH`: <https://gitlab.com/torkel104/libstrangle/-/issues/59>.

Some reliance is really about whether reimplementing `dlsym` is necessary.
If we assume that it is necessary (in some cases): they work if the software build system specifies `--hash-style=sysv` or `--hash-style=both` to override the distribution default `LDFLAGS` and make sure they don't need `DT_HASH` from their shared object dependencies.

What happens if glibc `libc.so.6` drops `DT_HASH`? The software almost assuredly use `libc.so.6` (on a Linux glibc system) and will likely break. This is an obvious instance of Hyrum's law to me:

> With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody.

glibc rtld continues supporting `DT_HASH` in user executables and shared objects but it decides to leave its own shared objects (e.g. `libc.so.6`, `libpthread.so.0`) to the GCC default.
I don't think the presence of `DT_HASH` is ever provided as a contract. This is a clear internal detail which isn't supposed to be relied upon by user programs.

## Discussions

### Is `DT_HASH` deprecated on Linux?

On every architecture except MIPS, `DT_HASH` has been de facto deprecated on many distributions for 10+ years.
Fedora's `--hash-style=gnu` transition in 2006 made `DT_HASH` executables and shared objects extremely rare.
libc.so.6 does contain `DT_HASH` for a long time, but it is just a rare exception.

Other distributions quickly caught up.
Debian patched GCC packages to use `--hash-style=both` for many ports in 2007.
Arch Linux had used `--hash-style=both` for a while and [switched to `--hash-style=gnu`](https://github.com/archlinux/svntogit-packages/commit/2b80ef30636bcf06afc26d7225028d41d71bb06e) in 2012-03.

### Could the accident have been detected earlier?

#### On the software side

"Easy Anti-Cheat" developers probably missed the fact that on many Linux distributions, most executables and shared objects do not have `.hash`/`DT_HASH` for a long time.

#### On the glibc side

Did glibc 2.36 need a release note about dropped `-Wl,--hash-style=both`?
Game users affected by the problem might argue that this was a high profile change and a deprecation or warning notice was needed.
I disagree.

I beg that you read Carlos's summary. `DT_HASH` is a protocol between a linker and a dynamic loader.
It is not intended to be consumed by a random non-standard ELF consumer.
In addition, 16 years have been sufficiently long for any non-standard ELF consumer to know that `DT_HASH` has been mostly eliminated from Linux distributions.
The glibc change removed one remnant `DT_HASH` use. It really was not as impactful as other changes in glibc 2.36.

#### On the user side

Gentoo users noticed the issue back in 2022-04 (<https://github.com/anyc/steam-overlay/issues/309>).
Sam James worked around "Easy Anti-Cheat"'s reliance on `DT_HASH` with [sys-libs/glibc: re-enable DT_HASH](https://gitweb.gentoo.org/repo/gentoo.git/commit/?id=8afecc68b8d689dbfdbff3b16ca50be66deb3cce).
This wasn't widely aware. Upstream glibc happened to subsequently made a different but with a similar behavior change (dropping `-Wl,--hash-style=both`), essentially dropping `DT_HASH` from glibc provided shared objects on many Linux distributions.

I empathesize with those who ran into the issue but with all due respect, I think this might have not been easily caught beforehand.
"Easy Anti-Cheat" is proprietary and IMO niche. It is probably popular among the gaming community but isn't that common taking account of the whole Linux glibc community.
If release testing did not catch the issue, that's it.

All in all, the issue was identified quickly. For Arch Linux (many Steam users use Arch Linux), Frederik Schwan pushed [re-add DT_HASH to glibc shared objects removed in 2.36](https://github.com/archlinux/svntogit-packages/commit/e1d69d80d07494e3c086ee2c5458594d5261d2e4).
We wish that Epic Games can fix the problem soon.

### Is omitting `DT_HASH` conforming to the generic ABI?

In general a processor supplement ABI or an operating system ABI can replace a generic ABI feature, and we should not read too much from the generic ABI wording.
When `DT_GNU_HASH` is shipped as a replacement, omitting the replaced feature `DT_HASH` is totally fine.
glibc's `ld-linux-x86_64.so.2` and `libc.so.6` have the OSABI value `ELFOSABI_GNU`. Nevertheless, it is worth discussing how `DT_GNU_HASH` fits the generic ABI and `ELFOSABI_NONE`.

### Is `DT_HASH` optional in the generic ABI?

If one reads much from the generic ABI wording, it says "mandatory", and therefore it is not optional.
Does this make sense?

Technically a dynamic loader does not need a hash table to perform symbol lookup. It can start at the dynamic symbol table beginning specified by `DT_SYMTAB`, and scan to the end.
Wait, in the absence of `DT_HASH` (`DT_GNU_HASH` is an extension, we want a way without an extension), there is no reliable way to get the number of dynamic symbol table entries.
I tend to think this is outside of the generic ABI's business to require something. An ELF object can freely use an extension to provide the information.
Specifying things in such a verbatim way is not ELF's spirit.
Michael Matz disagrees in a [reply](https://groups.google.com/g/generic-abi/c/th5919osPAQ/m/uGSA05KIAgAJ) to "Making DT_HASH optional?".

### Should `DT_GNU_HASH` upgrade `ELFOSABI_NONE` to `ELFOSABI_GNU`?

Ali Bahrami holds this opinion while Roland McGrath and I disagree.
Roland's argument is that `ELFOSABI_GNU` is for extensions like `STB_GNU_UNIQUE` and `STT_GNU_IFUNC`, not for extra non-standard `DT_*` tags.

`DT_GNU_HASH` predates `e_ident[EI_OSABI]`/`ELFOSABI_*` and belongs to a generic range (outside of `[DT_LOOS,DT_HIOS]` and `[DT_LOPROC,DT_HIPROC]`).
Cary Coutant proposed that we can retroactively [add `DT_GNU_HASH` to the generic ABI](https://groups.google.com/g/generic-abi/c/9L03yrxXPBc) and Ali Bahrami objected to the proposal.

Note: `SHT_GNU_HASH` belongs to a OS-specific range. If `DT_GNU_HASH` were accepted, we probably needed to find a new value in a generic range.

There is a related issue that Linux has used `ELFOSABI_NONE` for GNU specific things for many years. E.g. Just using GNU symbol versioning does not upgrade `ELFOSABI_NONE` to `ELFOSABI_GNU`.
Very few features use `ELFOSABI_GNU` as an indicator and later `ELFOSABI_LINUX` is defined as an alias for `ELFOSABI_GNU`.
The OSABI values can technically facilitate different systems running non-native objects.
In reality this interoperability isn't done very smoothly.

For Linux and many BSD systems, we are now on an interesting land:

* We use GNU and LLVM toolchains.
* Many features are provided for all ELF operating systems.

We do not have `ELFOSABI_GNUBASE` or `ELFOSABI_LLVM`. Forcing a OSABI value can be regarded as imposing (in some sense) unnecessary inconvenience.

If we really want to force an `e_ident[EI_OSABI]` value, what should we do? Cross compilation and build reproducibility is highly appreciated nowadays.
For a linker command line, using different `e_ident[EI_OSABI]` values on different systems is a bad practice.
Technically we can let the compiler driver pass `-m emulation` to ld and let ld set `e_ident[EI_OSABI]` according to the emulation.
As a linker maintainer, I think this is inconvenient and unnecessary, when the produced ELF object files are quite homogenous on many systems.
As a new OS developer, such distinction is unnecessary, too.

### `DT_SYMTABSZ` or `DT_SYMTAB_COUNT`

The second word in a `DT_HASH` hash table is `nchain`, which equals the number of dynamic symbol table entries.
People agree that a direct way obtaining the number will be great.
We can add `DT_SYMTABSZ` to the generic ABI.
In practice ELF consumers want to know the number of entries, not the size of the symbol table, so `DT_SYMTAB_COUNT` will be more convenient.

The argument favoring `DT_SYMTABSZ` is precedents such as `DT_PLTRELSZ, DT_RELASZ, DT_RELSZ`.

Here is the code to get the total number of symbols using `DT_GNU_HASH`.
```c
uint32_t gnu_hash_get_num_syms(uint32_t *hashtab) {
  uint32_t nbuckets = hashtab[0];
  uint32_t *buckets = hashtab + 4 + hashtab[2]*(sizeof(size_t)/4);
  uint32_t idx = 0;
  for (uint32_t i = nbuckets; i--; )
    if ((idx = buckets[i]) != 0)
      break;
  if (idx != 0) {
    uint32_t *chain = buckets + nbuckets - hashtab[1];
    while (chain[idx++] % 2 == 0);
  }
  return idx;
}
```

## Things for the operating systems using GNU and LLVM toolchains to sort out

Use Linux x86-64 as an example (for other processors, just check out the relavant psABI (processor supplement ABI)). We have these ABI documents:

* System V Application Binary Interface: <http://www.sco.com/developers/gabi/latest/contents.html>
* System V Application Binary Interface AMD64 Architecture Processor Supplement (With LP64 and ILP32 Programming Models): <https://gitlab.com/x86-psABIs/x86-64-ABI/>
* gABI supplement for program loading and dynamic linking on GNU <https://sourceware.org/gnu-gabi/>
* Linux Extensions to gABI: <https://github.com/hjl-tools/linux-abi/>
* Linux Standard Base: <https://refspecs.linuxfoundation.org/lsb.shtml>

gnu-gabi has documented `DT_GNU_HASH` since 2022-08-25. The `DT_GNU_HASH` description was from [a message posted two years ago](https://sourceware.org/pipermail/gnu-gabi/2020q3/000428.html).

The document shall also state that `DT_HASH` is optional.
