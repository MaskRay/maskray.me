layout: post
title: lld 16 ELF changes
author: MaskRay
tags: [linker,llvm]
---

llvm-project 16 was just released.
I added some lld/ELF notes to <https://github.com/llvm/llvm-project/blob/release/16.x/lld/docs/ReleaseNotes.rst>.
Here I will elaborate on some changes.

<!-- more -->

* Link speed improved greatly compared with lld 15.0. Notably input section
  initialization and relocation scanning are now parallel.
  ([D130810](https://reviews.llvm.org/D130810))
  ([D133003](https://reviews.llvm.org/D133003))
* `ELFCOMPRESS_ZSTD` compressed input sections are now supported.
  ([D129406](https://reviews.llvm.org/D129406))
* `--compress-debug-sections=zstd` is now available to compress debug sections with zstd (`ELFCOMPRESS_ZSTD`).
  ([D133548](https://reviews.llvm.org/D133548))
* `--no-warnings`/`-w` is now available to suppress warnings.
  ([D136569](https://reviews.llvm.org/D136569))
* `DT_RISCV_VARIANT_CC` is now produced if at least one `R_RISCV_JUMP_SLOT` relocation references a symbol with the `STO_RISCV_VARIANT_CC` bit.
  ([D107951](https://reviews.llvm.org/D107951))
* `DT_STATIC_TLS` is now set for AArch64/PPC32/PPC64 initial-exec TLS models
  when producing a shared object.
* `--no-undefined-version` is now the default; symbols named in version
  scripts that have no matching symbol in the output will be reported. Use
  `--undefined-version` to revert to the old behavior.
  ([D135402](https://reviews.llvm.org/D135402))
* `-V` is now an alias for `-v` to support `gcc -fuse-ld=lld -v` on many targets.
* `-r` no longer defines `__global_pointer$` or `_TLS_MODULE_BASE_`.
* A corner case of mixed GCC and Clang object files (`STB_WEAK` and
  `STB_GNU_UNIQUE` in different COMDATs) is now supported.
  ([D136381](https://reviews.llvm.org/D136381))
* The output `SHT_RISCV_ATTRIBUTES` section now merges all input components
  instead of picking the first input component.
  ([D138550](https://reviews.llvm.org/D138550))
* For x86-32, `-fno-plt` GD/LD TLS models `call *___tls_get_addr@GOT(%reg)`
  are now supported. Previous output might have runtime crash.
* Armv4(T) thunks are now supported.
  ([D139888](https://reviews.llvm.org/D139888))
  ([D141272](https://reviews.llvm.org/D141272))

## Speed

Link speed has greatly improved compared to lld 15.0.0.

In this release cycle, I made input section initialization and relocation scanning parallel. ([D130810](https://reviews.llvm.org/D130810) [D133003](https://reviews.llvm.org/D133003))

Different `OutputSection`s are now written in parallel. ([D131247](https://reviews.llvm.org/D131247))

```
% hyperfine --warmup 1 --min-runs 20 "numactl -C 20-27 "{/tmp/out/custom15,/tmp/out/custom16}"/bin/ld.lld @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):     676.3 ms ±   4.4 ms    [User: 751.5 ms, System: 472.3 ms]
  Range (min … max):   667.9 ms … 686.1 ms    20 runs

Benchmark 2: numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):     453.5 ms ±   5.3 ms    [User: 760.5 ms, System: 448.9 ms]
  Range (min … max):   445.4 ms … 462.8 ms    20 runs

Summary
  'numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8' ran
    1.49 ± 0.02 times faster than 'numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8'
```
(`--threads=4` => 1.38x (0.7009s => 0.5096s))


Linking a `-DCMAKE_BUILD_TYPE=Debug` build of clang:
```
% hyperfine --warmup 2 --min-runs 25 "numactl -C 20-27 /tmp/out/custom"{15,16}"/bin/ld.lld @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.664 s ±  0.039 s    [User: 6.657 s, System: 3.434 s]
  Range (min … max):    3.605 s …  3.781 s    25 runs

Benchmark 2: numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.141 s ±  0.028 s    [User: 7.071 s, System: 3.150 s]
  Range (min … max):    3.086 s …  3.212 s    25 runs

Summary
  'numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8' ran
    1.17 ± 0.02 times faster than 'numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8'
```
(`--threads=1` => 1.06x (7.620s => 7.202s), `--threads=4` => 1.11x (4.138s => 3.727s))

Linking a default build of chrome:
```
% hyperfine --warmup 2 --min-runs 25 "numactl -C 20-27 /tmp/out/custom"{15,16}"/bin/ld.lld @response.txt --threads=8"
Benchmark 1: numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      4.159 s ±  0.018 s    [User: 6.703 s, System: 2.705 s]
  Range (min … max):    4.136 s …  4.207 s    25 runs

Benchmark 2: numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8
  Time (mean ± σ):      3.700 s ±  0.027 s    [User: 7.417 s, System: 2.556 s]
  Range (min … max):    3.660 s …  3.771 s    25 runs

Summary
  'numactl -C 20-27 /tmp/out/custom16/bin/ld.lld @response.txt --threads=8' ran
    1.12 ± 0.01 times faster than 'numactl -C 20-27 /tmp/out/custom15/bin/ld.lld @response.txt --threads=8'
```
(`--threads=4` => 1.11x (4.387s => 3.940s))

After parsing command line options, we have a list of input files.
We parse them one by one, initialize sections and symbols, and perform symbol resolution.
Archive member extraction may add new input files to the list, so we use `i < files.size()`.
```cpp
for (size_t i = 0; i < files.size(); ++i)
  parseFile(files[i]);
```

Prior to lld 14, parsing a relocatable object file is relatively straightforward.
```cpp
template <class ELFT> void ObjFile<ELFT>::parse(bool ignoreComdats) {
  initializeSections(ignoreComdats);
  // Initialize local and non-local symbols, and perform symbol resolution.
  // Lazy object file extraction may cause parsing of other files.
  initializeSymbols();
}
```

As of lld 16, the code looks like:
```cpp
template <class ELFT> void ObjFile<ELFT>::parse(bool ignoreComdats) {
  Handle GRP_COMDAT SHT_GROUP sections

  // Initialize non-local symbols, and perform symbol resolution.
  // Lazy object file extraction may cause parsing of other files.
  initializeSymbols(obj);
}

template <class ELFT>
void ObjFile<ELFT>::initializeSymbols(const object::ELFFile<ELFT> &obj) {
  ...
  for (size_t i = firstGlobal, end = eSyms.size(); i != end; ++i)
    if (!symbols[i])
      symbols[i] = symtab.insert(CHECK(eSyms[i].getName(stringTable), this));

  // Perform symbol resolution on non-local symbols.
  SmallVector<unsigned, 32> undefineds;
  for (size_t i = firstGlobal, end = eSyms.size(); i != end; ++i) {
    const Elf_Sym &eSym = eSyms[i];
    uint32_t secIdx = eSym.st_shndx;
    if (secIdx == SHN_UNDEF) {
      undefineds.push_back(i);
      continue;
    }

    // Handle global defined symbols. Defined::section will be set in postParse.
    sym->resolve(Defined{this, StringRef(), binding, stOther, type, value, size,
                         nullptr});
  }

  // Undefined symbols (excluding those defined relative to non-prevailing
  // sections) can trigger recursive extract.
  for (unsigned i : undefineds) {
    const Elf_Sym &eSym = eSyms[i];
    Symbol *sym = symbols[i];
    sym->resolve(Undefined{this, StringRef(), eSym.getBinding(), eSym.st_other,
                           eSym.getType()});
    sym->isUsedInRegularObj = true;
    sym->referenced = true;
  }
}
```

Link: [lld 15 ELF changes](/blog/2022-09-05-lld-15-elf-changes)
