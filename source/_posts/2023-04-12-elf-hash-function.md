---
layout: post
title: ELF hash function may overflow
author: MaskRay
tags: [linker]
---

This article describes an interesting overflow bug in the ELF hash function. 

The System V Application Binary Interface (generic ABI) specifies the ELF object file format.
When producing an executable or shared object file needing a dynamic symbol table (`.dynsym`), a linker generates a `.hash` section with type `SHT_HASH` to hold a [symbol hash table](http://www.sco.com/developers/gabi/latest/ch5.dynamic.html#hash).
A `DT_HASH` tag is produced to hold the address of `.hash`.

The hash table is used by a dynamic loader to perform symbol lookup (for dynamic relocations and `dlsym` family functions).
A detailed description of the format can be found in [ELF: symbol lookup via `DT_HASH`](https://flapenguin.me/elf-dt-hash).

<!-- more -->

## Other use cases

In a Solaris Version Definition Section, `vd_hash` holds a value generated using the ELF hash function.
The GNU symbol versioning scheme inherits this field from Solaris.
Dynamic loaders use this field to accelerate symbol lookup of versioned symbols.

Given such a small number of version definitions, even if a dynamic loader implementation has a bug, the overflow bug described below is very unlikely to cause any problems.

## Overflow bug

The generic ABI gives the following code fragment in "Figure 5-13: Hashing Function".
```c
unsigned long
elf_hash(const unsigned char *name)
{
	unsigned long	h = 0, g;
	while (*name)
	{
		h = (h << 4) + *name++;
		if (g = h & 0xf0000000)
			h ^= g >> 24;
		h &= ~g;
	}
	return h;
}
```

_UNIX System V Release 4 Programmer's Guide: ANSI C and Programming Support Tools_ (ISBN 0-13-933706-7) published in 1990 contains this function as well.

The function is supposed to return a value no larger than 0x0fffffff.
`h &= ~g` clears the 4 highest bits in a 32-bit integer.
(This isn't a good hash function: the high bits are unnecessarily discarded.)

Unfortunately, there is a bug.
When `unsigned long` consists of more than 32 bits, the return value may be larger than `UINT32_MAX`.
If `h` is in the range `[0x0fffff01,0x0fffffff]` in the previous iteration, shifting it by 4 and adding `*name` may make `h` larger than `UINT32_MAX`.

For instance, `elf_hash((const unsigned char *)"\xff\x0f\x0f\x0f\x0f\x0f\x12")` returns 0x100000002, which is clearly unintended, as the function should behave the same way regardless of whether `long` represents a 32-bit integer or a 64-bit integer.

It is possible to use 7-bit ASCII characters to trigger the issue. For instance, 

* `elf_hash((const unsigned char *)"iiiiii\na")) == 100000001`
* `elf_hash((const unsigned char *)"ZZZZZX+a")) == 100000011`
* `elf_hash((const unsigned char *)"ZZZZZW9p")) == 100000000`

Most ELF operating systems have switched from `DT_HASH` to `DT_GNU_HASH` for many years and prefer `DT_GNU_HASH` for symbol search.
We can build a shared object with `ld -shared --hash-style=sysv` and check whether a dynamic symbol named "ZZZZZW9p" can be bound by relocation resolver/`dlsym`.

## A bug in FreeBSD rtld-elf

If we compile the following C source file and link it with `-shared -fuse-ld=lld -Wl,--hash-style=sysv`, we get 3 dynamic symbols and a SysV hash table with `nbuckets=3`.
As of April 2023, `dlsym(dl, "ZZZZZW9p")` returns NULL on FreeBSD.

```c
void ZZZZZW9p(void) {}
void fn(void) {}
```

If we link the shared object with `--hash-style=gnu` or `--hash-style=both`, rtld-elf will use the GNU hash table (`DT_GNU_HASH`) and `dlsym(dl, "ZZZZZW9p")` will return the correct value.

This was just fixed by [rtld: fix SysV hash function overflow](https://reviews.freebsd.org/D39517), prompted by this article.
I am so thrilled - my article led to a bug fix within a few hours of my posting it.
I did check the FreeBSD source before publishing this article and would have wanted to fix the issue for my own pleasure, but having a FreeBSD developer fix it made me even happier.:)

## A bug in OpenBSD rtld-elf

OpenBSD `libexec/ld.so/resolve.c` inherited the bug from the example code.
The bug was fixed by [shortly after this article was published](https://github.com/openbsd/src/commit/d82bde86a0be85e03ff574f75586adc43dc19ece).

## Project survey

glibc [fixed the overflow issue](https://sourceware.org/git/?p=glibc.git;a=commit;h=6e33fad374814f1a4bf80aa37d4ded9c9096edab) while optimizing the function in April 1995.
The two XOR operations were [optimized to one](https://sourceware.org/git/?p=glibc.git;a=commit;h=f039c043071f2f55943d052fa7d4ad5f1a67db09) in Dec 2011.

binutils-gdb [fixed the overflow issue](https://sourceware.org/git/?p=binutils-gdb.git;a=commit;h=32dfa85d9015feea4a06d423fe58f6eaf841456e) in May 2003.
The function has a lovely comment "... Do not change this function; you will cause invalid hash tables to be generated."

musl has had the elegant and efficient implementation since June 2011 (initial check-in of the dynamic linker). It is worth noting that `uint_fast32_t` is used, so that an architecture can optimize the implementation if the architecture has slow 32-bit integer arithmetic operations.
```c
static uint32_t sysv_hash(const char *s0)
{
	const unsigned char *s = (void *)s0;
	uint_fast32_t h = 0;
	while (*s) {
		h = 16*h + *s++;
		h ^= h>>24 & 0xf0;
	}
	return h & 0xfffffff;
}
```

Nathan Sidwell [raised the issue for llvm-project](https://reviews.llvm.org/D147890) and pointed out a bug about using `char` instead of `unsigned char` on 2023-04-09.

I asked [What if the result of elf_hash is larger than UINT32_MAX?](https://groups.google.com/g/generic-abi/c/8J_jtjsonrE/m/dd3V9JAIBgAJ) on 2023-04-11.
