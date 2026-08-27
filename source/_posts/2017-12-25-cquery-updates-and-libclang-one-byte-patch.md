---
layout: post
title: cquery最近改動與libclang.so一字節補丁
permalink: 2017-12-25-cquery-updates-and-libclang-one-byte-patch
tags: [emacs, lsp]
---

## cquery最近改動

cquery介紹參看[使用cquery：C++ language server](2017-12-03-c++-language-server-cquery)。

最近cquery改動比較多(<del>[終於要熬過這個聖誕了](https://twitter.com/HaskRay/status/933906701563994112)</del>)：

<!-- more -->

* 可執行文件從`build/app`變到`build/release/bin/cquery`了，支持`release/debug/asan`等多種waf variants，使用[RPATH](https://github.com/jacobdufault/cquery/pull/133)
* `./waf configure --bundled=5.0.1`可以用上最新出爐的clang+llvm 5.0.1
* Riatre把Windows構建修好了 [#154](https://github.com/jacobdufault/cquery/pull/154)
* FreeBSD可以使用了 [#155](https://github.com/jacobdufault/cquery/pull/155)及third party庫改動，感謝ngkaho1234把sparsepp FreeBSD kvm搞定
* 不需要在`initializationOptions`裏指定`resourceDir`了，感謝jiegec的[#137](https://github.com/jacobdufault/cquery/pull/137)
* 各種模板改進和function template/class template內函數引用的支持。支持了`CXCursor_OverloadedDeclRef`函數調用[#174](https://github.com/jacobdufault/cquery/issues/174)，但template call template clang-c接口沒有暴露相應信息，可能無解。
* `textDocument/hover`信息把函數名插入到函數類型裏。用了一些heuristics處理`_Atomic` `decltype()` `throw()` `__attribute(())` `typeof()`，碰到`-> int(*)()`這種還是沒救的，數組括號也不好，但大多數情況都顯示得不錯的。
* 加入了實驗性的`--enable-comments`，檢索註釋。[#183](https://github.com/jacobdufault/cquery/pull/183) [#188](https://github.com/jacobdufault/cquery/pull/188) [#191](https://github.com/jacobdufault/cquery/pull/191) 註釋和原來的聲明信息一起顯示，帶來了UI的挑戰。
* VSCode使用浮動窗口，顯示多行`textDocument/hover`不成問題。但Emacs lsp-mode和LanguageClient-neovim就遇到一些困難<https://github.com/autozimu/LanguageClient-neovim/issues/224> <https://github.com/emacs-lsp/lsp-ui/issues/17>
* `workspace/symbol`模糊匹配 [#182](https://github.com/jacobdufault/cquery/pull/182)
* danielmartin自己的repo加了實驗性的`textDocument/formatting`，格式化。這必須用clang C++ API，作者有一些顧慮。

用了一個O(n^2) sequence alignment算法，根據編輯距離、camelCase等啓發因素排序候選符號(func,type,path,...)。以`foo bar`爲模式會返回`fooBar foobar foozbar`等，而`fooBar`排在前面。Emacs `xref-find-apropos`會自作聰明地把模式用空格分割後當作正規表達式轉義，需要覆蓋掉。

## libclang `handleReference` null pointer dereference

下面介紹重點，libclang一字節補丁。

This is a longstanding issue bothering us, with all the 3 bundled clang+llvm versions: `--bundled-clang={4.0.0,5.0.0,5.0.1}`.

[topisani](https://github.com/topisani) raised the topic in <https://gitter.im/cquery-project/Lobby> and I fianlly made up my mind to diagnose it.

https://github.com/jacobdufault/cquery/issues/192

```zsh
% cd /tmp
% git clone https://github.com/nlohmann/json
% cd json
% (mkdir build; cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..)
% ln -s build/compile_commands.json
% nvim test/src/unit-constructor1.cpp  # or other editor
```

By appending `--log-file /tmp/cq.log` to the cquery command line, we can see the error message in `/tmp/cq.log`:
```
indexer.cc:1892  WARN| Indexing /tmp/json/test/src/unit-iterators2.cpp failed with errno=1
libclang: crash detected during indexing TU
```

`errno=1` indicates `CXError_Failure`，libclang uses a `sigaction+setjmp` based crash recovery mechanism to recover from SIGSEGV and returns `CXError_Failure`. Sadly cquery does not report enough diagnostics for this. For these files, there is no hover, definition, or references information.

https://github.com/jacobdufault/cquery/issues/219 [HighCommander4](https://github.com/HighCommander4) narrowed it down to a very simple reproduce：
```c++
template <typename>
struct actor;

template <template <typename> class Actor = actor>
struct terminal;
```

default template argument `= actor` causes the null pointer dereference.

In <https://github.com/llvm-mirror/clang/blob/master/tools/libclang/CXIndexDataConsumer.cpp#L203>

```c++
    handleReference(ND, Loc, Cursor,
                    dyn_cast_or_null<NamedDecl>(ASTNode.Parent),
                    ASTNode.ContainerDC, ASTNode.OrigE, Kind);
```

`dyn_cast_or_null<NamedDecl>(ASTNode.Parent)` may return NULL. In some code executed later on <https://github.com/llvm-mirror/clang/blob/master/tools/libclang/CXIndexDataConsumer.cpp#L935>:

```c++
  ContainerInfo Container;
  getContainerInfo(DC, Container);
```

`getContainerInfo` tries to cast `DC`, which uses a field in `DC` and causes a null pointer dereference.

[I have sent the patch to clang upstream for review](https://reviews.llvm.org/D41575), but clang+llvm 5.0.1 was just released. The holiday season is approaching and it is unrealistic to get this submitted and get a new release in the near future. For Linux users who do not want to build clang+llvm from source, my makeshift is `./waf configure --bundled-clang=5.0.1` with

```zsh
# --bundled-clang=5.0.1
% printf '\x4d' | dd of=build/release/lib/clang+llvm-5.0.1-x86_64-linux-gnu-ubuntu-14.04/lib/libclang.so.5.0 obs=1 seek=$[0x47aece] conv=notrunc

# --bundled-clang=4.0.0
% printf '\x4d' | dd of=build/release/lib/clang+llvm-4.0.0-x86_64-linux-gnu-ubuntu-14.04/lib/libclang.so.4.0 obs=1 seek=$[0x4172b5] conv=notrunc
```

What?

### Description

Let's check the problematic function in libclang:

```c++
bool CXIndexDataConsumer::handleReference(const NamedDecl *D, SourceLocation Loc,
                                      CXCursor Cursor,
                                      const NamedDecl *Parent,
                                      const DeclContext *DC,
                                      const Expr *E,
                                      CXIdxEntityRefKind Kind) {
  if (!CB.indexEntityReference)  // redundant check since it must be non NULL
    return false;

  if (!D)
    return false;
  if (Loc.isInvalid())
    return false;
```

According to System V x86-64 ABI, the arguments of this function are passed in the following way:

```
this: rdi
D: rsi
Loc: rdx
Cursor: stack
Parent: rcx
DC: r8
E: r9
Kind: stack
```

cquery sets `CB.indexEntityReference` to `OnIndexReference` so this field is guaranteed to be non NULL.

Scroll down the assembly listing a little bit and search for the `if (!CB.indexEntityReference)` condition:

```
% objdump -M intel -Cd build/release/lib/clang+llvm-5.0.1-x86_64-linux-gnu-ubuntu-14.04/lib/libclang.so.5 --start-address 0x47ae90 --stop-address 0x47b190
......
  47aeb0:	45 31 ed             	xor    r13d,r13d
  47aeb3:	45 85 ff             	test   r15d,r15d
  47aeb6:	0f 84 19 03 00 00    	je     47b1d5
  47aebc:	48 85 ed             	test   rbp,rbp
  47aebf:	0f 84 10 03 00 00    	je     47b1d5
  47aec5:	49 8b 44 24 18       	mov    rax,QWORD PTR [r12+0x18]  # load `IndexerCallbacks &CB`, which is actually a pointer
  47aeca:	48 8b 40 38          	mov    rax,QWORD PTR [rax+0x38]  # load CB.indexEntityReference
  47aece:	48 85 c0             	test   rax,rax     # a redundant check to see if it is null; we replace it with check of `DC`
  47aed1:	0f 84 fe 02 00 00    	je     47b1d5 <clang::cxindex::CXIndexDataConsumer::handleReference(clang::NamedDecl const*, clang::SourceLocation, CXCursor, clang::NamedDecl const*, clang::DeclContext const*, clang::Expr const*, CXIdxEntityRefKind)+0x345>
```

This redundant `if` statement produces 0x47aeca `test rax,rax`. If we replace it with `if (!DC)` (because `DC` may be NULL and we should avoid null pointer dereference), since `DC` is passed in the register R8, we may use `test r8,r8`:
```
48 85 c0  test rax,rax
4d 85 c0  test r8,r8
```

It is now clear that we only need to patch one byte, i.e. the aforementioned `printf+dd` hack. For [radare2](https://github.com/radare/radare2) users, `r2 -nwqc 'wx 4d @ 0x47aece' build/release/lib/clang+llvm-5.0.1-x86_64-linux-gnu-ubuntu-14.04/lib/libclang.so.5.0`.

Sadly radare2 dropped the ball when it was in need and I had to resort to `printf+dd`... Its assembling of `test r8,r8` was incorrect <https://github.com/radare/radare2/issues/9071>😢
