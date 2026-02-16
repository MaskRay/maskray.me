---
layout: post
title: "GCC 13.3.0 miscompiles LLVM"
author: MaskRay
tags: [assembler,gcc,llvm]
---

For years, I've been involved in updating LLVM's MC layer. A recent journey led me to [eliminate the `FK_PCRel_` fixup kinds](/blog/2025-05-26-llvm-integrated-assembler-improving-expressions-and-relocations):

    MCFixup: Remove FK_PCRel_

    The generic FK_Data_ fixup kinds handle both absolute and PC-relative
    fixups. ELFObjectWriter sets IsPCRel to true for `.long foo-.`, so the
    backend has to handle PC-relative FK_Data_.

    However, the existence of FK_PCRel_ encouraged backends to implement it
    as a separate fixup type, leading to redundant and error-prone code.

    Removing FK_PCRel_ simplifies the overall fixup mechanism.

<!-- more -->

As a prerequisite, I had to update several backends that relied on the now-deleted fixup kinds.
It was during this process that something unexpected happened.
Contributors [reported that when built by GCC 13.3.0](https://github.com/llvm/llvm-project/commit/c20379198c7fb66b9514d21ae1e07b0705e3e6fa), the LLVM integrated assembler had test failures.

To investigate, I downloaded and built GCC 13.3.0 locally:

```sh
../../configure --prefix=$HOME/opt/gcc-13.3.0 --disable-bootstrap --enable-languages=c,c++ --disable-libsanitizer --disable-multilib
make -j 30 && make -j 30 install
```

I then built a Release build (`-O3`) of LLVM. Sure enough, the failure was reproducible:

```
% /tmp/out/custom-gcc-13/bin/llc llvm/test/CodeGen/X86/2008-08-06-RewriterBug.ll -mtriple=i686 -o s -filetype=obj
Unknown immediate size
UNREACHABLE executed at /home/ray/llvm/llvm/lib/Target/X86/MCTargetDesc/X86BaseInfo.h:904!
PLEASE submit a bug report to https://github.com/llvm/llvm-project/issues/ and include the crash backtrace.
Stack dump:
0.      Program arguments: /tmp/out/custom-gcc-13/bin/llc llvm/test/CodeGen/X86/2008-08-06-RewriterBug.ll -mtriple=i686 -o s -filetype=obj
1.      Running pass 'Function Pass Manager' on module 'llvm/test/CodeGen/X86/2008-08-06-RewriterBug.ll'.
2.      Running pass 'X86 Assembly Printer' on function '@foo'
Stack dump without symbol names (ensure you have llvm-symbolizer in your PATH or set the environment var `LLVM_SYMBOLIZER_PATH` to point to it):
0  llc 0x0000000002f06bcb
fish: Job 1, '/tmp/out/custom-gcc-13/bin/llc …' terminated by signal SIGABRT (Abort)
```

Interestingly, a RelWithDebInfo build (`-O2 -g`) of LLVM did not reproduce the failure, suggesting either an undefined behavior, or an optimization-related issue within GCC 13.3.0.

## The Bisection trail

I built GCC at the `releases/gcc-13` branch, and the issue vanished. This strongly indicated that the problem lay somewhere between the `releases/gcc-13.3.0` tag and the `releases/gcc-13` branch.

The bisection led me to a specific commit, directing me to <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109934#c6>.

I developed a workaround at the code block with a typo "RemaningOps".
Although I had observed it before, I was hesitant to introduce a commit solely for a typo fix.
However, it became clear this was the perfect opportunity to address both the typo and implement a workaround for the GCC miscompilation.
This led to the landing of [this commit](https://github.com/llvm/llvm-project/commit/6d67794d164ebeedbd287816e1541964fb5d6c99), resolving the miscompilation.

Sam James from Gentoo mentioned that the miscompilation was introduced by a commit cherry-picked into GCC 13.3.0. GCC 13.2.0 and GCC 13.4.0 are good.
