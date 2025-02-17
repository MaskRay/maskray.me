layout: post
title: RISC-V TLSDESC works!
author: MaskRay
tags: [linker]
---

Updated in 2024-05.

Back in 2019, I studied a bit about RISC-V and filed [Support Thread-Local Storage Descriptors (TLSDESC)](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/issues/94).
Last year, Tatsuyuki Ishi [added a specification](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/pull/373) for TLSDESC.

## LLVM

On the LLVM side, the RISC-V TLSDESC work has been completed.

The the most important patch is [[RISCV] Support Global Dynamic TLSDESC in the RISC-V backend](https://github.com/llvm/llvm-project/pull/66915) by Paul Kirth.
The [linker patch](https://github.com/llvm/llvm-project/pull/79099) by me is also significant.
Furthermore, Clang requires a [`-mtls-dialect=` patch](https://github.com/llvm/llvm-project/pull/79256).

These patches are expected to be included in the upcoming LLVM 18.1 release.
To obtain TLSDESC code sequences, compile your program with `clang --target=riscv64-linux -fpic -mtls-dialect=desc`.

## GCC

[RISC-V: Implement TLS Descriptors.](https://gcc.gnu.org/cgit/gcc/commit/?id=97069657c4e40b209c7b774e12faaca13812a86c) landed in April 2024.

## binutils

[RISC-V: Initial ld.bfd support for TLSDESC.](https://sourceware.org/cgit/binutils-gdb/commit/?id=159afbb7617d2c8a9cd3f3350374711f37f60442) landed in February 2024.

## glibc

Latest patch: <https://inbox.sourceware.org/libc-alpha/20230914084033.222120-1-ishitatsuyuki@gmail.com/>

## musl

musl [added support](https://git.musl-libc.org/cgit/musl/commit/?id=407aea628af8c81d9e3f5a068568f2217db71bba) in February 2024.

## Bionic

No patch yet.

## Testing

The LLVM patches need testing. Unfortunately, I didn't have a RISC-V image at hand, so I used qemu-user.

Patch musl per [Re: Draft riscv64 TLSDESC implementation](https://www.openwall.com/lists/musl/2024/01/22/1)
```diff
diff --git c/arch/riscv64/reloc.h w/arch/riscv64/reloc.h
index 1ca13811..7c7c0611 100644
--- c/arch/riscv64/reloc.h
+++ w/arch/riscv64/reloc.h
@@ -17,6 +17,7 @@
 #define REL_DTPMOD      R_RISCV_TLS_DTPMOD64
 #define REL_DTPOFF      R_RISCV_TLS_DTPREL64
 #define REL_TPOFF       R_RISCV_TLS_TPREL64
+#define REL_TLSDESC     R_RISCV_TLSDESC

 #define CRTJMP(pc,sp) __asm__ __volatile__( \
        "mv sp, %1 ; jr %0" : : "r"(pc), "r"(sp) : "memory" )
diff --git c/include/elf.h w/include/elf.h
index 72d17c3a..7f342a23 100644
--- c/include/elf.h
+++ w/include/elf.h
@@ -3254,6 +3254,7 @@ enum
 #define R_RISCV_TLS_DTPREL64    9
 #define R_RISCV_TLS_TPREL32     10
 #define R_RISCV_TLS_TPREL64     11
+#define R_RISCV_TLSDESC         12

 #define R_RISCV_BRANCH          16
 #define R_RISCV_JAL             17
diff --git c/src/ldso/riscv64/tlsdesc.s w/src/ldso/riscv64/tlsdesc.s
new file mode 100644
index 00000000..56d1ce89
--- /dev/null
+++ w/src/ldso/riscv64/tlsdesc.s
@@ -0,0 +1,33 @@
+.text
+.global __tlsdesc_static
+.hidden __tlsdesc_static
+.type __tlsdesc_static,%function
+__tlsdesc_static:
+       ld a0,8(a0)
+       jr t0
+
+.global __tlsdesc_dynamic
+.hidden __tlsdesc_dynamic
+.type __tlsdesc_dynamic,%function
+__tlsdesc_dynamic:
+       add sp,sp,-16
+       sd t1,(sp)
+       sd t2,8(sp)
+
+       ld t2,-8(tp) # t2=dtv
+
+       ld a0,8(a0)  # a0=&{modidx,off}
+       ld t1,8(a0)  # t1=off
+       ld a0,(a0)   # a0=modidx
+       sll a0,a0,3  # a0=8*modidx
+
+       add a0,a0,t2 # a0=dtv+8*modidx
+       ld a0,(a0)   # a0=dtv[modidx]
+       add a0,a0,t1 # a0=dtv[modidx]+off
+       sub a0,a0,tp # a0=dtv[modidx]+off-tp
+
+       ld t1,(sp)
+       ld t2,8(sp)
+       add sp,sp,16
+       jr t0
+
```

```sh
(mkdir -p out/rv64 && cd out/rv64 && ../../configure --target=riscv64-linux-gnu && make -j 50)
```

Adjust `~/musl/out/rv64/lib/musl-gcc.specs` and update `~/musl/out/rv64/obj/musl-gcc`
```sh
cat > ~/musl/out/rv64/obj/musl-gcc <<eof
#!/bin/sh
exec "${REALGCC:-riscv64-linux-gnu-gcc}" "$@" -specs ~/musl/out/rv64/lib/musl-gcc.specs
eof
```

I have also modified musl-clang (clang wrapper).
Adjust `~/musl/out/rv64/obj/musl-clang` to use `--target=riscv64-linux-musl`.
Adjust `~/musl/out/rv64/obj/ld.musl-clang` to define `cc="/tmp/Rel/bin/clang --target=riscv64-linux-gnu"` and invoke `exec /tmp/Rel/bin/ld.lld "$@" -lc`.

Prepare a runtime test mentioned at the end of <https://maskray.me/blog/2021-02-14-all-about-thread-local-storage>
```sh
cat > ./a.c <<eof
#include <assert.h>
int foo();
int bar();
int main() {
  assert(foo() == 2);
  assert(foo() == 4);
  assert(bar() == 2);
  assert(bar() == 4);
}
eof

cat > ./b.c <<eof
#include <stdio.h>
__thread int tls0;
extern __thread int tls1;
int foo() { return ++tls0 + ++tls1; }
static __thread int tls2, tls3;
int bar() { return ++tls2 + ++tls3; }
eof

echo '__thread int tls1;' > ./c.c

sed 's/        /\t/' > ./Makefile <<'eof'
.MAKE.MODE = meta curDirOk=true

CC := ~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w
LDFLAGS := -Wl,-rpath=.

all: a0 a1 a2

run: all
        ./a0 && ./a1 && ./a2

c.so: c.o; ${LINK.c} -shared $> -o $@
bc.so: b.o c.o; ${LINK.c} -shared $> -o $@
b.so: b.o c.so; ${LINK.c} -shared $> -o $@

a0: a.o b.o c.o; ${LINK.c} $> -o $@
a1: a.o b.so; ${LINK.c} $> -o $@
a2: a.o bc.so; ${LINK.c} $> -o $@
eof
```

`bmake run` => succeeded!

```
% bmake run
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -c a.c
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -c b.c
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -c c.c
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. a.o b.o c.o -o a0
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. -shared c.o -o c.so
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. -shared b.o c.so -o b.so
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. a.o b.so -o a1
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. -shared b.o c.o -o bc.so
~/musl/out/rv64/obj/musl-clang -O1 -g -fpic -mtls-dialect=desc -w -g  -Wl,-rpath=. a.o bc.so -o a2
./a0 && ./a1 && ./a2
```

### Test GCC

During my development of the linker patch, the Clang Driver patch was actually not ready yet.
I used a more hacky approach by compiling using GCC, replacing some assembly fragments with TLSDESC code sequences, and assemblying using Clang.

Compile `b.c` to `bb.s`. Replace general-dynamic code sequences (e.g. `la.tls.gd a0,tls0; call __tls_get_addr@plt`) with TLSDESC, e.g.
```
.Ltlsdesc_hi0:
  auipc a0, %tlsdesc_hi(tls0)
  ld  a1, %tlsdesc_load_lo(.Ltlsdesc_hi0)(a0)
  addi  a0, a0, %tlsdesc_add_lo(.Ltlsdesc_hi0)
  jalr  t0, 0(a1), %tlsdesc_call(.Ltlsdesc_hi0)
  add   a0, a0, tp
```

Create an alias `bin/ld.lld` to be used with `-Bbin -fuse-ld=lld`.
I made some adjustment to the `Makefile` so that an invocation looks like:

```
% bmake run
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -c a.c
/tmp/Rel/bin/clang --target=riscv64-linux -c bb.s -o b.o
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -c c.c
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. a.o b.o c.o -o a0
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. -shared c.o -o c.so
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. -shared b.o c.so -o b.so
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. a.o b.so -o a1
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. -shared b.o c.o -o bc.so
~/musl/out/rv64/obj/musl-gcc -O1 -g -fpic -Bbin -fuse-ld=lld -g  -Wl,-rpath=. a.o bc.so -o a2
./a0 && ./a1 && ./a2
```
