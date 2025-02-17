---
layout: post
title: "$ccls/navigate和index.multiVersion"
author: MaskRay
tags: [c++, emacs, lsp]
---

忙碌的Labor Day長週末。Growth hacking(希望Hudson River Trading用上無進展，和前clangd、Eclipse CDT開發者交談、[cdt-lsp計劃](https://bugs.eclipse.org/bugs/show_bug.cgi?id=538515)，給LanguageClient-neovim vim-lsp加wiki頁面成功)，增加了少量stargazers。學習clangd並悄悄改typo、摸索musl的開發模式，希望r2走上正途(C11/C99)，LeetCode第一次用上`std::adjacent_find`……

<!-- more -->

Language Server Protocol中最新的`textDocument/documentSymbol`可以返回`DocumentSymbol[]`，帶層次結構。但lsp-mode中尚未支持。
C/C++中的難點是out-of-line definition，declaration可以和definition分離，lexical/semantic parent的區別，可以聲明多次(如namespace)。能稍微改進我的imenu顯示效果，但實用性並不是特別大。

而利用document symbol進行declaration間快速移動更有用。我決定加個方法`$ccls/navigate`支持上下左右移動。實現很簡單：<https://github.com/MaskRay/ccls/blob/master/src/messages/ccls_navigate.cc>

```lisp
(ccls-navigate "D")
```

![](/static/2018-09-03-ccls-navigate-and-index-multiversion/D.gif)

```lisp
(ccls-navigate "L")
```

![](/static/2018-09-03-ccls-navigate-and-index-multiversion/L.gif)

```lisp
(ccls-navigate "R")
```

![](/static/2018-09-03-ccls-navigate-and-index-multiversion/R.gif)

```lisp
(ccls-navigate "U")
```

![](/static/2018-09-03-ccls-navigate-and-index-multiversion/U.gif)

LanguageClient-neovim用戶可以使用(我覺得x不值得一個單獨的鍵，挪用作前綴鍵了)

```vim
nn <silent> xh :call LanguageClient#findLocations({'method':'$ccls/navigate','direction':'L'})<cr>
nn <silent> xj :call LanguageClient#findLocations({'method':'$ccls/navigate','direction':'D'})<cr>
nn <silent> xk :call LanguageClient#findLocations({'method':'$ccls/navigate','direction':'U'})<cr>
nn <silent> xl :call LanguageClient#findLocations({'method':'$ccls/navigate','direction':'R'})<cr>
nn xx x
```

## `index.multiVersion`

一直以來，ccls使用繼承自cquery的每個文件只檢索一次的模型，一個文件即使被不同方式編譯(如#include前使用不同的macro)，也只有一個版本被記錄下來。好處是不同translation unit間索引的東西基本不重複，大大減少索引文件的尺寸。

但缺陷是對於下面的例子只能索引到一個分支：

```cpp
#ifdef D
// foo() A
#else
// foo() B
#endif
```

```cpp
// a.h
template<class T>
int bar(T a) { return a.foo(); }  // foo或者跳到a.cc中，或者跳到b.cc中，取決於誰最晚索引

// a.cc
#include "a.h"
struct A { int foo() { return 0; } };
int a = bar(A());

// b.cc
#include "a.h"
struct B { int foo() { return 0; } };
int b = bar(B());
```

這個改動需要變更索引數據結構和pipeline，把`all_symbols` `outline`的構建時刻從indexer推遲到main thread合併indexer信息時。示意：

```cpp
struct Reference {
  Range range; // 8 bytes
  Usr usr;     // 8 bytes
  SymbolKind kind; // 1 byte followed by 1-byte padding
  Role role;   // 2 bytes
};
struct SymbolRef : Reference {};

// before
struct QueryFile {
  std::vector<SymbolRef> all_symbols;
  std::vector<SymbolRef> outline;
};

// after
struct QueryFile {
  llvm::DenseMap<SymbolRef, int> symbol2refcnt;
  llvm::DenseMap<SymbolRef, int> outline2refcnt;
};
```

開啓方式：
```javascript
{
  "index": {
    "multiVersion": 1,
    "multiVersionBlacklist": ["^/usr/include"]
  }
}
```

用blacklist讓系統headers仍然只索引一遍，否則C++ headers會大幅膨脹索引文件(在一個例子中兩倍體積)。

glibc中很多文件會被非PIC和PIC編譯兩遍，產生重複的`compile_commands.json` `a.c`條目，`index.multiVersion: 1`並不解決這個問題。另外`a.h`保存後是否所有包含它的`*.c`都要重新索引呢？我現在仍然使用`textDocument/didSave`時只索引一個的方案，否則太浪費。

內存和索引文件體積比較。下面RSS爲索引結束後resident set size，其實如果重啓language server減少內存碎片，可以少一些，ccls-cache爲保存在磁盤上的索引目錄的du -sh。

```
musl
RSS: 72M, ccls-cache: 16M
RSS: 196M, ccls-cache: 16M

radare2
RSS: 388M, ccls-cache: 112M
RSS: 1297M, ccls-cache: 496M

ccls
RSS: 319M, ccls-cache: 51M
RSS: 874M, ccls-cache: 204M
```

上週用上了clangd的StoreInMemory preamble技巧改進completion速度、移除ASTUnit優化diagnostics。

有人願意幫忙往ale(Vim插件)、eglot、Oni(Neovim GUI)、theia-ide等提issue要求支持ccls我會很感激(自己不好意思做嘛)

其他可以幫忙的地方至少有：

* 改進vscode插件。上游的vscode-cquery也很久沒動了……我討厭VSCode肯定不願意多動vscode-ccls
* 弄清楚Windows msys2用新gcc鏈接llvm https://github.com/MaskRay/ccls/issues/59
* 加FreeBSD port、加Gentoo ebuild、弄MacPorts Homebrew等……
* spacemacs https://github.com/syl20bnr/spacemacs/pull/11242
