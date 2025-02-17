---
layout: post
title: 2018-09-09 ccls最近更新
author: MaskRay
tags: [c++, emacs, lsp]
---

加了一些initialization options，參見<https://github.com/MaskRay/ccls/tree/master/src/config.h>。

<!-- more -->

* `cacheDirectory: ""`

默認值爲`.ccls-cache`，索引時會在項目目錄下創建`.ccls-cache`目錄放置cache文件，再次打開時無需重新索引。現在這個選項可以指定爲空字串，使索引文件放在內存中。

這個索引不要和ccls另外的全局索引混淆了。全局索引始終在內存中。打開項目讀入索引文件時，會把索引文件合併到全局索引。當一個文檔修改時，需要把全局索引中數據改文檔的部分刪除，插入新的，因此每個文檔留有一份記錄。

* `clang.excludeArgs: ["-fopenmp"]`

可以用於排除僅GCC支持的選項、讓clang產生不想要的diagnostics的選項、產生索引問題的選項等。

* `diagnostics.onChange` `diagnostics.onSave`

默認`diagnostics.onChange: true`，`diagnostics.onSave`和`diagnostics.onSave`爲false，代表文檔編輯時觸發diagnostics，保存時則不會。
編輯時會用clang反覆parse，如果擔心性能問題，可以設置`diagnostics.onChange: false`。

* `index.onChange: true`

默認爲false，設置爲true可以即時索引，注意把`cacheDirectory`設爲空字串防止反覆讀寫文件。
```javascript
{
  "cacheDirectory": "",
  "index": {"onChange": true}
}
```

* `$ccls/navigate`可以用於`namespace`了

之前無法知道`namespace foo {}`中`}`的位置，現在非definition的declaration表示爲：
```cpp
struct Reference {
  Range range;
  Usr usr;
  SymbolKind kind;
  Role role;
};
struct Use : Reference {
  int file_id = -1;
};
struct DeclRef : Use {
  Range extent; // 能表示namespace foo {}整個範圍
};
```

參見wiki中Emacs和LanguageClient-neovim頁面介紹。

另外之前`textDocument/hover`作用在`namespace foo`上不顯示名字，現在顯示了。

* indexer改進

conversion function `operator int`的整個作爲spelling range，之前只有`operator`部分是。`textDocument/documentHighlight`顯示不好看。

destructor `~foo`整體作爲spelling range了，之前只有`~`部分。

C中常用的`typedef struct {} foo;`寫法，原來anonymous struct沒有名字，現在表示爲`anon struct foo`(name for linkage purposes)。

indexer也能復用completion和diagnostics的preamble，加速索引。

修復了一個out-of-bands修改源文件，載入索引文件可能導致初始錯誤引用計數`symbol2refcnt`修改的bug。pipeline的multiversion concurrency control真的超級粗糙，需要人來研究下……
