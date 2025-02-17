layout: post
title: Compiler output files
author: MaskRay
tags: [clang,gcc]
---

For a GCC or Clang command, there is typically one primary output file, specified by `-o` or the default (`a.out` or `a.exe`).
There can also be temporary files and auxiliary files.

<!-- more -->

## Primary output file

We can specify the primary output file with the `-o` option.
When unspecified, a default output file name is inferred from the input files and the final phase.
When the final phase is linking, the default output file name is `a.out` or `a.exe`.

```sh
gcc -S d/a.c      # a.s
gcc -c d/a.c      # a.o
gcc d/a.c         # a.out
gcc d/a.c -o e/a  # e/a
```

For `-S` and `-c`, specifying `-o` in the presence of more than one input files leads to an error.

```
% gcc -c d/a.c d/b.c -o e/x
gcc: fatal error: cannot specify ‘-o’ with ‘-c’, ‘-S’ or ‘-E’ with multiple files
compilation terminated.
```

## Temporary files

For compilation and linking in one command, the linker output file is the primary output file while the relocatable object files are temporary files.

For GCC, when generating a relocatable object file, it needs to generate a temporary assembly file, then feeds it to GNU assembler.

We can set the temporary directory with one of the environment variables `TMPDIR`, `TMP`, and `TEMP`.
When none is specified, compilers have a fallback, `/tmp` on \*NIX systems.

## Auxiliary files

Beside primary output files and temporary output files, we have a third output file type called auxiliary output files.
Some options such as `-ftest-coverage` (see [below](#gcov)) and [`-gsplit-dwarf`](/blog/2022-10-30-distribution-of-debug-information#split-dwarf-object-files) cause the compiler to generate auxiliary output files.

For compilation without linking (`-c`, `-S`, etc), the auxiliary output file names are derived from the primary output file name.

For compilation and linking in one command, the primary output file name affects the auxiliary output file names.

```sh
gcc -c -g -gsplit-dwarf d/a.c d/b.c      # a.o b.o a.dwo b.dwo
gcc -c -g -gsplit-dwarf d/a.c -o e/a.o   # e/a.o e/a.dwo
gcc -g -gsplit-dwarf d/a.c d/b.c -o e/x  # e/x (temporary .o files) e/x-a.dwo e/x-b.dwo
gcc -g -gsplit-dwarf d/a.c d/b.c         # (temporary .o files) a-a.dwo a-b.dwo

# a.out is special cased.
gcc -g -gsplit-dwarf d/a.c.c -o e/x.out  # e/x.out-a.dwo
gcc -g -gsplit-dwarf d/a.c.c -o e/a.out  # e/a-a.dwo
```

GCC provides some [options](https://gcc.gnu.org/onlinedocs/gcc/Developer-Options.html) that are primarily of interest to GCC developers.
These dump output files are treated the same way as auxiliary output files.

`-dumpdir` and `-dumpbase` are provided to control the auxiliary output file names.
The [official documentation](https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html) may be difficult to follow.
Let's see some examples.

```sh
gcc -g -gsplit-dwarf -dumpdir f d/a.c -c # fa.dwo
gcc -g -gsplit-dwarf -dumpdir f d/a.c    # fa.dwo
gcc -g -gsplit-dwarf -dumpdir f/ d/a.c   # f/a.dwo
gcc -g -gsplit-dwarf -dumpbase f d/a.c   # f-a.dwo
gcc -g -gsplit-dwarf -dumpbase f/ d/a.c  # f/-a.dwo

gcc -g -gsplit-dwarf d/a.c -o e/x -dumpdir f/g  # f/ga.dwo
gcc -c -g -gsplit-dwarf d/a.c -o e/xa.o -dumpdir f/g  # f/gxa.dwo; different from above
```

In the absence of `-dumpdir`, `-dumpbase` appends a dash, which makes it inconvenient.

If we specify both `-dumpdir` and `-dumpbase`, we can avoid the influence of the source filename when there is one input file.
```sh
gcc -g -gsplit-dwarf -dumpdir f/ -dumpbase x d/a.c        # f/x.dwo
gcc -g -gsplit-dwarf -dumpdir f/ -dumpbase x d/a.c d/b.c  # f/x-a.dwo f/x-b.dwo
```

I suggest that you only use `-dumpdir`.
When only `-dumpdir` is used, the behavior is still easy to explain.

I added `-dumpdir` to Clang 17 ([D149193](https://reviews.llvm.org/D149193)).
The behavior is simplified from GCC's. In the common cases the behavior are identical.

Some GCC's strange rules are not ported.
For example, `clang -c -g -gsplit-dwarf d/a.c -o e/xa.o -dumpdir f/g` creates `f/ga.dwo` instead of `f/gxa.dwo`.
When `-dumpdir` is specified, `-o` is completely for deriving the auxiliary output file names.

## `-save-temps`

[`-save-temps` and `-save-temps={cwd,obj}`](https://gcc.gnu.org/onlinedocs/gcc/Developer-Options.html) generate intermediate files, which are treated as auxiliary output files in GCC.

```sh
gcc -save-temps d/a.c
ls a.i a.s a.o
```

In the absence of `-dumpdir`/`-dumpbase`, `-save-temps=cwd` places intermediate files in the current directory while `-save-temps=obj` places intermediate files in the directory of the primary output file.
`-save-temps` behaves like `-save-temps=obj` when `-o` is specified, and `-save-temps=cwd` otherwise.

When `-dumpdir` is specified, there is complex interaction between `-dumpdir` and `-save-temps`/`-save-temps={cwd,obj}`.
```
# The last of -dumpdir and -save-temps wins.
gcc -g -gsplit-dwarf d/a.c -o e/x -dumpdir f/ -save-temps=obj # e/a.{i,s,o,dwo}
gcc -g -gsplit-dwarf d/a.c -o e/x -save-temps=obj -dumpdir f/ # f/a.{i,s,o,dwo}
```

For Clang, I think we should abandon the idea to treat these intermdiate files (`*.i`, `*.bc`, `*.s`, etc) as auxiliary output files. Just make `-dumpdir` and `-save-temps`/`-save-temps={cwd,obj}` orthogonal.
```sh
clang -g -gsplit-dwarf d/a.c -o e/x -save-temps=obj -dumpdir f/ # e/a.{i,s,o} f/a.dwo
```

## Other auxiliary files

Clang supports a few options (e.g. `-ftime-trace`) to generate other auxiliary output files.
I plan to change their file names to be controlled by `-dumpdir`.

### `-ftime-trace`

This option instructs Clang to print time summaries for stages of the compilation process.
The output is a time trace file in JSON that can be loaded into <https://ui.perfetto.dev> or the legacy <chrome://tracing> UI.

There is a variant `-ftime-trace=` that specifies the output JSON file or a directory to place the trace file.

After my <https://reviews.llvm.org/D150282>, Clang driver derives the trace file from `-o` or `-dumpdir` using a `-gsplit-dwarf` style approach.
```sh
#!/bin/sh -e
PATH=/tmp/Rel/bin:$PATH
mkdir -p d e f
echo 'int main() {}' > d/a.c
echo > d/b.c

a() { rm $1 || exit 1; }

clang -ftime-trace d/a.c d/b.c         # previously /tmp/[ab]-*.json
a a-a.json; a a-b.json
clang -ftime-trace d/a.c d/b.c -o e/x  # previously /tmp/[ab]-*.json
a e/x-a.json; a e/x-b.json
clang -ftime-trace d/a.c d/b.c -o e/x -dumpdir f/
a f/a.json; a f/b.json
clang -ftime-trace=f d/a.c d/b.c -o e/x
a f/a-*.json; a f/b-*.json

clang -c -ftime-trace d/a.c d/b.c
a a.json b.json
clang -c -ftime-trace=f d/a.c d/b.c
a f/a.json f/b.json

clang -c -ftime-trace d/a.c -o e/xa.o
a e/xa.json
clang -c -ftime-trace d/a.c -o e/xa.o -dumpdir f/g
a f/ga.json
```

Feature request to support device compilation for offloading targets: <https://github.com/llvm/llvm-project/issues/55455>.
Feature request to support `clang++ -c -std=c++20 -ftime-trace a.cppm -o a.o`: <https://github.com/llvm/llvm-project/issues/60555>.

GCC [feature request](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=92396) to implement the option.

### `-fproc-stat-report`

This option ([D78903](https://reviews.llvm.org/D78903)) dumps the primary output filename and memory usage to stdout.

There is a variant `-fproc-stat-report=a.txt` that dumps the information to a text file.
The file will be opened in the append mode.

The output behavior is quite unusual.

### gcov

GCC implements a code coverage feature called gcov. The driver provides `-ftest-coverage` for generating notes files (`.gcno`) and `-fprofile-arcs` for instrumenting code to produce data files (`.gcda`) during execution.
For convenience, `--coverage` encompasses both options.

Previously, when Clang performs compilation and linking in one command, it places `.gcno` files in the current directory.
The generated `.gcda` files during program execution go to the current directory as well.
I have [changed](https://github.com/llvm/llvm-project/commit/a07b135ce0c0111bd83450b5dc29ef0381cdbc39) Clang to consult the primary output file for `.gcno` and `.gcda` files.

```sh
#!/bin/zsh -e
PATH=/tmp/Rel/bin:$PATH                # adapt according to your build directory
mkdir -p d e f
echo 'int main() {}' > d/a.c
echo > d/b.c

a() { rm $@ || exit 1; }

clang --coverage d/a.c d/b.c && ./a.out
a a-[ab].gc{no,da}
clang --coverage d/a.c d/b.c -o e/x && e/x
a e/x-[ab].gc{no,da}
clang --coverage d/a.c d/b.c -o e/x -dumpdir f/ && e/x
a f/[ab].gc{no,da}
clang --coverage -fprofile-dir=f d/a.c d/b.c -o e/x && e/x
a e/x-[ab].gcno f$PWD/e/x-[ab].gcda
# gcc: e/x-[ab].gcno f/${PWD//\//#}#e#x-a.gcda

clang -c --coverage d/a.c d/b.c && clang --coverage a.o b.o && ./a.out
a [ab].gc{no,da}
clang -c --coverage -fprofile-dir=f d/a.c d/b.c && clang --coverage a.o b.o && ./a.out
a [ab].gcno f$PWD/[ab].gcda
# gcc: [ab].gcno f/${PWD//\//#}#a.gcda

clang -c --coverage d/a.c -o e/xa.o && clang --coverage e/xa.o && ./a.out
a e/xa.gc{no,da}
clang -c --coverage d/a.c -o e/xa.o -dumpdir f/g && clang --coverage e/xa.o && ./a.out
a f/ga.gc{no,da}
# gcc: a f/gxa.gc{no,da}
```

GCC provides `-fprofile-dir=` to set the location for generating `.gcda` files.
When the primary output path is absolute, GCC mangles the absolute output file path to create a file within the `-fprofile-dir=` directory.

I find this behavior to be peculiar. Clang, on the other hand, always creates hierarchies within the `-fprofile-dir=` directory.
```
% c() { for i in **/*.gcno **/*.gcda; do echo $i; 'rm' -f $i; done; }
% gcc --coverage -fprofile-dir=f d/a.c -o e/x && e/x && c
./f/#tmp#c#e#x-a.gcda
./e/x-a.gcno
% gcc --coverage -fprofile-dir=f d/a.c -o $PWD/e/x && e/x && c
e/x-a.gcno
f/tmp/c/e/x-a.gcda
% clang --coverage -fprofile-dir=f d/a.c -o e/x && e/x && c
e/x-a.gcno
f/tmp/c/e/x-a.gcda
% clang --coverage -fprofile-dir=f d/a.c -o $PWD/e/x && e/x && c
e/x-a.gcno
f/tmp/c/e/x-a.gcda
```

The compiled object files encode the paths of the emitted `.gcda` files, so the directory of the final executable doesn't matter.
The absolute `.gcda` paths have a benefit. For a recursive Make style build systems, we will get different `*.gcda` files even if two directories contain files of the same name.
```
(cd d0; gcc -c --coverage a.c)
(cd d1; gcc -c --coverage a.c)
gcc --coverage d0/a.o d1/a.o -o e/x
```

However, the absolute `.gcda` paths are not friendly to build determinism.
To achieve local determinism (make builds independent of the build directory's name), we need to set the `PWD` environment variable.
```
% gcc --coverage d/a.c -c -o e/x.o && strings e/x.o | grep gcda
     b0 /tmp/c/e/x.gcda
% PWD=/proc/self/cwd gcc --coverage d/a.c -c -o e/x.o && strings e/x.o | grep gcda
     b0 /proc/self/cwd/e/x.gcda
```

Unfortunately, `PWD=/proc/self/cwd/d` doesn't work (say, `d` is a directory within the current directory).
If we have a recursive Make style build system and files of the same name within differect directories, it's very tricky to avoid duplicate `.gcda` paths while achieving local determinism.
In my experiments, `-fprofile-prefix-map=` and `-ffile-prefix-map=` don't affect the emitted `.gcda` path.

## Offloading

Clang supports offloading to various architectures using programming models like CUDA, HIP, and OpenMP.
Using offloading options may generate multiple output files.

For example, the following command spawns a host compile action and two device compile actions.
Many features with auxiliary output files don't support offloading.
Uses are not rejected, but the behavior may be undesired (e.g. an auxiliary ouput file gets overwritten by multiple actions).
```sh
clang -c --target=x86_64-unknown-linux-gnu --cuda-gpu-arch=gfx803 --cuda-gpu-arch=gfx900 -fgpu-rdc -nogpulib -nogpuinc -ftime-trace a.hip -ccc-print-phases
```

TODO offloading and -save-stats=obj
