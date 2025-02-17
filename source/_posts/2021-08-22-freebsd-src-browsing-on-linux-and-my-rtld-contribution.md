layout: post
title: FreeBSD src browsing on Linux and my rtld contribution
author: MaskRay
tags: [elf,freebsd,rtld]
---

Before September 2020 FreeBSD could only be built on a FreeBSD host.
Alexander Richardson did a lot of work making this possible: <https://wiki.freebsd.org/BuildingOnNonFreeBSD>.

<!-- more -->

## Get prebuilt Clang and LLD

I use prebuilt Clang and LLD from Chromium: <https://chromium.googlesource.com/chromium/src/tools/clang/+/refs/heads/main/scripts/update.py>.
My output directory is `~/Stable`.

## Build

FreeBSD src enables `-Werror` by default. Our Clang is new and may have many new diagnostics.
We can use `-DWITHOUT_WERROR` to drop `-Werror`.

Now kick off our build.

```sh
mkdir -p obj/default
MAKEOBJDIRPREFIX=$PWD/obj/default ./tools/build/make.py --cross-bindir=~/Stable/bin -j 20 buildworld TARGET=amd64 TARGET_ARCH=amd64 -DWITHOUT_WERROR
```

`tools/build/make.py` forwards unknown options to bmake.

It may stop at some step due to various non-hermetic issues.

Search for `.OBJDIR` in `man bmake` to learn `MAKEOBJDIRPREFIX`.
bmake comes with built-in support for separate src/obj trees.
The recursive makefile build system used by FreeBSD does not handle relative `MAKEOBJDIRPREFIX` paths well, so just use an absolute path.

## Build compile_commands.json

I tried [Bear](https://github.com/rizsotto/Bear) but it could not handle such a complex build system.
The process got stuck at some step, so I just gave up on it.

### bmake meta mode

I learned a bit about bmake (default `make` on FreeBSD) and noticed a nice feature: meta mode.
In meta mode bmake records build commands into `.meta` files.
For the next build, bmake will consult `.meta` files to evaluate whether the target has become out-of-date.
This is more robust than just comparing file modification times.
_Build Systems à la Carte_ says such a build system is _self-tracking_.

For FreeBSD src, we can enable meta mode with `-DWITH_META_MODE`.
After `buildworld`, we can parse these `.meta` files under objdir and build `compile_commands.json`.

```sh
MAKEOBJDIRPREFIX=$PWD/obj/meta ./tools/build/make.py --cross-bindir=~/Stable/bin -j 20 buildworld TARGET=amd64 TARGET_ARCH=amd64 -DWITH_META_MODE -DNO_FILEMON -DWITHOUT_WERROR
(
cd obj/meta
ruby -r find -r shellwords -r stringio -e '
  cdb = StringIO.new
  cdb << "["
  Find.find "home/ray/Dev/freebsd/amd64.amd64" do |path|
    next unless FileTest.file?(path) && path.end_with?(".meta")
    args = nil
    File.readlines(path).each do |line|
      line = line.chomp
      if line =~ /^CMD.*\/clang/; args=Shellwords::split(line[4..])
      elsif line =~ /^OODATE /; file=line[7..]
        if file =~ /\.(c|cc|cpp)$/ && args
          cdb << "," if cdb.pos > 1
          cdb << %{{"directory":#{Dir.pwd.dump}, "arguments":#{args}, "file":#{file.dump}}\n}
        end
      end
    end
  end
  cdb << "]"
  print cdb.string' > compile_commands.json
)
```

Some notes about the other `-D` variables.

bmake has code dealing with filemon, which is a FreeBSD driver.
On Linux we need to disable it with `-DNO_FILEMON`.

As of 2021-08, building on Linux still has some issues.
I mostly read `libexec/rtld-elf` and the build process can proceed beyond `libexec/rtld-elf`, so I am satisfied.

### ccls

With `compile_commands.json`, my [ccls](https://github.com/MaskRay/ccls) can index the repository.

Here is a screenshot browsing `libexec/rtld-elf` code in Emacs with (lsp-mode + emacs-ccls).

![](/static/2021-08-22-freebsd-src-browing-on-linux-and-my-rtld-contribution/emacs.webp)

```elisp
(setq ccls-sem-highlight-method 'font-lock)
(add-hook 'lsp-after-open-hook #'ccls-code-lens-mode)
(ccls-use-default-rainbow-sem-highlight)
```

## Contribute to libexec/rtld-elf

I stumbled upon FreeBSD `libexec/rtld-elf` in 2019 to sort out how ld.lld should set the `p_memsz` field of `PT_GNU_RELRO`.
I noticed an issue but did not get a chance to create a patch. Scroll down for details.

When working on some TLS issues in ld.lld, I noticed that rtld did not handle `p_vaddr % p_align != 0` correctly.
(Note: fixed for i386 and amd64.)

In 2020 I noticed a symbol resolution issue related to `STB_WEAK`, but did not follow up with the patch.
(Note: introduced the environment variable `LD_DYNAMIC_WEAK=0` to match ELF spec (glibc/musl behavior).)

Now that I have a proper setup, I can work on the aforementioned problems in a virtual machine running FreeBSD 12.2.
`qemu-system-x86_64 -enable-kvm -m 16384 -smp 16 -drive file=~/Images/freebsd.qcow2,if=virtio -net nic,model=virtio -net user,hostfwd=tcp::2223-:22`

```sh
% cat /etc/src.conf
WITHOUT_CLANG=yes
WITHOUT_LLD=yes
WITHOUT_LLDB=yes
WITHOUT_CLEAN=yes
```

```sh
# rsync changes to the src repository
MAKEOBJDIRPREFIX=$PWD/obj/default make -j 12 buildworld

# To build just libexec/rtld-elf
MAKEOBJDIRPREFIX=$PWD/obj/default make -j 12 buildenv
export CPUTYPE=  # zsh hack
make -C libexec/rtld-elf
```

(My experience with `SUBDIR_OVERRIDE=libexec/rtld-elf` is bad.)

The versions of rtld and libc should match if they are of different major versions.
Simple programs may work even if you don't use a libc of the matching version.

```sh
mkdir -p /tmp/opt/lib
echo 'GROUP ( libc/libc.so.7 libc_nonshared/libc_nonshared.a libssp_nonshared/libssp_nonshared.a )' > /tmp/opt/lib/libc.so

objdir=$HOME/freebsd/obj/default/usr/home/ray/freebsd/amd64.amd64

# Build a.c with fresh rtld and libc.
clang a.c -Wl,--dynamic-linker=$objdir/libexec/rtld-elf/ld-elf.so.1.full -L/tmp/opt/lib -L$objdir/lib -Wl,-rpath=$objdir/lib/libc -Wl,-t

# Debug ld.so.
cgdb --args $objdir/libexec/rtld-elf/ld-elf.so.1.full ./a
```

Thanks to kib who reviewed these patches and lwhsu who added me to this contributor list: <https://docs.freebsd.org/en/articles/contributors/#contrib-additional>.

### p_memsz of PT_GNU_RELRO

An ELF component usually needs a `PT_LOAD` program header with the permission bits `PF_R|PF_W`.
Some sections are only needed to be writable at relocation processing time and can be made read-only during regular program execution.
glibc invented `PT_GNU_RELRO` which has been ported to FreeBSD/NetBSD/OpenBSD.

While linkers ensure that there is an alignment boundary of max-page-size bytes between two `PT_LOAD` program headers, the alignment boundary following `PT_GNU_RELRO` is just common-page-size bytes.
GNU ld, gold, and ld.lld ensure that `p_vaddr+p_memsz` is a multiple of common-page-size.

glibc and musl do something like

```c
size_t start = roundDown(p_vaddr, PAGE_SIZE);
size_t size = roundDown(p_vaddr+p_memsz, PAGE_SIZE) - roundDown(p_vaddr, PAGE_SIZE);
mprotect(laddr(start), size, PROT_READ);
```

FreeBSD rtld did something like
```c
size_t start = roundDown(p_vaddr, PAGE_SIZE);
// roundUp instead of roundDown
size_t size = roundUp(p_vaddr+p_memsz, PAGE_SIZE) - roundDown(p_vaddr, PAGE_SIZE);
mprotect(laddr(start), size, PROT_READ);
```

If `PAGE_SIZE` (the system page size) is larger than the link-time common-page-size, mprotect may incorrectly map some non-RELRO pages read-only.

<https://reviews.freebsd.org/D31498> fixed the bug.

### STB_WEAK in symbol lookup

The first version of the ELF specification <http://www.sco.com/developers/gabi/1998-04-29/ch5.dynamic.html> says:

> When resolving symbolic references, the dynamic linker examines the symbol tables with a breadth-first search. That is, it first looks at the symbol table of the executable program itself, then at the symbol tables of the DT_NEEDED entries (in order), and then at the second level DT_NEEDED entries, and so on...

(This paragraph has not been updated in the latest snapshot.)

The common(?) interpretation is that STB_WEAK/STB_GLOBAL have no differences for symbol lookup.

I asked <https://groups.google.com/g/generic-abi/c/YdmpBmukW0g> for clarification and archaeology.

glibc and musl use this symbol lookup behavior: resolve to the first found symbol definition, regardless of `STB_WEAK`/`STB_GLOBAL`.

FreeBSD/NetBSD/OpenBSD use this non-conforming behavior: when a weak symbol definition is found, remember the definition and keep searching in the remaining shared objects for a non-weak definition.
If found, the non-weak definition is preferred, otherwise the remembered weak definition is returned.

<https://reviews.freebsd.org/D26352> implemented the Linux behavior under the environment variable `LD_DYNAMIC_WEAK=1`.

### p_vaddr % p_align != 0 for PT_TLS

It is very complex.
See [All about thread-local storage](/blog/2021-02-14-all-about-thread-local-storage).

<https://reviews.freebsd.org/D31538> fixed the i386/amd64 ports.

## Epilogue

### bmake

bmake supports a few features which make a make based build system less daunting:

* built-in support for separte src/obj trees
* meta mode (self-tracking build system)
* logical AND operators `&&` and `||` are supported in `.if` and `.elif` conditional structures
* variable modifiers

These are major pain points in GNU make.

GNU make needs something like <http://make.mad-scientist.net/papers/multi-architecture-builds/> to support separate src/obj trees.

Most GNU make based build systems cannot rebuild the target when the commands change.
The Linux kernel uses `.cmd` files to solve the problem.

In glibc, the following pattern is quite common to work around the lack of logical AND operators.
```make
ifeq ($(have-foo)$(have-bar),yesyes)
endif
```

Some variable modifiers can be difficult to remember, but a good use of them drops the need to spawn various shell utilities.

### rtld

FreeBSD rtld supports many GNU extensions:

* GNU indirect functions (`STT_GNU_IFUNC`, `R_*_IRELATIVE`)
* GNU symbol versioning

I came from a musl background. In many places I think FreeBSD's is overly complex, but is still relatively clean.

It has implemented some features which musl does not support:

* lazy binding PLT
* dlclose which actually unloads a DSO

In comparison, many stuff are quite messy in glibc rtld. It can learn a lot from musl and FreeBSD rtld.
