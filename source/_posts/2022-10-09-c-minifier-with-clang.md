layout: post
title: C minifier with Clang
author: MaskRay
tags: [clang]
---

I recently revamped [Competitive programming in Nim](/blog/2021-09-26-competitive-programming-in-nim).
In short, I can create a C amalgamation from a Nim program and submit the C source code to various competitive programming websites.

Then I use a Clang based tool to shorten the C source code. It does two things:

* Shorten function, variables, and type names
* Use the `clangFormat` library to remove some whitespace

For the first step, the tool uses a derived `ASTFrontendAction` to traverse the AST twice, one for collecting function/var/type names and the other for renaming.
Building `clang::CompilerInstance` from command lines needs some boilerplate.
An alternative is to use `clang::tooling::CommonOptionsParser` and `clang::tooling::ClangTool`.

```cpp
/*
 * MiniASTConsumer collects identifiers in `used` and rename candidates (in the main file) in `d2name`.
 * MiniASTConsumer iterates over `d2name` and assigns new names.
 * Renamer creates clang::tooling::Replacement instances.
 * HandleTranslationUnit calls clang::tooling::applyAllReplacements.
 */

#include <clang/AST/ASTConsumer.h>
#include <clang/AST/Decl.h>
#include <clang/AST/RecursiveASTVisitor.h>
#include <clang/Basic/FileManager.h>
#include <clang/Basic/LangOptions.h>
#include <clang/Basic/SourceManager.h>
#include <clang/Basic/TargetInfo.h>
#include <clang/Driver/Action.h>
#include <clang/Driver/Compilation.h>
#include <clang/Driver/Driver.h>
#include <clang/Driver/Tool.h>
#include <clang/Format/Format.h>
#include <clang/Frontend/CompilerInstance.h>
#include <clang/Frontend/FrontendAction.h>
#include <clang/Lex/Lexer.h>
#include <clang/Lex/PreprocessorOptions.h>
#include <clang/Tooling/Core/Replacement.h>
#include <llvm/ADT/CachedHashString.h>
#include <llvm/ADT/DenseSet.h>
#include <llvm/ADT/MapVector.h>
#include <llvm/ADT/STLExtras.h>
#include <llvm/Support/Host.h>
#include <llvm/Support/Path.h>
#include <llvm/Support/raw_ostream.h>

#include <memory>
#include <vector>

#include <assert.h>
#include <err.h>
#include <unistd.h>

using namespace clang;
using namespace llvm;

namespace {
std::unique_ptr<CompilerInvocation> buildCompilerInvocation(ArrayRef<const char *> args) {
  IntrusiveRefCntPtr<DiagnosticsEngine> diags(
      CompilerInstance::createDiagnostics(new DiagnosticOptions, new IgnoringDiagConsumer, true));

  driver::Driver d(args[0], llvm::sys::getDefaultTargetTriple(), *diags, "cminify", llvm::vfs::getRealFileSystem());
  d.setCheckInputsExist(false);
  std::unique_ptr<driver::Compilation> comp(d.BuildCompilation(args));
  if (!comp)
    return nullptr;
  const driver::JobList &jobs = comp->getJobs();
  if (jobs.size() != 1 || !isa<driver::Command>(*jobs.begin()))
    return nullptr;

  const driver::Command &cmd = cast<driver::Command>(*jobs.begin());
  if (StringRef(cmd.getCreator().getName()) != "clang")
    return nullptr;
  const llvm::opt::ArgStringList &cc_args = cmd.getArguments();
  auto ci = std::make_unique<CompilerInvocation>();
  if (!CompilerInvocation::CreateFromArgs(*ci, cc_args, *diags))
    return nullptr;

  ci->getDiagnosticOpts().IgnoreWarnings = true;
  ci->getFrontendOpts().DisableFree = false;
  return ci;
}

SmallVector<StringRef, 0> ignores;
MapVector<Decl *, std::string> d2name;
DenseSet<CachedHashStringRef> used;
std::string newCode;

struct Collector : RecursiveASTVisitor<Collector> {
  SourceManager &sm;

  Collector(ASTContext &ctx) : sm(ctx.getSourceManager()) {}
  bool VisitFunctionDecl(FunctionDecl *fd) {
    if (fd->isOverloadedOperator() || !fd->getIdentifier())
      return true;
    used.insert(CachedHashStringRef(fd->getName()));
    if (!fd->isDefined())
      return true;
    std::string name = fd->getNameAsString();
    if (sm.isWrittenInMainFile(fd->getLocation())) {
      if (!is_contained(ignores, name))
        d2name[fd->getCanonicalDecl()] = "_f";
      for (ParmVarDecl *param : fd->parameters())
        VisitVarDecl(param);
    }
    return true;
  }
  bool VisitVarDecl(VarDecl *vd) {
    if (!vd->getIdentifier())
      return true;
    used.insert(CachedHashStringRef(vd->getName()));
    auto kind = vd->isThisDeclarationADefinition();
    if (kind != VarDecl::Definition || !sm.isWrittenInMainFile(vd->getLocation()))
      return true;
    d2name[vd->getCanonicalDecl()] = "_v";
    return true;
  }

  bool VisitTagDecl(TagDecl *td) {
    used.insert(CachedHashStringRef(td->getName()));
    if (!td->isThisDeclarationADefinition() || !sm.isWrittenInMainFile(td->getLocation()))
      return true;
    d2name[td->getCanonicalDecl()] = "_t";
    return true;
  }
  bool VisitTypedefNameDecl(TypedefNameDecl *d) {
    if (d->isTransparentTag() || !sm.isWrittenInMainFile(d->getLocation()))
      return true;
    d2name[d->getCanonicalDecl()] = "_t";
    return true;
  }
};

struct Renamer : RecursiveASTVisitor<Renamer> {
  SourceManager &sm;
  tooling::Replacements &reps;

  Renamer(ASTContext &ctx, tooling::Replacements &reps) : sm(ctx.getSourceManager()), reps(reps) {}
  void replace(CharSourceRange csr, StringRef newText) { cantFail(reps.add(tooling::Replacement(sm, csr, newText))); }

  bool VisitFunctionDecl(FunctionDecl *fd) {
    auto *canon = fd->getCanonicalDecl();
    auto it = d2name.find(canon);
    if (it != d2name.end())
      replace(CharSourceRange::getTokenRange(fd->getLocation()), it->second);
    return true;
  }
  bool VisitVarDecl(VarDecl *vd) {
    auto *canon = vd->getCanonicalDecl();
    auto it = d2name.find(canon);
    if (it != d2name.end())
      replace(CharSourceRange::getTokenRange(vd->getLocation()), it->second);
    return true;
  }
  bool VisitDeclRefExpr(DeclRefExpr *dre) {
    Decl *d = dre->getDecl();
    if (!(isa<FunctionDecl>(d) || isa<VarDecl>(d)))
      return true;
    auto it = d2name.find(d->getCanonicalDecl());
    if (it != d2name.end())
      replace(CharSourceRange::getTokenRange(SourceRange(dre->getBeginLoc(), dre->getEndLoc())), it->second);
    return true;
  }

  bool VisitTagDecl(TagDecl *d) {
    auto *canon = d->getCanonicalDecl();
    if (d->getTypedefNameForAnonDecl())
      return true;
    if (auto it = d2name.find(canon); it != d2name.end())
      replace(CharSourceRange::getTokenRange(d->getLocation()), it->second);
    return true;
  }
  bool VisitTagTypeLoc(TagTypeLoc tl) {
    TagDecl *td = tl.getDecl()->getCanonicalDecl();
    if (td->getTypedefNameForAnonDecl())
      return true;
    if (auto it = d2name.find(td); it != d2name.end())
      replace(CharSourceRange::getTokenRange(tl.getNameLoc()), it->second);
    return true;
  }
  bool VisitTypedefNameDecl(TypedefNameDecl *d) {
    if (auto it = d2name.find(d->getCanonicalDecl()); it != d2name.end())
      replace(CharSourceRange::getTokenRange(d->getLocation()), it->second);
    return true;
  }
  bool VisitTypedefTypeLoc(TypedefTypeLoc tl) {
    TypedefNameDecl *td = tl.getTypedefNameDecl();
    if (auto it = d2name.find(td); it != d2name.end())
      replace(CharSourceRange::getTokenRange(tl.getNameLoc()), it->second);
    return true;
  }
};

struct MiniASTConsumer : ASTConsumer {
  ASTContext *ctx;
  int n_fn = 0, n_var = 0, n_type = 0;

  void Initialize(ASTContext &ctx) override { this->ctx = &ctx; }
  static std::string getName(StringRef prefix, int &id) {
    static const char digits[] = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    std::string newName;
    for (;;) {
      newName = std::string(1, prefix[id % prefix.size()]);
      if (int i = id / prefix.size())
        while (newName += digits[i % 62], i /= 62);
      id++;
      if (!used.contains(CachedHashStringRef(newName))) break;
    }
    return newName;
  }
  bool HandleTopLevelDecl(DeclGroupRef dgr) override {
    for (auto s : {"j0", "j1", "jn", "j0f", "j1f", "jnf", "j0l", "j1l", "jnl"})
      used.insert(CachedHashStringRef(s));
    for (auto s : {"y0", "y1", "yn", "y0f", "y1f", "ynf", "y0l", "y1l", "ynl"})
      used.insert(CachedHashStringRef(s));

    Collector c(*ctx);
    for (Decl *d : dgr)
      c.TraverseDecl(d);
    for (auto &[d, name] : d2name) {
      if (name == "_f")
        name = getName("abcdefghijklm", n_fn);
      else if (name == "_v") {
        int old_n_var = n_var;
        auto newName = getName("nopqrstuvwxyz", n_var);
        if (newName.size() < static_cast<VarDecl *>(d)->getName().size())
          name = newName;
        else {
          name = static_cast<VarDecl *>(d)->getName();
          n_var = old_n_var;
        }
      } else if (name == "_t")
        name = getName("ABCDEFGHIJKLMNOPQRSTUVWXYZ", n_type);
    }
    return true;
  }
  void HandleTranslationUnit(ASTContext &ctx) override {
    tooling::Replacements reps;
    Renamer c(ctx, reps);
    c.TraverseDecl(ctx.getTranslationUnitDecl());

    auto &sm = ctx.getSourceManager();
    StringRef code = sm.getBufferData(sm.getMainFileID());
    auto res = tooling::applyAllReplacements(code, reps);
    if (!res)
      errx(2, "failed to apply replacements: %s", toString(res.takeError()).c_str());
    newCode = *res;
  }
};

struct MiniAction : ASTFrontendAction {
  std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &ci,
                                                 StringRef inFile) override {
    return std::make_unique<MiniASTConsumer>();
  }
};

void reformat() {
  auto buf = MemoryBuffer::getMemBuffer(newCode, "", true);
  format::FormatStyle style = cantFail(format::getStyle("LLVM", "-", "LLVM", newCode, nullptr));
  style.ColumnLimit = 9999;
  style.IndentWidth = 0;
  style.ContinuationIndentWidth = 0;
  style.SpaceBeforeAssignmentOperators = false;
  style.SpaceBeforeParens = format::FormatStyle::SBPO_Never;
  style.AlignEscapedNewlines = format::FormatStyle::ENAS_DontAlign;

  format::FormattingAttemptStatus status;
  std::vector<tooling::Range> ranges{{0, unsigned(newCode.size())}};
  tooling::Replacements reps = format::reformat(style, newCode, ranges, "-", &status);
  auto res = tooling::applyAllReplacements(newCode, reps);
  if (!res)
    errx(2, "failed to apply replacements: %s", toString(res.takeError()).c_str());
  newCode = *res;
}
}

int main(int argc, char *argv[]) {
  std::vector<const char *> args{argv[0], "-fsyntax-only"};
  bool inplace = false;
  const char *outfile = "/dev/stdout";
  const char usage[] = R"(Usage: %s [-i] [-f fun]... a.c

Options:
-i      edit a.c in place\n)";
  for (int i = 1; i < argc; i++) {
    StringRef opt(argv[i]);
    if (opt[0] != '-')
      args.push_back(argv[i]);
    else if (opt == "-h") {
      fputs(usage, stdout);
      return 0;
    } else if (opt == "-i")
      inplace = true;
    else if (opt == "-f" && i + 1 < argc)
      ignores.push_back(argv[++i]);
    else if (opt == "-o" && i + 1 < argc)
      outfile = argv[++i];
    else {
      fputs(usage, stderr);
      return 1;
    }
  }
  ignores.push_back("main");

  auto ci = buildCompilerInvocation(args);
  if (!ci)
    errx(1, "failed to build CompilerInvocation");

  auto inst = std::make_unique<CompilerInstance>(std::make_shared<PCHContainerOperations>());
  IgnoringDiagConsumer dc;
  inst->setInvocation(std::move(ci));
  inst->createDiagnostics(&dc, false);
  inst->getDiagnostics().setIgnoreAllWarnings(true);
  inst->setTarget(TargetInfo::CreateTargetInfo(inst->getDiagnostics(), inst->getInvocation().TargetOpts));
  if (!inst->hasTarget())
    errx(1, "hasTarget returns false");
  inst->createFileManager(llvm::vfs::getRealFileSystem());
  inst->setSourceManager(new SourceManager(inst->getDiagnostics(), inst->getFileManager(), true));

  MiniAction action;
  if (!action.BeginSourceFile(*inst, inst->getFrontendOpts().Inputs[0]))
    errx(2, "failed to parse");
  if (Error e = action.Execute())
    errx(2, "failed to execute");
  action.EndSourceFile();
  reformat();

  std::error_code ec;
  raw_fd_ostream(inplace ? inst->getFrontendOpts().Inputs[0].getFile() : outfile, ec, sys::fs::OF_None) << newCode;
}
```

`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.14)
project(cminify LANGUAGES C CXX)

add_executable(cminify "")
set(DEFAULT_CMAKE_BUILD_TYPE Release)
set_property(TARGET cminify PROPERTY CXX_STANDARD 17)
set_property(TARGET cminify PROPERTY CXX_STANDARD_REQUIRED ON)
set_property(TARGET cminify PROPERTY CXX_EXTENSIONS OFF)

find_package(Clang REQUIRED)

if(CLANG_LINK_CLANG_DYLIB)
  target_link_libraries(cminify PRIVATE clang-cpp)
else()
  target_link_libraries(cminify PRIVATE
    clangIndex
    clangFormat
    clangTooling
    clangToolingInclusions
    clangToolingCore
    clangFrontend
    clangParse
    clangSerialization
    clangSema
    clangAST
    clangLex
    clangDriver
    clangBasic
  )
endif()

if(LLVM_LINK_LLVM_DYLIB)
  target_link_libraries(cminify PRIVATE LLVM)
else()
  target_link_libraries(cminify PRIVATE LLVMOption LLVMSupport)
endif()

if(NOT LLVM_ENABLE_RTTI)
  # releases.llvm.org libraries are compiled with -fno-rtti
  # The mismatch between lib{clang,LLVM}* and cminify can make libstdc++ std::make_shared return nullptr
  # _Sp_counted_ptr_inplace::_M_get_deleter
  if(MSVC)
    target_compile_options(cminify PRIVATE /GR-)
  else()
    target_compile_options(cminify PRIVATE -fno-rtti)
  endif()
endif()

target_sources(cminify PRIVATE main.cc)

foreach(include_dir ${LLVM_INCLUDE_DIRS} ${CLANG_INCLUDE_DIRS})
  get_filename_component(include_dir_realpath ${include_dir} REALPATH)
  # Don't add as SYSTEM if they are in CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES.
  # It would reorder the system search paths and cause issues with libstdc++'s
  # use of #include_next. See https://github.com/MaskRay/ccls/pull/417
  if(NOT "${include_dir_realpath}" IN_LIST CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES)
    target_include_directories(cminify SYSTEM PRIVATE ${include_dir})
  endif()
endforeach()

install(TARGETS cminify RUNTIME DESTINATION bin)
```

Define `LLVM` as the llvm-project repository and `LLVMOUT` as the build directory (make sure you have at least built these targets: `ninja clang clangFormat clangIndex clangTooling`).
```sh
cmake -GNinja -S. -Bout/release -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$LLVMOUT;$LLVMOUT/tools/clang;$LLVM/llvm;$LLVM/clang"
ninja -C out/release
```

If LLVM and Clang's CMake, library, and header files are installed in well-known locations, then `-DCMAKE_PREFIX_PATH` can be omitted.

It's certainly not straightforward to find all these APIs. I mainly use ccls as a reference which was inspired by `clangIndex`.
For writing this tool, I read a bit code of `clang-rename`, `clang-format`, and C-Reduce `clang_delta`.
C-Reduce provides [`clang_delta/RenameFun.cpp`](https://github.com/csmith-project/creduce/blob/master/clang_delta/RenameFun.cpp) and two other passes (RenameVar, RenameParam) which do similar stuff.
Its code was a bit old now as it was written based on a Clang in circa 2012.

Let's see an example. Unfortunately I don't find clangFormat options removing whitespace after `=` and `,`.
That can perhaps be done by a post-processing string substitution tool without introducing too much risk.

```text
% cat test/a.c
#include <stdint.h>
#include <string.h>

#pragma GCC diagnostic ignored "-Wpragmas"
static float foo(int, int);
static float foo(int aaa, int bbb) { aaa = 3; bbb = 5; return 1.0f; }

struct NimStrPayload;
typedef struct NimStrPayload NimStrPayload;
struct NimStringV2;
typedef struct NimStringV2 NimStringV2;

struct NimStrPayload { int64_t cap; char data[]; };
struct NimStringV2 { int64_t cap; NimStrPayload *p; };

#define XX NimStringV2

float goo() {
  int u, v, w, x, y, z;
  int s1, t1, u1, v1, w1, x1, y1, z1;
  NimStringV2 s;
  XX t;
  return 1.0f;
}

int main() {
  char a[10];
  memset(a, 0, 10);
  float _ = foo(3, 5) + goo();
}
% out/release/cminify test/a.c
#include <stdint.h>
#include <string.h>

#pragma GCC diagnostic ignored "-Wpragmas"
static float a(int, int);
static float a(int k, int l) {
k= 3;
l= 5;
return 1.0f;
}

struct C;
typedef struct C A;
struct D;
typedef struct D B;

struct C {
int64_t cap;
char data[];
};
struct D {
int64_t cap;
A *p;
};

#define XX B

float b() {
int u, v, w, x, y, z;
int m, n, o, p, q, r, y1, z1;
B s;
XX t;
return 1.0f;
}

int main() {
char a[10];
memset(a, 0, 10);
float _= a(3, 5) + b();
}
```
