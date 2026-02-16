---
layout: post
title: Removing global state from LLD
author: MaskRay
tags: [linker,llvm]
---

[LLD, the LLVM linker](https://lld.llvm.org/), is a mature and fast linker supporting multiple binary formats (ELF, Mach-O, PE/COFF, WebAssembly).
Designed as a standalone program, the code base relies heavily on global state, making it less than ideal for library integration.
As outlined in [RFC: Revisiting LLD-as-a-library design](https://discourse.llvm.org/t/rfc-revisiting-lld-as-a-library-design/58445), two main hurdles exist:

* Fatal errors: they exit the process without returning control to the caller. This was actually addressed for most scenarios in 2020 by utilizing `llvm::sys::Process::Exit(val, /*NoCleanup=*/true)` and `CrashRecoveryContext` (`longjmp` under the hood).
* Global variable conflicts: shared global variables do not allow two concurrent invocation.

I understand that calling a linker API could be convenient, especially when you want to avoid shipping another executable (which can be large when you link against LLVM statically).
However, I believe that invoking LLD as a separate process remains the recommended approach. There are several advantages:

* Build system control: Build systems gain greater control over scheduling and resource allocation for LLD. In an edit-compile-link cycle, the link could need more resources and threading is more useful.
* Better parallelism management
* Global state isolation: LLVM's global state (primarily `cl::opt` and `ManagedStatic`) is isolated.

<!-- more -->

While spawning a new process offers build system benefits, the issue of global state usage within LLD remains a concern.
This is a factor to consider, especially for advanced use cases. Here are global variables in the LLD 15 code base.

```
% rg '^extern [^(]* \w+;' lld/ELF
lld/ELF/SyntheticSections.h
1290:extern InStruct in;

lld/ELF/Symbols.h
51:extern SmallVector<SymbolAux, 0> symAux;

lld/ELF/SymbolTable.h
87:extern std::unique_ptr<SymbolTable> symtab;

lld/ELF/InputSection.h
33:extern std::vector<Partition> partitions;
403:extern SmallVector<InputSectionBase *, 0> inputSections;
408:extern llvm::DenseSet<std::pair<const Symbol *, uint64_t>> ppc64noTocRelax;

lld/ELF/OutputSections.h
156:extern llvm::SmallVector<OutputSection *, 0> outputSections;

lld/ELF/InputFiles.h
43:extern std::unique_ptr<llvm::TarWriter> tar;

lld/ELF/Driver.h
23:extern std::unique_ptr<class LinkerDriver> driver;

lld/ELF/LinkerScript.h
366:extern std::unique_ptr<LinkerScript> script;

lld/ELF/Config.h
372:extern std::unique_ptr<Configuration> config;
406:extern std::unique_ptr<Ctx> ctx;
```

Some global states exist as static member variables.

## Cleaning up global variables

LLD has been undergoing a transformation to reduce its reliance on global variables. This improves its suitability for library integration.

* In 2020, [[LLD][COFF] Cover usage of LLD as a library](https://reviews.llvm.org/D70378) enabled running the LLD driver multiple times even if there is a fatal error.
* In 2021, global variables were [removed from `lld/Common`](https://reviews.llvm.org/D108850).
* The COFF port followed suite, [eliminating most of its global variables](https://reviews.llvm.org/D109634).

Inspired by theseadvancements, I conceived a plan to eliminate global variables from the ELF port.
In 2022, as part of the work to enable parallel section initialization, I [introduced a class `struct Ctx`](https://reviews.llvm.org/D120626) to `lld/ELF/Config.h`.
Here is my plan:

* Global variables will be migrated into `Ctx`.
* Functions will be modified to accept a new `Ctx &ctx` parameter.
* The previously global variable lld::elf::ctx will be transformed into a local variable within `lld::elf::link`.

## Encapsulating global variables into `Ctx`

Over the past two years and a half, I have migrated global variables into the `Ctx` class, [e.g.](https://github.com/llvm/llvm-project/commit/e980f16d52196fb2bc672ecb87e0f622253addec).

```
diff --git a/lld/ELF/Config.h b/lld/ELF/Config.h
index 590c19e6d88d..915c4d94e870 100644
--- a/lld/ELF/Config.h
+++ b/lld/ELF/Config.h
@@ -382,2 +382,10 @@ struct Ctx {
   std::atomic<bool> hasSympart{false};
+  // A tuple of (reference, extractedFile, sym). Used by --why-extract=.
+  SmallVector<std::tuple<std::string, const InputFile *, const Symbol &>, 0>
+      whyExtractRecords;
+  // A mapping from a symbol to an InputFile referencing it backward. Used by
+  // --warn-backrefs.
+  llvm::DenseMap<const Symbol *,
+                 std::pair<const InputFile *, const InputFile *>>
+      backwardReferences;
 };
diff --git a/lld/ELF/Driver.cpp b/lld/ELF/Driver.cpp
index 8315d43c776e..2ab698c91b01 100644
--- a/lld/ELF/Driver.cpp
+++ b/lld/ELF/Driver.cpp
@@ -1776,3 +1776,3 @@ static void handleUndefined(Symbol *sym, const char *option) {
   if (!config->whyExtract.empty())
-    driver->whyExtract.emplace_back(option, sym->file, *sym);
+    ctx->whyExtractRecords.emplace_back(option, sym->file, *sym);
 }
@@ -1812,3 +1812,3 @@ static void handleLibcall(StringRef name) {

-void LinkerDriver::writeArchiveStats() const {
+static void writeArchiveStats() {
   if (config->printArchiveStats.empty())
@@ -1834,3 +1834,3 @@ void LinkerDriver::writeArchiveStats() const {
       ++extracted[CachedHashStringRef(file->archiveName)];
-  for (std::pair<StringRef, unsigned> f : archiveFiles) {
+  for (std::pair<StringRef, unsigned> f : driver->archiveFiles) {
     unsigned &v = extracted[CachedHashString(f.first)];
```

I did not do anything thing with the global variables in 2024.
The work was resumed in July 2024.
I moved `TarWriter`, `SymbolAux`, `Out`, `ElfSym`, `outputSections`, etc into `Ctx`.

```cpp
struct Ctx {
  Config arg;
  LinkerDriver driver;
  LinkerScript *script;
  std::unique_ptr<TargetInfo> target;
  ...
};
```

The `config` variable, used to store command-line options, was pervasive throughout lld/ELF.
To enhance code clarity and maintainability, I renamed it to `ctx.arg` (mold naming).

I've removed other instances of static storage variables throught lld/ELF, e.g.

* [static member `LinkerDriver::nextGroupId`](https://github.com/llvm/llvm-project/commit/f2b01338584c90f48dba1a937bf5b1da8dcedbd5)
* [static member `SharedFile::vernauxNum`](https://github.com/llvm/llvm-project/commit/33ff9e43b4c5bdc3da31c6b11ad51d35a69bec5f)
* [`sectionMap` in `lld/ELF/Arch/ARM.cpp`](https://github.com/llvm/llvm-project/commit/4092c0deef466e5b96a221e4066a78ae72efa7af)

## Passing `Ctx &ctx` as parameters

The subsequent phase involved adding `Ctx &ctx` as a parameter to numerous functions and classes, gradually eliminating references to the global `ctx`.

I incorporated `Ctx &ctx` as a member variable to a few classes (e.g. `SyntheticSection`, `OutputSection`) to minimize the modifications to member functions.
This approach was not suitable for `Symbol` and `InputSection`, since even a single word could increase memory consumption significantly.

```cpp
// Writer.cpp
template <class ELFT> class Writer {
public:
  LLVM_ELF_IMPORT_TYPES_ELFT(ELFT)

  Writer(Ctx &ctx) : ctx(ctx), buffer(ctx.e.outputBuffer) {}
...

template <class ELFT> void elf::writeResult(Ctx &ctx) {
  Writer<ELFT>(ctx).run();
}
...

bool elf::includeInSymtab(Ctx &ctx, const Symbol &b) {
  if (auto *d = dyn_cast<Defined>(&b)) {
    // Always include absolute symbols.
    SectionBase *sec = d->section;
    if (!sec)
      return true;
    assert(sec->isLive());

    if (auto *s = dyn_cast<MergeInputSection>(sec))
      return s->getSectionPiece(d->value).live;
    return true;
  }
  return b.used || !ctx.arg.gcSections;
}
```

## Eliminating the global `ctx` variable

Once the global `ctx` variable's reference count reached zero, it was time to remove it entirely.
I implemented the change on November 16, 2024.

```patch
diff --git a/lld/ELF/Config.h b/lld/ELF/Config.h
index 72feeb9d49cb..a9b7a98e5b54 100644
--- a/lld/ELF/Config.h
+++ b/lld/ELF/Config.h
@@ -539,4 +539,2 @@ struct InStruct {
   std::unique_ptr<SymtabShndxSection> symTabShndx;
-
-  void reset();
 };
@@ -664,3 +662,2 @@ struct Ctx {
   Ctx();
-  void reset();

@@ -671,4 +668,2 @@ struct Ctx {

-LLVM_LIBRARY_VISIBILITY extern Ctx ctx;
-
 // The first two elements of versionDefinitions represent VER_NDX_LOCAL and
diff --git a/lld/ELF/Driver.cpp b/lld/ELF/Driver.cpp
index 334dfc0e3ba1..631051c27381 100644
--- a/lld/ELF/Driver.cpp
+++ b/lld/ELF/Driver.cpp
@@ -81,4 +81,2 @@ using namespace lld::elf;

-Ctx elf::ctx;
-
 static void setConfigs(Ctx &ctx, opt::InputArgList &args);
@@ -165,2 +114,3 @@ bool link(ArrayRef<const char *> args, llvm::raw_ostream &stdoutOS,
           llvm::raw_ostream &stderrOS, bool exitEarly, bool disableOutput) {
+  Ctx ctx;
   // This driver-specific context will be freed later by unsafeLldMain().
@@ -169,7 +119,2 @@ bool link(ArrayRef<const char *> args, llvm::raw_ostream &stdoutOS,
   context->e.initialize(stdoutOS, stderrOS, exitEarly, disableOutput);
-  context->e.cleanupCallback = []() {
-    Ctx &ctx = elf::ctx;
-    ctx.reset();
-    ctx.partitions.emplace_back(ctx);
-  };
   context->e.logName = args::getFilenameWithoutExe(args[0]);
```

Prior to this modification, the cleanupCallback function was essential for resetting the global ctx when lld::elf::link was called multiple times. 

Previously, `cleanupCallback` was essential for resetting the global `ctx` when `lld::elf::link` was invoked multiple times.
With the removal of the global variable, this callback is no longer necessary. We can now rely on the constructor to initialize `Ctx` and avoid the need for a `reset` function.

## Removing global state from `lld/Common`

While significant progress has been made to `lld/ELF`, `lld/Common` needs a lot of work as well.
A lot of shared utility code (diagnostics, bump allocator) utilizes the global `lld::context()`.

```cpp
/// Returns the default error handler.
ErrorHandler &errorHandler();

void error(const Twine &msg);
void error(const Twine &msg, ErrorTag tag, ArrayRef<StringRef> args);
[[noreturn]] void fatal(const Twine &msg);
void log(const Twine &msg);
void message(const Twine &msg, llvm::raw_ostream &s = outs());
void warn(const Twine &msg);
uint64_t errorCount();
```

Although thread-local variables are an option, worker threads spawned by `llvm/lib/Support/Parallel.cpp` don't inherit their values from the main thread.
Given our direct access to `Ctx &ctx`, we can leverage context-aware APIs as replacements.

<https://github.com/llvm/llvm-project/pull/112319> introduced context-aware diagnostic utilities:

* `log("xxx")` => `Log(ctx) << "xxx"`
* `message("xxx")` => `Msg(ctx) << "xxx"`
* `warn("xxx")` => `Warn(ctx) << "xxx"`
* `errorOrWarn(toString(f) + "xxx")` => `Err(ctx) << f << "xxx"`
* `error(toString(f) + "xxx")` => `ErrAlways(ctx) << f << "xxx"`
* `fatal("xxx")` => `Fatal(ctx) << "xxx"`

As of Nov 16, 2024, I have eliminated `log/warn/error/fatal` from lld/ELF.

The underlying functions `lld::ErrorHandler::fatal`, and `lld::ErrorHandler::error` when the error limit is hit and `exitEarly` is true, call `exitLld(1)`.

This transformation eliminates a lot of code size overhead due to `llvm::Twine`.
Even in the simplest `Twine(123)` case, the generated code needs a stack object to hold the value and a Twine kind.

`lld::make` from [`lld/include/lld/Common/Memory.h`](https://github.com/llvm/llvm-project/blob/main/lld/include/lld/Common/Memory.h) is an allocation function that uses the global context.
When the ownership is clear, `std::make_unique` might be a better choice.

Guideline:

* Avoid `lld::saver`
* Avoid `void message(const Twine &msg, llvm::raw_ostream &s = outs());`, which utilizes `lld::outs()`
* Avoid `lld::make` from [`lld/include/lld/Common/Memory.h`](https://github.com/llvm/llvm-project/blob/main/lld/include/lld/Common/Memory.h)
* Avoid fatal error in a half-initialized object, e.g. fatal error in a base class constructor (`ELFFileBase::init`) ([[LLD][COFF] When using LLD-as-a-library, always prevent re-entrance on failures](https://reviews.llvm.org/D88348))

## Global state in LLVM

LTO link jobs utilize LLVM. Understanding its global state is crucial.

While LLVM allows for multiple `LLVMContext` instances to be allocated and used concurrently, it's important to note that these instances share certain global states, such as `cl::opt` and `ManagedStatic`.
Specifically, it's not possible to run two concurrent LLVM compilations (including LTO link jobs) with distinct sets of `cl::opt` option values.
To link with distinct `cl::opt` values, even after removing LLD's global state, you'll need to spawn a new LLD process.

Any proposal that moves away from global state seems to complicate `cl::opt` usage, making it impractical.

LLD also utilizes functions from [`llvm/Support/Parallel.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Support/Parallel.h) for parallelism.
These functions rely on global state like `getDefaultExecutor` and `llvm::parallel::strategy`.
Ongoing work by Alexandre Ganea aims to make these functions context-aware.
(It's nice to meet you in person in LLVM Developers' Meeting last month)

## Supported library usage scenarios

You can repeatedly call `lld::lldMain` from [`lld/Common/Driver.h`](https://github.com/llvm/llvm-project/blob/main/lld/include/lld/Common/Driver.h).
If `fatal` has been invoked, it will not be safe to call `lld::lldMain` again in certain rare scenarios.
Running `lld::lldMain` concurrently in two threads is not supported.

The command `LLD_IN_TEST=3 lld-link ...` runs the link process three times, but only the final invocation outputs diagnostics to stdout/stderr.
`lld/test/lit.cfg.py` has configured the COFF port to run tests twice ([[lld] Add test suite mode for running LLD main twice](https://reviews.llvm.org/D112898)).
Other ports need work to make this mode work.
