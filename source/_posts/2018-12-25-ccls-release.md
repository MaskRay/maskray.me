---
layout: post
title: ccls 0.20181225 release
author: MaskRay
tags: [ccls]
---

聖誕發佈新版<https://github.com/MaskRay/ccls/releases/tag/0.20181225>

<!-- more -->

* `workspace/didChangeWatchedFiles` works (tested on VSCode and coc.nvim) and file deletion will remove index symbols
* `.ccls` enhancement [#171](https://github.com/MaskRay/ccls/pull/171) added new directives `%compile_commands.json` `%h` `%hpp` `%objective-c` `%objective-cpp`
* `-v=1` dumps command line options of files for parsing
* fuzzy_match: when the completion filter begins with a letter, builtin macros prefixed with an underscore will not be returned
* Support `textDocument/declaration` and `LocationLink[]` return types.
* Properly handle `"exit"` [#159](https://github.com/MaskRay/ccls/pull/159)
* Support `signatureHelp.signatureInformationparameterInformation.labelOffsetSupport` cf. <https://github.com/Microsoft/language-server-protocol/issues/640>
* codeAction: fixed an incompatibility issue with VSCode

## Completion

* More precise diagnostics/completion: an included file that has been changed does not need to be saved to take effect.
* Decreased `Content-Length:` from 32K to 25K for some cases
* On clang < 8, fixed `#include <` completion for `-I dir`
* Macros are categorized as `Text`, instead of `Interface`
* Refactor and rename (`clang_complete.cc` -> `sema_manager.cc`)
* C++17 deduction guide #173

## Others

* Support multiple `-init=`: "initializationOptions" from client are applied first, then `-init=`. Scalar options will be overridden while arrays will get concatenated

重點是

* 更加精確的diagnostics/completion(針對#include中的依賴unsaved buffers)，之前cquery/ccls的做法都不好
* 檔案刪除、重命名：之前這兩個專案都不會清理掉刪除的檔案 [#170](https://github.com/MaskRay/ccls/pull/170)
* #159 是之前LSP exit觸發`exit(0)`(中斷一個正在寫cache的indexer thread下次可能讀不出來，但非常罕見)
* 補全的其他大大小小改進
* 如果一個textDocument的索引未被加載，request (`textDocument/documentHighlight` `textDocument/hover`等)會報告`not indexed`，顯示在echo area，比較惱人。我在`pipeline.cc`引入了一個backlog解決這一問題，詳見<https://github.com/MaskRay/ccls/pull/176>。code lens可以用`(add-hook 'lsp-after-open-hook #'ccls-code-lens-mode)`而不需要`(run-at-time 0.5 nil #'ccls-code-lens-mode)`了
* 增強`.ccls`和`compile_commands.json`的協作，詳見Paul Smith的分析<https://github.com/MaskRay/ccls/issues/115#issuecomment-449455357>

## 其他

* ccls/wiki/Emacs拆分成[wiki/lsp-mode](https://github.com/MaskRay/ccls/wiki/lsp-mode)和[wiki/egot](https://github.com/MaskRay/ccls/wiki/eglot)，後者還比較簡陋(歡迎貢獻)
* eglot支援`workspace/didChangeWatchedFiles`，lsp-mode尚未
* 我給lsp-mode加了`LocationLink`支援，`targetSelectRange`可以指向名字，而`targetRange`指向輪廓。mouse hover時可以有可視效果(對於Emacs用途不大)
* xref.el的`xref-file-location`並非表示`interface Range`的良好選擇，因爲僅能表示點(line column)，不能表示區間。lsp-ui-peek裏需要區間信息highlight文本，無法良好地用xref表達，在lsp-ui-peek裏的實現有點噁心(希望有人能改好)
```
;; xref.el
(defclass xref-file-location (xref-location)
  ((file :type string :initarg :file)
   (line :type fixnum :initarg :line :reader xref-location-line)
   (column :type fixnum :initarg :column :reader xref-file-location-column))
  :documentation "A file location is a file/line/column triple.
Line numbers start from 1 and columns from 0.")

;; lsp-ui-peek.el
          (-if-let (uri (gethash "uri" x))
              (-let (((&hash "start" (&hash "line" "character")) (gethash "range" x)))
                (lsp-ui-peek--goto-xref `(:file ,(lsp--uri-to-path uri) :line ,line :column ,character)))
            (-let (((&hash "start" (&hash "line" "character")) (or (gethash "targetSelectionRange" x) (gethash "targetRange" x))))
              (lsp-ui-peek--goto-xref `(:file ,(lsp--uri-to-path (gethash "targetUri" x)) :line ,line :column ,character))))

```
* 非常希望看到有人能把lsp-mode的hash tables+string keys (`(gethash "range" xx)`)都改成plists + keyword keys (`(plist-get xx :range)`)，這樣會讓我感覺還是有希望的……
* code lens我另外也在等待eglot的處理：<https://github.com/joaotavora/eglot/pull/71>，lsp-mode中的issue是<https://github.com/emacs-lsp/lsp-mode/issues/361>
