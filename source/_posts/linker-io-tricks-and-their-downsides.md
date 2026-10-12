---
layout: post
title: Linker I/O tricks and their downsides
date: 2026-09-20
tags: [linker]
---

mold and wild enable several tricks by default that other linkers don't do.
This post looks at three of them:

* overwrite an existing output file in place instead of creating a new file
* fork a child to do the link, so that the parent can exit before the child releases its memory
* ask the kernel for transparent huge pages

Each saves a few percent of wall time in an edit-relink loop.
Each also breaks an assumption that build systems, debuggers, and profilers make about a process.
I measured what they save and what they break.

Setup: Linux 6.18, i7-14700K (28 threads), ext4 on a SATA SSD, transparent huge pages `enabled=always`.
GNU ld 2.47, gold 2.47, LLD 23, mold 2.42.1 (TODO: I used the Rust port under development; check against the C++ release), wild at commit 47cc8e8e.

<!-- more -->

## Overwrite the output file in place

### History

In 2006, Ali Bahrami wrote [Settling An Old Score (Linker Division)](https://www.linker-aliens.org/blogs/ali/entry/settling_an_old_score_linker/).
The Solaris linker truncated and rewrote an existing output file in place.
It killed any running process that had the old file mapped, with a SIGSEGV or SIGBUS far from the real cause.
Linux and macOS linkers unlinked the old file and created a new one.
Bahrami filed PSARC 2006/353 to switch Solaris to unlink-then-create.

GNU ld, gold, and lld still work this way.
A successful relink creates a new inode and leaves other hard links alone.
A failed relink removes the output and leaves other hard links alone.
lld unlinks the old file first (`unlinkAsync`: open it, unlink it, close the descriptor on another thread so that freeing its blocks is off the critical path), mmaps a temporary file `<out>.tmp%%%%%%%` in the same directory, and renames it over the output on success.

### mold

mold went back to in-place overwriting almost from the start.
Its November 2020 history has commits trying `O_TRUNC` and `O_DIRECT`; after them the output is opened without `O_TRUNC`.
[Commit 8a415515](https://github.com/rui314/mold/commit/8a41551562e58c18af664fff5a48531d1a36941e) (2020-12-23, "Handle ETXTBSY") added the current scheme, released in v0.1:

* rename the existing output to `.<name>.<pid>`
* open it read-write without `O_TRUNC`, `ftruncate` to the new size, `fallocate`, mmap, write
* rename it back at the end
* if the open fails with `ETXTBSY` (the file is being executed), unlink the renamed original and create a new file

If the link fails after the output is opened (a relocation out of range, a disk-full SIGBUS), the cleanup handler unlinks the temporary file.
On this path the temporary file is the original inode.

[Issue #188](https://github.com/rui314/mold/issues/188) (2021-12) asked for create-and-rename.
The fix in v1.3.0 (2022-05) only disabled overwriting for `-shared`.
The comment explains why:

> By default, mold tries to ovewrite to an output file if exists because at least on Linux, writing to an existing file is much faster than creating a fresh file and writing to it.
> However, if an existing file is in use, writing to it will mess up processes that are executing that file. Linux prevents a write to a running executable file; it returns ETXTBSY on open(2). However, that mechanism doesn't protect .so files. Therefore, we want to disable this optimization if we are creating a shared object file.

Then the kernel changed.
Linux 6.11 stopped denying writes to running executables ([commit 2a010c412853](https://git.kernel.org/torvalds/c/2a010c41285345da60cece35575b4e0af7e7bf44), "fs: don't block i_writecount during exec").
Its message says: "Yes, someone in userspace could potentially be relying on this. It's not completely out of the realm of possibility but let's find out if that's actually the case and not guess."

In October 2024, [issue #1361](https://github.com/rui314/mold/issues/1361) reported that ninja's bootstrap segfaulted on Gentoo with mold 2.34.1.
The stage-1 ninja was running while the stage-2 link rewrote its inode.
This is Bahrami's scenario.
It took a week and an `rr` recording to find.
mold 2.35.0 disabled the reuse on 6.11 and later.
Rui Ueyama reported the regression to LKML.
Christian Brauner asked on the mold issue whether mold could keep its workaround so that the kernel could keep the change, "as this was done for the sake of new work and because the mechanism is really broken", then reverted it for 6.13 ([commit 3b832035387f](https://git.kernel.org/torvalds/c/3b832035387ff508fdcf0fba66701afc78f79e3d)): "It seems we found out that someone is relying on this obscure behavior. So revert the change."
mold 2.36.0 re-enabled the reuse on 6.13 and later.
6.11 and 6.12 stay excluded: the revert was backported, but distribution kernels can't be told apart by version number.

I commented on the issue that this feels like an instance of Hyrum's Law.
A few years earlier I had insisted that llvm-objcopy emulate `cp`'s overwrite behavior, before I realized the merit of atomic rename.

### wild

wild added `--update-in-place` in [January 2025](https://github.com/davidlattimore/wild/commit/0a66760eded961d23c50be4076ed5ddacb567f69): "On one benchmark (rustc without debug) this gave about a 4% speedup. On another (clang with debug) the difference was small enough that it was hard to measure."
It became the default on Linux in [November 2025](https://github.com/davidlattimore/wild/commit/11c291f5e47d073a86fd71a390c4153764bccf06), with `--no-update-in-place` added [a month later](https://github.com/davidlattimore/wild/commit/61a6a38543e3e10125a2ee634868bdd6dbf24703).
wild opens the output path itself without truncation, falls back to unlink-and-replace on `ETXTBSY`, and always unlinks for shared objects and on macOS (the code signature cache is keyed by inode).
Neither mold nor wild checks `st_nlink`.

### Hard links

```sh
printf '.globl _start\n_start: movl $_start, %%eax; jmp _start\n' | as -o a.o -
mold -o a.out a.o; ln a.out b.out
ls -li a.out b.out   # 47594364 nlink=2, both names
mold -o a.out a.o
ls -li a.out b.out   # 47594364 nlink=2: b.out was rewritten too
ld.bfd -o a.out a.o
ls -li a.out b.out   # a.out 47594365 nlink=1; b.out 47594364, untouched. gold and lld: same
```

A failed relink is worse:

```sh
mold -o a.out a.o; ln -f a.out b.out
mold -o a.out a.o --image-base=0x100000000   # error: relocation R_X86_64_32 against _start out of range
ls -li a.out b.out
# ls: cannot access 'a.out': No such file or directory
# 47594375 -rwxr-xr-x 1 ray ray 3232 Sep 20 18:11 b.out
readelf -h b.out | grep Entry               # Entry point address: 0x1000012b0
```

`a.out` is gone (it was the temporary file).
`b.out` holds the half-written output of the failed link.
GNU ld, gold, and lld would have left `b.out` as the last good binary.

Hard-linked outputs are common.
Cargo hard-links `target/debug/deps/foo-<hash>` to `target/debug/foo`.
rustc relinks `deps/foo-<hash>` while `target/debug/foo` still names the previous inode.
So a failed relink corrupts `target/debug/foo`, and a successful one rewrites it in place.
In my mold checkout, `target/release/mold` and `target/release/deps/mold-75e55cc9443d1590` share one inode.

Hard-link snapshots of a build tree are the other victims:

```sh
cp -al build snap-cp                           # nlink=3
rsync -a --link-dest=../build build/ snap-rsync/
mold -o build/a.out build/a.o --image-base=0x400000
cmp good snap-cp/a.out                         # differ; same for snap-rsync. ld.bfd and ld.lld: intact
```

### What the guards miss

`ETXTBSY` works for its intended case.
With `b.out` running, the relink creates a new inode and the running program keeps the old one:

```sh
./b.out & mold -o a.out a.o; ls -li a.out b.out   # a.out 47594382, b.out 47594381
```

But the kernel only denies writes to an inode that is being `execve`d.
A process that just maps the file (`mmap(PROT_READ|PROT_EXEC, MAP_PRIVATE)`, which is what every user-space ELF loader does) doesn't count.
With such a process alive, `open(O_RDWR)` succeeds and mold reuses the inode.

qemu-user is the important case.
It maps the guest binary's segments `MAP_PRIVATE` from the file descriptor (`linux-user/elfload.c`, `imgsrc_mmap`).
When it is invoked through binfmt_misc, the kernel's write denial goes to `mm->exe_file`, which is the interpreter, not the guest binary: when a binfmt handler substitutes an interpreter, `fs/exec.c` releases the original file's denial and `begin_new_exec` pins the interpreter (a binfmt_misc `T` flag merged after v7.2 changes this).

A static aarch64 program that prints a string from `.data` every 200 ms, started through binfmt_misc (`flags: PF`):

```sh
./guest > out.txt &
readlink /proc/$!/exe                  # /usr/bin/qemu-aarch64-static
ls -i guest                            # 47634213 guest
mold -m aarch64linux -o guest new.o    # same layout, string AAA -> BBB
ls -i guest                            # 47634213 guest: the inode was reused
tr '\n' ' ' < out.txt                  # AAA AAA AAA AAA BBB BBB BBB BBB
```

The running process's output changed.
Its `MAP_PRIVATE` data page was still the page-cache page that the linker wrote through.
For a code page qemu hasn't translated yet, or a data structure mid-update, this is Bahrami's crash.
The same applies to anything that maps or reads an executable without exec'ing it: a `dlopen`ed PIE, valgrind, a `cp` or `rsync` in flight.
`-shared` covers only the most common member of this class.

### What it saves

The saving comes from the page cache.
Pages of a recently written output are still resident, and a write fault on a resident page is cheaper than allocating and zeroing a new one.
To isolate the output side, I linked one 2 GiB `.data` section (`objcopy -I binary` of `/dev/urandom`), so the link is just a copy.
The output is on ext4 on a SATA SSD (all numbers in this section are).
Warm page cache:

| | in place | new inode |
|---|---|---|
| mold | 0.14 s | 0.38 s |
| wild | 0.09 s | 0.17 s |
| lld | | 0.37 s |

That's 40–120 ms per GiB of output, an upper bound.
For a real link it is a few percent, matching wild's commit message.
lld's path is at parity with mold's non-reuse path.
Note that "relink the same output N times and take the minimum" is the best case for this trick.

The pages are not always resident: after a reboot, after a large build, on a memory-constrained CI machine.
Then the first write to each page of the existing file is a fault that has to read the old page from disk before it can be overwritten.
A new file just zero-fills.

```sh
mold --no-fork -o out big.o -e _binary_big_bin_start
sync; python3 -c 'import os; fd=os.open("out",os.O_RDONLY); os.posix_fadvise(fd,0,0,os.POSIX_FADV_DONTNEED)'
fincore -n out                                        #    0B 0    2G out
r0=$(awk '{print $3}' /sys/block/sdb/stat)            # sectors read so far
/usr/bin/time -f %e mold --no-fork -o out big.o -e _binary_big_bin_start   # 3.89
echo $(( ($(awk '{print $3}' /sys/block/sdb/stat) - r0) / 2048 ))MiB       # 2048MiB
rm out
r0=$(awk '{print $3}' /sys/block/sdb/stat)
/usr/bin/time -f %e mold --no-fork -o out big.o -e _binary_big_bin_start   # 0.38
echo $(( ($(awk '{print $3}' /sys/block/sdb/stat) - r0) / 2048 ))MiB       # 0MiB
```

The in-place link reads the old 2 GiB back at SATA speed, only to overwrite it.
wild: 3.79 s in place, 0.15 s with `--no-update-in-place`.
Network storage would be worse.
A linker that wants to keep the trick could check `cachestat(2)` (Linux 6.5) and `st_nlink == 1`.

I don't plan to adopt this in lld.
It changes semantics other tools rely on, it depends on a kernel behavior that was removed once, it is Linux-only, and it gains a few percent in the best case with a 10× cliff in the worst.

## Fork so that the parent can exit early

### Mechanism

Before starting any thread, mold creates a pipe and forks.
The parent blocks on the pipe.
The child does the whole link.
After the output file is closed and renamed, the child writes one byte to the pipe.
The parent calls `_exit(0)` without waiting for the child.
The child then releases its input mappings in parallel and exits.
If the child dies before writing the byte, the parent `waitpid`s and propagates the status.
The comment: "Exiting from a program with large memory usage is slow -- it may take a few hundred milliseconds. To hide the latency, we fork a child and let it do the actual linking work."
`--no-fork` disables it.

wild does the same, with the same pipe protocol and the same `--no-fork`.

### What it saves

The hidden latency is the kernel's `exit_mmap`, which frees the address space on one thread.
A program that touches N GiB of anonymous memory and then measures `_exit` to `wait()` in the parent:

| heap | 1 GiB | 4 GiB | 8 GiB |
|---|---|---|---|
| 4 KiB pages (`PR_SET_THP_DISABLE`) | 37 ms | 148 ms | 203 ms |
| huge pages | 2 ms | 6 ms | 36 ms |

So the saving depends on whether the heap uses huge pages.
mold's does here: it uses mimalloc and calls `prctl(PR_SET_THP_DISABLE, 0)` at startup, so with `enabled=always` 99.8% of its 1.7 GB heap was in `AnonHugePages`.
(That `prctl` also undid my first attempt to disable THP from a wrapper. I had to intercept the call with `LD_PRELOAD`.)

A synthetic link with 5.8M symbols and a ~1.5 GB heap, interleaved, 5 runs each:

| heap | fork | `--no-fork` |
|---|---|---|
| 4 KiB pages | 0.37–0.40 s | 0.44–0.46 s |
| huge pages | 0.48–0.50 s | 0.50–0.51 s |

60–70 ms with 4 KiB pages, matching the table above.
About 10 ms with huge pages.
On a 43 MB output (the C++ mold: 700 objects, 400 MB linker RSS) the difference was not measurable.

### What it breaks

The parent is the process the build system spawned and the only one it `wait`s for.
The child is reparented to init or a subreaper.
Nothing that watched the parent sees it again.

Resource accounting sees nothing.
`/usr/bin/time`, the shell's `time`, `getrusage(RUSAGE_CHILDREN)`, and anything else built on `wait4` report the parent:

```sh
/usr/bin/time -f "wall=%e user=%U sys=%S maxrss=%MkB" mold @ld.args
# wall=0.04 user=0.00 sys=0.00 maxrss=11152kB
/usr/bin/time -f "wall=%e user=%U sys=%S maxrss=%MkB" mold --no-fork @ld.args
# wall=0.04 user=0.26 sys=0.51 maxrss=401692kB
```

wild: 4 MB reported, 1.65 GB actual.
The wall time also stops at the notification, so a wall-clock comparison between a fork-mode linker and another linker doesn't measure the same interval.

gdb follows the parent by default and detaches from the child, so a breakpoint in the link never fires:

```
(gdb) break rename
(gdb) run
[Detaching after fork from child process 2677804]
[Inferior 1 (process 2677801) exited normally]
```

`set follow-fork-mode child` or `--no-fork` fixes it.
`strace` needs `-f`: without it you see the parent's 252 syscalls (exec, `pipe`, one `clone`, a `read`), not the child's 19,801.
`perf stat` works: its default counter inheritance includes the child, even after the parent exits (774 ms task-clock for fork mode, 648 ms for `--no-fork`).

The job slot is freed before the memory is.
When the parent exits, make or ninja starts the next job while the orphan is still freeing its heap.
I ran four links under `make -jN` in a transient cgroup (`systemd-run --user --scope`) and sampled the cgroup's anonymous memory every 5 ms.
The orphan stays in the cgroup, so this is the build's real footprint:

| heap | `-j1` fork | `-j1` `--no-fork` | `-j2` fork | `-j2` `--no-fork` |
|---|---|---|---|---|
| 4 KiB pages | 2.13 GB | 1.50 GB | 3.44 GB | 2.97 GB |
| huge pages | 1.72 GB | 1.72 GB | 3.43 GB | 3.42 GB |

With 4 KiB pages, `-j1` peaks 40% above one link, and `-j2` about 15% above.
With huge pages the orphan is gone in ~5 ms and there is no difference.
The excess is bounded by one heap per job slot and lasts for the orphan's lifetime.
It is a margin problem, not a doubling.
But a machine sized to the edge of RAM has that much less margin, and the build system can't see it.

Per-process RSS can't see it either.
Within half a millisecond of the parent's exit, the orphan shows `State: Z` with no `VmRSS` in `/proc/<pid>/status`, and `statm` reads 0, while the cgroup still holds 1.5 GB for another 60 ms.
(In `exit_group`, each thread drops its `mm` before the last one runs `exit_mmap`, and `/proc/<pid>` reports the group leader.)
Only system-wide or cgroup counters see it.

Errors after the notification are lost.
The parent has reported success before the child releases its mappings and exits.
Nothing in that window can change the exit status.

### Assessment

With a 4 KiB-page heap the trick saves the kernel's exit cost, 25–40 ms per GiB of heap, a few hundred milliseconds for a multi-gigabyte debug link.
With transparent huge pages, which mold enables for itself, it saves about 10 ms on the same link.
In exchange, the linker is no longer the process the build system started, and every tool built on `wait4` misreports it.
If you benchmark, profile, debug, or memory-budget a build with mold or wild, pass `--no-fork`.

## Ask for huge pages

### Mechanism

mold calls `madvise(MADV_HUGEPAGE)` on its output file mapping (`src/output-file-unix.cc`: "Linking a Chromium debug build is ~20% faster with this"), its arena allocator (`lib/arena.cc`), and its large hash tables (`lib/lib.h`).
It also calls `prctl(PR_SET_THP_DISABLE, 0)` at startup to undo an inherited THP-disable flag.
wild hints its output mapping by default since [September 2026](https://github.com/davidlattimore/wild/commit/96944c96cb7740235c97325860a7d3fea5257054).
lld does not hint.

`madvise` only sets `VM_HUGEPAGE` on the mapping.
The effect comes at page faults.
The first touch of each 2 MiB of an anonymous region asks the allocator for a physically contiguous 2 MiB block.

* A free block exists: the fault returns a huge page. With `enabled=always`, an unadvised region gets one too.
* No free block, because free memory is fragmented or low: an unadvised region takes a 4 KiB page and moves on. An advised region, under the default `defrag=madvise`, allocates with `__GFP_DIRECT_RECLAIM` (`vma_thp_gfp_mask()` in `mm/huge_memory.c`). The faulting thread runs compaction itself, moving other processes' pages out of a candidate block until 2 MiB is free, and reclaim first if memory is low.
* Compaction fails when every candidate block holds a page that can't be moved. The fault then retries with a 4 KiB page, having paid for the attempt.

The hint opts the program into doing the kernel's defragmentation in its own page faults.
The output file mapping is a milder case: page cache large folios use `readahead_gfp_mask()`, which adds `__GFP_NORETRY`, so that path gives up quickly.

### What it does

Free 2 MiB blocks are scarce on an ordinary developer machine.
This one has been up 48 days: 18 GB free, but only 3,400 free 2 MiB blocks (6.6 GB) in `/proc/buddyinfo`.
A Chromium debug link (13 GB of inputs, 4.5 GB output, 19.6 GB linker RSS) uses them up.

`nohuge.so` is an `LD_PRELOAD` shim that turns `madvise(MADV_HUGEPAGE)` into a no-op.

```sh
hyperfine -N -w 1 -r 5 --prepare sync "mold --no-fork @rsp -o ext4out/a.out" "env LD_PRELOAD=./nohuge.so mold --no-fork @rsp -o ext4out/a.out"
# ext4:  3.374 s ± 0.357 s (hint)  vs  2.877 s ± 0.044 s (no hint)
# tmpfs: 2.825 s ± 0.113 s (hint)  vs  2.720 s ± 0.015 s (no hint)
```

`/proc/vmstat` deltas over one link, ext4:

```
hint:    compact_stall=+594 compact_fail=+483 compact_success=+111 thp_fault_alloc=+3759 thp_fault_fallback=+195 thp_file_mapped=+546
no hint: compact_stall=+0   compact_fail=+0   compact_success=+0   thp_fault_alloc=+3708 thp_fault_fallback=+247 thp_file_mapped=+497
```

Both runs get the same 3,700 huge pages; `enabled=always` does that without the hint.
The hint adds 550 compaction stalls per link, 80% of them failed, and the link is slower and noisier.
The ~20% may hold with more free contiguous memory, or with `enabled=madvise`, where nothing gets huge pages without the hint.
I have no data for those.

Adding the hint to lld (an `LD_PRELOAD` wrapper that calls `madvise(MADV_HUGEPAGE)` after every `mmap` of 2 MiB or more) changes nothing either:

```sh
hyperfine -N -w 2 -r 12 "ld.lld @rsp -o ext4out/a.out" "env LD_PRELOAD=./thp_wrapper.so ld.lld @rsp -o ext4out/a.out"
# clang,  ext4:  224.7 ms ± 15.5 ms  vs  232.8 ms ± 27.5 ms
# clang,  tmpfs: 194.6 ms ±  8.6 ms  vs  192.8 ms ± 12.0 ms
# chrome, ext4:  4.708 s ± 0.090 s   vs  4.840 s ± 0.075 s   (594 MB output)
# chrome, tmpfs: 4.553 s ± 0.036 s   vs  4.611 s ± 0.076 s
```
