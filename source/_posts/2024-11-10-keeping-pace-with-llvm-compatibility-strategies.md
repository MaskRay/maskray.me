---
layout: post
title: "Keeping pace with LLVM: compatibility strategies"
author: MaskRay
tags: [ccls,llvm]
---

LLVM's C++ API doesn't offer a stability guarantee. This means function signatures can change or be removed between versions, forcing projects to adapt.

On the other hand, LLVM has an extensive API surface. When a library like `llvm/lib/Y` relies functionality from another library, the API is often exported in header files under `llvm/include/llvm/X/`, even if it is not intended to be user-facing.

To be compatible with multiple LLVM versions, many projects rely on `#if` directives based on the `LLVM_VERSION_MAJOR` macro.
This post explores the specific techniques used by ccls to ensure compatibility with LLVM versions 7 to 19.
For the latest release ([ccls 0.20241108](https://github.com/MaskRay/ccls/releases/tag/0.20241108)), support for LLVM versions 7 to 9 has been discontinued.

Given the tight coupling between LLVM and Clang, the `LLVM_VERSION_MAJOR` macro can be used for both version detection. There's no need to check `CLANG_VERSION_MAJOR`.

<!-- more -->

---

## Changed namespaces

In Oct 2018, <https://reviews.llvm.org/D52783> moved the namespace `clang::vfs` to `llvm::vfs`.
To remain compatibility, I renamed `clang::vfs` uses and added a conditional namespace alias:

```cpp
#if LLVM_VERSION_MAJOR < 8
// D52783 Lift VFS from clang to llvm
namespace llvm {
namespace vfs = clang::vfs;
}
#endif
```

## Removed functions

In March 2019, <https://reviews.llvm.org/D59377> removed the member variable `VirtualFileSystem` and removed `setVirtualFileSystem`.
To adapt to this change, ccls employs an `#if`.
```cpp
#if LLVM_VERSION_MAJOR >= 9 // rC357037
  Clang->createFileManager(FS);
#else
  Clang->setVirtualFileSystem(FS);
  Clang->createFileManager();
#endif
```

## Changed function parameters

In April 2020, the LLVM monorepo integrated a new subproject: flang.
flang developers made many changes to clangDriver to reuse it for flang.
<https://reviews.llvm.org/D86089> changed the constructor `clang::driver::Driver`.
I added
```cpp
#if LLVM_VERSION_MAJOR < 12 // llvmorg-12-init-5498-g257b29715bb
  driver::Driver d(args[0], llvm::sys::getDefaultTargetTriple(), *diags, vfs);
#else
  driver::Driver d(args[0], llvm::sys::getDefaultTargetTriple(), *diags, "ccls", vfs);
#endif
```

In November 2020, <https://reviews.llvm.org/D90890> changed an argument of `ComputePreambleBounds` from `const llvm::MemoryBuffer *Buffer` to `const llvm::MemoryBufferRef &Buffer`.
```cpp
std::unique_ptr<llvm::MemoryBuffer> buf =
    llvm::MemoryBuffer::getMemBuffer(content);
#if LLVM_VERSION_MAJOR >= 12 // llvmorg-12-init-11522-g4c55c3b66de
  auto bounds = ComputePreambleBounds(*ci.getLangOpts(), *buf, 0);
#else
  auto bounds = ComputePreambleBounds(*ci.getLangOpts(), buf.get(), 0);
#endif
```

<https://reviews.llvm.org/D91297> made a similar change and I adapted it similarly.

In Jan 2022, <https://reviews.llvm.org/D116317> added a new parameter `bool Braced` to `CodeCompleteConsumer::ProcessOverloadCandidates`.

```cpp
  void ProcessOverloadCandidates(Sema &s, unsigned currentArg,
                                 OverloadCandidate *candidates,
                                 unsigned numCandidates
#if LLVM_VERSION_MAJOR >= 8
                                 ,
                                 SourceLocation openParLoc
#endif
#if LLVM_VERSION_MAJOR >= 14
                                 ,
                                 bool braced
#endif
                                 ) override {
```

In late 2022 and early 2023, there were many changes to migrate from `llvm::Optional` to `std::optional`.

```cpp
#if LLVM_VERSION_MAJOR >= 16 // llvmorg-16-init-12589-ge748db0f7f09
    std::array<std::optional<StringRef>, 3>
#else
    std::array<Optional<StringRef>, 3>
#endif
        redir{StringRef(stdinPath), StringRef(path), StringRef()}; 0 ref
    std::vector<StringRef> args{g_config->compilationDatabaseCommand, root}; 0 ref
    if (sys::ExecuteAndWait(args[0], args, {}, redir, 0, 0, &err_msg) < 0) {
```

In Sep 2023, <https://github.com/llvm/llvm-project/pull/65647> changed `CompilerInvocationRefBase` to `CompilerInvocationBase`.
I duplicated the code with `.`.
```cpp
#if LLVM_VERSION_MAJOR >= 18
  ci->getLangOpts().SpellChecking = false;
  ci->getLangOpts().RecoveryAST = true;
  ci->getLangOpts().RecoveryASTType = true;
#else
  ci->getLangOpts()->SpellChecking = false;
#if LLVM_VERSION_MAJOR >= 11
  ci->getLangOpts()->RecoveryAST = true;
  ci->getLangOpts()->RecoveryASTType = true;
#endif
#endif
```

In April 2024, <https://github.com/llvm/llvm-project/pull/89548/> removed `llvm::StringRef::startswith` in favor of `starts_with`.
`starts_with` has been available [since Oct 2022](https://reviews.llvm.org/D136030) and `startswith` had been deprecated.
I added the following snippet:

```cpp
#if LLVM_VERSION_MAJOR >= 19
#define startswith starts_with
#define endswith ends_with
#endif
```

It's important to note that the converse approach
```cpp
#define starts_with startswith
#define ends_with endswith
```

could break code that calls `std::string_view::starts_with`.

## Changed enumerators

In November 2023, <https://github.com/llvm/llvm-project/pull/71160> changed an unnamed enumeration to a scoped enumeration.
To keep the following snippet compiling,

```cpp
          switch (tag_d->getTagKind()) {
          case TTK_Struct:
            tag = "struct";
            break;
          case TTK_Interface:
            tag = "__interface";
            break;
          case TTK_Union:
            tag = "union";
            break;
          case TTK_Class:
            tag = "class";
            break;
          case TTK_Enum:
            tag = "enum";
            break;
          }
```

I introduced macros.

```cpp
#if LLVM_VERSION_MAJOR >= 18 // llvmorg-18-init-10631-gedd690b02e16
#define TTK_Class TagTypeKind::Class
#define TTK_Enum TagTypeKind::Enum
#define TTK_Interface TagTypeKind::Interface
#define TTK_Struct TagTypeKind::Struct
#define TTK_Union TagTypeKind::Union
#endif
```

In April 2024, <https://github.com/llvm/llvm-project/pull/89639> renamed an enumerator.
I have made the following adaptation:

```cpp
#if LLVM_VERSION_MAJOR >= 19 // llvmorg-19-init-9465-g39adc8f42329
  case BuiltinType::ArraySection:
#else
  case BuiltinType::OMPArraySection:
    return "<OpenMP array section type>";
#endif
    return "<array section type>";
```

## Build system changes

In Dec 2022, <https://reviews.llvm.org/D137838> added a new LLVM library LLVMTargetParser.
I adjusted ccls's `CMakeLists.txt`:
```cmake
  target_link_libraries(ccls PRIVATE LLVMOption LLVMSupport)
  if(LLVM_VERSION_MAJOR GREATER_EQUAL 16) # llvmorg-16-init-15123-gf09cf34d0062
    target_link_libraries(ccls PRIVATE LLVMTargetParser)
  endif()
```

## Summary

The above examples illustrate how to adapt to changes in the LLVM and Clang APIs.
It's important to remember that API changes are a natural part of software development, and testing with different releases is crucial for maintaining compatibility with a wide range of LLVM versions.

When introducing new interfaces, we should pay a lot of attention to reduce the chance that the interface will be changed in a way that causes disruption to the downstream.
That said, changes are normal. When an API change is justified, do it.

Downstream projects should be mindful of the stability guarantees of different LLVM APIs.
Some API may be more prone to change than others.
It's essential to write code in a way that can easily adapt to changes in the LLVM API.

## LLVM C API

While LLVM offers a C API with an effort made towards compatibility, its capabilities often fall short.

Clang provides a C API called [libclang](https://clang.llvm.org/docs/LibClang.html).
While highly stable, libclang's limited functionality makes it unsuitable for many tasks.

In 2018, when creating ccls (a fork of cquery), I encountered multiple limitations in libclang's ability to handle code completion and indexing.
This led to rewriting the relevant code to leverage the Clang C++ API for a more comprehensive solution.
The following commits offer insights into how the C API and the mostly equivalent but better C++ API works:

* [First draft: replace libclang indexer with clangIndex](https://github.com/MaskRay/ccls/commit/103aa711d3f06f011c0f010e58e7e4ad1b48318a)
* [Use Clang C++ for completion and diagnostics](https://github.com/MaskRay/ccls/commit/c4e22bde23dcc149ef52011e334c7fdeddc79d79)
