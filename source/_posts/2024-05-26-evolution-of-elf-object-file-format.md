---
layout: post
title: Evolution of the ELF object file format
author: MaskRay
tags: [elf,linux,unix]
---

Updated in 2025-09.

The ELF object file format is adopted by many UNIX-like operating systems.
While I've [previously delved into](/blog/2024-01-14-exploring-object-file-formats) the control structures of ELF and its predecessors, tracing the historical evolution of ELF and its relationship with the System V ABI can be interesting in itself.

The format consists of the generic specification, processor-specific specifications, and OS-specific specifications.
Three key documents often surface when searching for the generic specification:

* _Tool Interface Standard (TIS) Portable Formats Specification, version 1.2_ on <https://refspecs.linuxfoundation.org/>
* [_System V Application Binary Interface - DRAFT - 10 June 2013_](https://www.sco.com/developers/gabi/latest/contents.html) on www.sco.com
* _Oracle Solaris Linkers and Libraries Guide_

The TIS specification breaks ELF into the generic specification, a processor-specific specification (x86), and an OS-specific specification (System V Release 4).
However, it has not been updated since 1995. The Solaris guide,  though well-written, includes Solaris-specific extensions not applicable to Linux and *BSD.
This leaves us primarily with the System V ABI hosted on www.sco.com, which dedicates Chapters 4 and 5 to the ELF format.

Let's trace the ELF history to understand its relationship with the System V ABI.

<!-- more -->

## History

[Unix System Laboratories (USL)](https://en.wikipedia.org/wiki/Unix_System_Laboratories) created ELF for their System V Release 4 in late 1980s.
USL also maintained the System V Application Binary Interface, of which ELF was a core component.
The dynamic shared library system was contributed by Sun Microsystems from their [SunOS](https://en.wikipedia.org/wiki/SunOS) 4.x (in 1988, SunOS 4.0 got an extended a.out format with dynamic shared library support).

USL intended ELF to be an open standard and published documents about the format, e.g.

* In Proceedings of the Summer 1990 USENIX Conference, _ELF: An Object File to Mitigate Mischievous Misoneism_ by James Q. Arnold
* _UNIX System V Release 4 Programmer's Guide: ANSI C and Programming Support Tools_ (ISBN 0-13-933706-7) published in 1990
* _System V Application Binary Interface (Standards)_ (ISBN 0-13-104670-5) published in 1993

In 1993, the Tool Interface Standard (TIS) Committee, a consortium of industry leaders, adopted ELF and developed the "Tool Interface Standard (TIS) Portable Formats Specification".
Version 1.2 was released in May 1995.

ELF has been very influential.
In the 1990s, many Unix and Unix-like operating systems, including Solaris, IRIX, HP-UX, Linux, and FreeBSD, switched to ELF.
The 86open Project's FAQ specified:

> Q18: How can you get a single binary to work identically across all these diverse systems?
>
> Most Unix-on-Intel binary packages are already largely similar. Almost all such operating systems use the "ELF" binary 'packaging'; the various operating systems have small but significant differences, though, that make each system's ELF binary unusable on others'.

### The evolving stewardship of the System V ABI

The Tool Interface Standard (TIS) Committee essentially dissolved after 1995.
The stewardship of the System V ABI, and consequently the generic ELF specification, has followed a complex path mirroring the transfer of Unix software assets.

Between 1993 and 2011, Unix assets underwent a few transfers.

* In 1993, Novell [acquired Unix assets](https://en.wikipedia.org/wiki/Unix_System_Laboratories#Acquisition_by_Novell) including all copyrights, trademarks, and licensing contracts.
* In September 1995, Novell sold the "develop and sell licenses to Unix binaries" plus "handle source licencees" business to The Santa Cruz Operation (sometimes referred to as "old SCO"). Novell still owned the copyrights ([SCO vs Novell](https://en.wikipedia.org/wiki/SCO_Group,_Inc._v._Novell,_Inc.) verdict).
* In 2001, The Santa Cruz Operation sold its Unix software asserts to Caldera Systems (later renamed The SCO Group, Inc; sometimes referred to as "new SCO" or "SCOX").
* In 2011, The SCO Group's Unix software assets were sold off to UnXis (later renamed Xinuos).

**The task of maintaining and updating the generic ABI fell to these successive owners of Unix software assets**.
The Santa Cruz Operation, and later The SCO Group and Xinuos, managed updates and extensions to the ABI, including the ELF specification.

In this [binutils commit](https://sourceware.org/cgit/binutils-gdb/commit/?id=723b0f0d39ebe18c9f28e238c9ecc27931faffa7) in November 2000, it was said that `e_machine` values should eventually ask `registry@sco.com` for blessing (now `registry@xinuos.com`).

Dave Prosser had [maintained](http://www.groklaw.net/article.php?story=20040130235310123) the System V ABI at USL, then The Santa Cruz Operation, and then The SCO Group.
The last maintainer at The SCO Group and UnXis/Xinuous was John Wolfe, who oversaw updates until his [departure from Xinuos](https://groups.google.com/g/generic-abi/c/IakWYdGABjQ) in 2015.
**The generic ABI (including the ELF specification) then became unmaintained**.

The final functional update on <https://www.sco.com/developers/gabi/latest/contents.html> was made in June 2013 [for `SHF_COMPRESSED`](https://groups.google.com/g/generic-abi/c/9CUHDfWYeu4).
Since then, the specification has remained frozen.

### "All rights reserved"?

The copyright notices on the SCO website's documentation for the System V ABI seem potentially misleading.

The footnotes of <https://www.sco.com/developers/gabi/1998-04-29/contents.html> pages today (and in 2003 per web.archive.org) specify:

> © 1997, 1998, 1998 The Santa Cruz Operation, Inc. All rights reserved.

The footnotes of <https://www.sco.com/developers/gabi/latest/contents.html> pages specify:

> © 1997, 1998, 1999, 2000, 2001 The Santa Cruz Operation, Inc. All rights reserved. © 2002 Caldera International. All rights reserved. © 2003-2010 The SCO Group. All rights reserved. © 2011-2015 Xinuos Inc. All rights reserved.

The repeated phrase "All rights reserved" could be interpreted as implying exclusive ownership over the ELF format itself.
This is inaccurate, as ELF is an open standard developed through the collaboration of many organizations and individuals.
The Santa Cruz Operation's role in the evolution of the System V ABI seems to have been more of an editor than an innovator.
After The Santa Cruz Operation sold its Unix assets in 2001, the specification has largely stayed unchanged with occasional constant updates.

The earliest available snapshot on the Wayback Machine dates back to 2003, a time when The SCO Group had assumed ownership and initiated a lawsuit against IBM, alleging that the success of Linux was due to the misappropriation of SCO's technology.
Regrettably, earlier snapshots are unavailable to provide a more complete historical context.

_Tool Interface Standard (TIS) Portable Formats Specification, version 1.2_ effectively [put the specification in the public domain](http://www.groklaw.net/article.php?story=20040722135616439):

> The TIS Committee grants you a non-exclusive, worldwide, royalty-free license to use the information disclosed in this Specification to make your software TIS-compliant; no other license, express or implied, is granted or intended hereby.

Further reading:

* [The SCO lawsuit, 20 years later](https://lwn.net/Articles/924577/) 
* [A Tall Tale About ELF - by Frank Sorenson, Dr Stupid and PJ](http://www.groklaw.net/article.php?story=20040722135616439)
* [SCO shows more code](https://lwn.net/Articles/87556/)
* [The SCO lawsuit, 20 years later](https://lwn.net/Articles/924577/)
* [SCO's Summary Judgment Hearing Binder](https://www.sco.com/company/legal/update/website2.3.pdf)

### The generic-abi Google Group

A neutral [Google Group](https://groups.google.com/g/generic-abi) not affliated with The SCO Group/Xinuous exists for discussing the generic ABI. Hongjiu Lu might be the owner.
The group served as a platform for OS and toolchain vendors to collaborate. In recent years, participation has dwindled to primarily representatives from Oracle Solaris (just Ali Bahrami) and the GNU toolchain.

The reduced activity might not seem critical, as **significant non-OS-specific changes to the ELF format are infrequent**.

## Evolution of the generic ABI

<https://www.sco.com/developers/gabi/latest/revision.html> outlines the evolution of the ELF from 1998 to 2013.
Important features were all available as of April 2001.

* Symbol visibility
* [Section groups](/blog/2021-07-25-comdat-and-section-group)
* `EI_OSABI` and `EI_ABIVERSION`
* [`SHF_MERGE` and `SHF_STRINGS`](/blog/2021-12-19-why-isnt-ld.lld-faster#shf_merge-duplicate-elimination)
* [`SHF_LINK_ORDER`](/blog/2021-01-31-metadata-sections-comdat-and-shf-link-order)

However, **discussions regarding these specific features seem unavailable**. Please let me know if you have any information on them.

There were merely constant updates from April 2001 to June 2013.
`SHF_COMPRESSED` was added in June 2013.

The generic-abi Google Group **reached consensus on proposals** that haven't been reflected on the www.sco.com website:

* 2018: [RELR relative relocation format](/blog/2021-10-31-relative-relocations-and-relr#relr-relative-relocation-format)
* 2022: [`ELFCOMPRESS_ZSTD`](/blog/2022-09-09-zstd-compressed-debug-sections)

### A future in flux

In April 2020, Cary Coutant reached a [preliminary agreement](https://groups.google.com/g/generic-abi/c/9OO5vhxb00Y) with Xinuos, but **the future remains uncertain**.
While some constants (e.g., `e_machine` and `EI_OSABI` values, `ELFCOMPRESS_ZSTD`) have been defined, no functional updates to the ABI have materialized.

The absence of a centralized, up-to-date repository for the specification complicated matters.

<del>
While some clarifications and consensus have been reached within the generic-abi group, accessing the latest, definitive text remained a challenge.

A potential solution could be to **decouple the ELF specification from the broader System V ABI**, as was done in the past with the TIS specification.
This would create a dedicated and accessible reference for ELF, independent of the broader System V specificities that are of less general interest.

Despite this uncertainty, innovation within the ELF ecosystem should continue.
Efforts like my own to replace ELF control structures to reduce object file sizes (e.g., [compact relocations](/blog/2024-03-31-a-compact-section-header-table-for-elf).) can still move forward.
In practice, achieving consensus among major toolchain vendors (GNU and LLVM) may be sufficient, even without formal approval from the generic ABI.
While aligning with Solaris would be ideal and I will try doing so, this might not always be feasible due to varying priorities.

FreeBSD, which Xinuos's OpenServer is based on, utilizes the LLVM toolchain. Xinuos might indirectly benefit from my heavy involvement into the LLVM toolchain.
</del>

In August 2025, Cary Coutant published <https://gabi.xinuos.com/> (ELF Object File Format) and <https://github.com/xinuos/gabi>.

## System V ABI Processor Supplement (psABI)

Processor-specific details for the System V ABI are found in the psABI documents.
Actively maintained psABIs exist for various architectures including [AArch32](/blog/2023-04-23-linker-notes-on-aarch32), [AArch64](/blog/2023-03-05-linker-notes-on-aarch64), LoongArch, [PPC64](/blog/2023-02-26-linker-notes-on-power-isa), RISC-V, [s390x](/blog/2024-02-11-toolchain-notes-on-z-architecture), [i386, and x86-64](/blog/2023-02-19-linker-notes-on-x86). (These links refer to my notes.)

Many architectures have older or unavailable psABIs. For instance:

* ppc32: _Power Architecture® 32-bit Application Binary Interface Supplement 1.0 - Linux & Embedded_ was published in 2011. [My notes](/blog/2023-02-26-linker-notes-on-power-isa)
* MIPS: The most recent o32 ABI dates back to February 1996, and the n64 ABI is unavailable. The n32 ABI accompanies the discontinued compiler MIPSpro: _MIPSpro N32 ABI Handbook_. [My notes](/blog/2023-09-04-toolchain-notes-on-mips)

Noteworthy details

* Architectures like Motorola 6800, which have 16-bit address spaces, use the ELFCLASS32 format.
* Many architectures have never been ported to any System V derivative OS, but their psABI documents still use the "System V" name.
* Some behaviors are not formally documented and can only be found in the binutils project's source code.

## Operating System Specific ABIs

In the System V ABI, Operating System Specific ABIs (OSABI) are extensions that provide operating system-specific details to supplement the generic ABI.

For example, _Oracle Solaris Linkers and Libraries Guide_ defines the OSABI for Solaris.

The term OSABI is vague and might not be one single document. For Linux, we need the following two documents:

* [gABI supplement for program loading and dynamic linking on GNU](https://sourceware.org/gnu-gabi/)
* <https://gitlab.com/x86-psABIs/Linux-ABI/>

The Linux Standard Base (LSB) is a related document that aims to standardize the Linux system interface.

Until the recent introduction of _gABI supplement for program loading and dynamic linking on GNU_, `SHT_GNU_HASH`, while widely adopted, was absent from any official documentation.

Interestingly, many Linux ABI extensions are generic enough to be adopted by other operating systems like FreeBSD, suggesting that a dedicated FreeBSD OSABI document may not be necessary.
