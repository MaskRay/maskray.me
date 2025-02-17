---
layout: post
title: Compiling C++ with the Clang API
author: MaskRay
tags: [clang,llvm]
---

This post describes how to compile a single C++ source file to an object file with the Clang API.
Here is the code. It behaves like a simplified `clang` executable that handles `-c` and `-S`.

<!-- more -->

```sh
cat > main.cc <<eof
```

```cpp
#include <clang/CodeGen/CodeGenAction.h> // EmitObjAction
#include <clang/Driver/Compilation.h>
#include <clang/Driver/Driver.h>
#include <clang/Frontend/CompilerInstance.h>
#include <clang/Frontend/FrontendOptions.h>
#include <llvm/Config/llvm-config.h>   // LLVM_VERSION_MAJOR
#include <llvm/Support/TargetSelect.h> // LLVMInitialize*
#include <llvm/Support/VirtualFileSystem.h>

using namespace clang;

constexpr llvm::StringRef kTargetTriple = "x86_64-unknown-linux-gnu";

namespace {
struct DiagsSaver : DiagnosticConsumer {
  std::string message;
  llvm::raw_string_ostream os{message};

  void HandleDiagnostic(DiagnosticsEngine::Level diagLevel, const Diagnostic &info) override {
    DiagnosticConsumer::HandleDiagnostic(diagLevel, info);
    const char *level;
    switch (diagLevel) {
    default:
      return;
    case DiagnosticsEngine::Note:
      level = "note";
      break;
    case DiagnosticsEngine::Warning:
      level = "warning";
      break;
    case DiagnosticsEngine::Error:
    case DiagnosticsEngine::Fatal:
      level = "error";
      break;
    }

    llvm::SmallString<256> msg;
    info.FormatDiagnostic(msg);
    auto &sm = info.getSourceManager();
    auto loc = info.getLocation();
    auto fileLoc = sm.getFileLoc(loc);
    os << sm.getFilename(fileLoc) << ':' << sm.getSpellingLineNumber(fileLoc)
       << ':' << sm.getSpellingColumnNumber(fileLoc) << ": " << level << ": "
       << msg << '\n';
    if (loc.isMacroID()) {
      loc = sm.getSpellingLoc(loc);
      os << sm.getFilename(loc) << ':' << sm.getSpellingLineNumber(loc) << ':'
         << sm.getSpellingColumnNumber(loc) << ": note: expanded from macro\n";
    }
  }
};
}

static std::pair<bool, std::string> compile(int argc, char *argv[]) {
  auto fs = llvm::vfs::getRealFileSystem();
  DiagsSaver dc;
  std::vector<const char *> args{"clang"};
  args.insert(args.end(), argv + 1, argv + argc);
  auto diags = CompilerInstance::createDiagnostics(
#if LLVM_VERSION_MAJOR >= 20
      *fs,
#endif
      new DiagnosticOptions, &dc, false);
  driver::Driver d(args[0], kTargetTriple, *diags, "cc", fs);
  d.setCheckInputsExist(false);
  std::unique_ptr<driver::Compilation> comp(d.BuildCompilation(args));
  const auto &jobs = comp->getJobs();
  if (jobs.size() != 1)
    return {false, "only support one job"};
  const llvm::opt::ArgStringList &ccArgs = jobs.begin()->getArguments();

  auto invoc = std::make_unique<CompilerInvocation>();
  CompilerInvocation::CreateFromArgs(*invoc, ccArgs, *diags);
  auto ci = std::make_unique<CompilerInstance>();
  ci->setInvocation(std::move(invoc));
  ci->createDiagnostics(*fs, &dc, false);
  // Disable CompilerInstance::printDiagnosticStats, which might display "2 warnings generated."
  ci->getDiagnostics().getDiagnosticOptions().ShowCarets = false;
  ci->createFileManager(fs);
  ci->createSourceManager(ci->getFileManager());

  // Clang calls BuryPointer on the internal AST and CodeGen-related elements like TargetMachine.
  // This will cause memory leaks if `compile` is executed many times.
  ci->getCodeGenOpts().DisableFree = false;
  ci->getFrontendOpts().DisableFree = false;

  LLVMInitializeX86AsmParser();
  LLVMInitializeX86AsmPrinter();
  LLVMInitializeX86Target();
  LLVMInitializeX86TargetInfo();
  LLVMInitializeX86TargetMC();

  switch (ci->getFrontendOpts().ProgramAction) {
  case frontend::ActionKind::EmitObj: {
    EmitObjAction action;
    ci->ExecuteAction(action);
  } break;
  case frontend::ActionKind::EmitAssembly: {
    EmitAssemblyAction action;
    ci->ExecuteAction(action);
  } break;
  default:
    return {false, "unhandled action"};
  }
  return {true, std::move(dc.message)};
}

int main(int argc, char *argv[]) {
  auto [ok, err] = compile(argc, argv);
  llvm::errs() << err;
}
```

```
eof
```

## Building the code with CMake

Let's write a `CMakeLists.txt` that links against the needed Clang and LLVM libraries.

```sh
cat > CMakeLists.txt <<eof
project(cc)
cmake_minimum_required(VERSION 3.16)
find_package(LLVM REQUIRED CONFIG)
find_package(Clang REQUIRED CONFIG)

include_directories(${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS})
add_executable(cc main.cc)

if(NOT LLVM_ENABLE_RTTI)
  target_compile_options(cc PRIVATE -fno-rtti)
endif()

if(CLANG_LINK_CLANG_DYLIB)
  target_link_libraries(cc PRIVATE clang-cpp)
else()
  target_link_libraries(cc PRIVATE
    clangAST
    clangBasic
    clangCodeGen
    clangDriver
    clangFrontend
    clangLex
    clangParse
    clangSema
  )
endif()

if(LLVM_LINK_LLVM_DYLIB)
  target_link_libraries(cc PRIVATE LLVM)
else()
  target_link_libraries(cc PRIVATE LLVMOption LLVMSupport LLVMTarget
    LLVMX86AsmParser LLVMX86CodeGen LLVMX86Desc LLVMX86Info)
endif()
eof
```

We need an LLVM and Clang installation that provides both `lib/cmake/llvm/LLVMConfig.cmake` and `lib/cmake/clang/ClangConfig.cmake`.
You can grab these from system packages (dev versions may be required) or build LLVM yourself-I'll skip the detailed steps here. For a DIY build, use:

```
# cmake ... -DLLVM_ENABLE_PROJECTS='clang'

ninja -C out/stable clang-cmake-exports clang
```

No `install` step is needed. Next, create a build directory with the CMake configuration above:

```sh
cmake -S. -Bout/debug -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER=$HOME/Stable/bin/clang++ -DCMAKE_PREFIX_PATH="$HOME/llvm/out/stable"
ninja -C out/debug
```

I've set a prebuilt Clang as `CMAKE_CXX_COMPILER`-just a habit of mine.
llvm-project isn't guaranteed to build warning-free with GCC, since GCC `-Wall -Wextra` has many false positives and LLVM developers avoid cluttering the codebase.

```
% echo 'void f() {}' > a.cc
% out/debug/cc -S a.cc && head -n 5 a.s
        .file   "a.cc"
        .text
        .globl  _Z1fv                           # -- Begin function _Z1fv
        .p2align        4
        .type   _Z1fv,@function
% out/debug/cc -c a.cc && ls a.o
a.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

## Anonymous files

The input source file and the output ELF file are stored in the filesystem.
We could create a temporary file and delete it with a RAII class `llvm::FileRemover`:

```cpp
std::error_code ec = llvm::sys::fs::createTemporaryFile("clang", "cc", fdIn, tempPath);
llvm::raw_fd_stream osIn(fdIn, /*ShouldClose=*/true);
llvm::FileRemover remover(tempPath);
```

On Linux, we could utilzie `memfd_create` to create a file in RAM with a volatile backing storage.

```cpp
int fdIn = memfd_create("input", MFD_CLOEXEC);
if (fdIn < 0)
  return {"", "failed to create input memfd"};
int fdOut = memfd_create("output", MFD_CLOEXEC);
if (fdOut < 0) {
  close(fdIn);
  return {"", "failed to create output memfd"};
}

std::string pathIn = "/proc/self/fd/" + std::to_string(fdIn);
std::string pathOut = "/proc/self/fd/" + std::to_string(fdOut);

// clang -c -xc++ /proc/self/fd/3 -o /proc/self/fd/4
```

## `LLVMInitialize*`

To generate x86 code, we need a few LLVM X86 libraries defined by `llvm/lib/Target/X86/**/CMakeLists.txt` files.

```cpp
LLVMInitializeX86AsmPrinter();
LLVMInitializeX86Target();
LLVMInitializeX86TargetInfo();
LLVMInitializeX86TargetMC();
```

If inline assembly is used, we will also need the AsmParser library:

```cpp
LLVMInitializeX86AsmParser();
```

We could also call `LLVMInitializeAll*` functions instead, which initialize all supported targets (build-time `LLVM_TARGETS_TO_BUILD`).

Here are some notes about the LLVMX86 libraries:

* LLVMX86Info: `llvm/lib/Target/X86/TargetInfo/`
* LLVMX86Desc: `llvm/lib/Target/X86/MCTargetDesc/` (depends on LLVMX86Info)
* LLVMX86AsmParser: `llvm/lib/Target/X86/AsmParser` (depends on LLVMX86Info and LLVMX86Desc)
* LLVMX86CodeGen: `llvm/lib/Target/X86/` (depends on LLVMX86Info and LLVMX86Desc)

## `EmitAssembly` and `EmitObj`

The code supports two frontend actions, `EmitAssembly` (`-S`) and `EmitObj` (`-c`).

You could also utilize the API in `clang/include/clang/FrontendTool/Utils.h`, but that would pull in another library `clangFrontendTool` (different from `clangFrontend`).

## Diagnostics

The diagnostics system is quite complex. We have `DiagnosticConsumer`, `DiagnosticsEngine`, and `DiagnosticOptions`.

```
DiagnosticsEngine
├─ DiagnosticIDs (defines diagnostics)
├─ SourceManager (provides locations)
├─ DiagnosticOptions (configures output)
└─ DiagnosticConsumer (handles output)
   └─ Diagnostic (individual message)
```

We define a simple `DiagnosticConsumer` that handles notes, warnings, errors, and fatal errors.
When macro expansion comes into play, we report two key locations:

* The physical location (`fileLoc`), where the expanded token triggers an issue-matching Clang's error line, and
* The spelling location within the macro's replacement list (`sm.getSpellingLoc(loc)`).

Although Clang also highlights intermediate locations for chained expansions, our simple approach offers a solid approximation.

```
% cat a.h
#define FOO(x) x + 1
% cat a.cc
#include "a.h"
#define BAR FOO
void f() {
  int y = BAR("abc");
}
% out/debug/cc -c -Wall a.cc
a.cc:4:11: warning: adding 'int' to a string does not append to the string
./a.h:1:18: note: expanded from macro
a.cc:4:11: note: use array indexing to silence this warning
./a.h:1:18: note: expanded from macro
a.cc:4:7: error: cannot initialize a variable of type 'int' with an rvalue of type 'const char *'
% clang -c -Wall a.cc
a.cc:4:11: warning: adding 'int' to a string does not append to the string [-Wstring-plus-int]
    4 |   int y = BAR("abc");
      |           ^~~~~~~~~~
a.cc:2:13: note: expanded from macro 'BAR'
    2 | #define BAR FOO
      |             ^
./a.h:1:18: note: expanded from macro 'FOO'
    1 | #define FOO(x) x + 1
      |                ~~^~~
a.cc:4:11: note: use array indexing to silence this warning
a.cc:2:13: note: expanded from macro 'BAR'
    2 | #define BAR FOO
      |             ^
./a.h:1:18: note: expanded from macro 'FOO'
    1 | #define FOO(x) x + 1
      |                  ^
a.cc:4:7: error: cannot initialize a variable of type 'int' with an rvalue of type 'const char *'
    4 |   int y = BAR("abc");
      |       ^   ~~~~~~~~~~
1 warning and 1 error generated.
```

We call a convenience function `CompilerInstance::ExecuteAction`, which wraps lower-level API like `BeginSource`, `Execute`, and `EndSource`.
However, it will print `1 warning and 1 error generated.` unless we set `ShowCarets` to false.

## `clang::createInvocation`

`clang::createInvocation`, renamed from [`createInvocationFromCommandLine`](https://reviews.llvm.org/D124971) in 2022, combines `clang::Driver::BuildCompilation` and `clang::CompilerInvocation::CreateFromArgs`.
While it saves a few lines for certain tasks, it lacks the flexibility we need for our specific use cases.
