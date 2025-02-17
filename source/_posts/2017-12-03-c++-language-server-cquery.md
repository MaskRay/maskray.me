---
layout: post
title: 使用cquery：C++ language server
author: MaskRay
tags: [emacs, lsp]
---

更新：我現在用自己的[ccls](https://maskray.me/blog/2018-04-01-c++-language-server-ccls)了。

請先瞭解[Language Server Protol](https://microsoft.github.io/language-server-protocol/)

## C++代碼索引工具現狀

<!-- more -->

Tag system流派

* [Universal Ctags](https://ctags.io/)很精妙，用正則表達式跳轉，因此文檔編輯後仍能使用。<https://github.com/universal-ctags/ctags/blob/master/parsers/c.c>
  但不帶引用，Vim會用二分查找，大約會有log2(size/4096)次seek。
* [Cscope](http://cscope.sourceforge.net/)似乎已經荒廢了。
* [ID Utils](https://www.gnu.org/software/idutils/)
* [GNU GLOBAL](https://www.gnu.org/s/global/) `libparser/C.c`。用Berkeley DB存儲definition/reference/path name。帶有插件系統可以使用ctags idutils的parser。對於Emacs/Vim用戶來說，可能是tag流派中最好用的工具了。輔以一些heuristics和[ripgrep](https://github.com/BurntSushi/ripgrep)等，很多用戶不覺得自己生活在水深火熱中……
* [Elixir Cross Referencer](https://github.com/free-electrons/elixir)
* [OpenGrok](https://opengrok.github.io/OpenGrok/)

clang流派

* [clang-tags](https://github.com/ffevotte/clang-tags)荒廢。
* [YouCompleteMe](https://github.com/Valloric/YouCompleteMe)不夠好用，因爲只處理單一translation unit，無法查找引用。
* [clangd](https://clang.llvm.org/extra/clangd.html)最有前景，有大廠大項目願意採用，Xcode會用讓clangd有助益的libindexstore。但目前尚無存儲系統，因此無法處理多translation units。作爲[clang-tools-extra](https://clang.llvm.org/extra/)一部分，而clang+llvm構建/貢獻門檻高([https://reviews.llvm.org/])。對於這類工具類應用，貢獻難易程度是個重要因素。目前有嘗試引入存儲模型(MarkZ3)，但目前設計較爲複雜，而實際上不帶garbage collection的`std::vector`(cquery風格)足夠應對大部分使用場景。很擔心他們走上歧路。
* [Google Kythe](https://github.com/google/kythe)，(mostly) language-agnostic，概念複雜，配置困難。不重視language server protocol，[當前](https://github.com/google/kythe/blob/master/kythe/go/languageserver/languageserver.go)僅提供`ReferencesProvider,HoverProvider,DefinitionProvider`，且交互使用可能有極大延遲。大多數人並不在意C++ Haskell Python代碼間無縫跳轉。<https://github.com/google/kythe/tree/master/kythe/cxx/indexer/cxx>
* [rtags](https://github.com/Andersbakken/rtags)可以查找引用，但每個translation unit 6個文件`info,symbols,symnames,targets,tokens,usrs`(過多)，沒有使用in-memory索引，查找引用請求會讀項目所有translation units的文件。導致性能低下<https://github.com/Andersbakken/rtags/issues/1007>。rtags.el裏應該還有很多東西可供Emacs lsp-mode學習，有經驗的人介紹一下～
* [cquery](https://github.com/jacobdufault/cquery)現階段的妥協。主要數據結構爲不帶garbage collection(變量/函數/類型等的id不會回收)的`std::vector`(`src/indexer.h`)。有一些Emacs用戶積極貢獻[code navigation功能](https://code.visualstudio.com/docs/editor/editingevolved)。

IDE(Any sufficiently complicated IDE contains an ad-hoc, informally-specified, bug-ridden, slow implementation of half of C++.)

* NetBeans使用[Clank](http://llvm.org/devmtg/2017-03//assets/slides/clank_java_port_of_c_cxx_compiler_frontend.pdf)
* Eclipse用[CDT](https://www.eclipse.org/cdt/)

## cquery安裝、配置

* git clone <https://github.com/jacobdufault/cquery>
* 構建language server可執行文件(Arch Linux可用[aur/cquery-git](https://aur.archlinux.org/packages/cquery-git)的`/usr/bin/cquery`)
  + `./waf configure` # 或用--bundled-clang=5.0.1選擇<http://releases.llvm.org/>上的release版本
  + `./waf build` # 構建`build/release/bin/cquery`
* 編輯器安裝language client插件(Emacs lsp-mode、Neovim LanguageClient-neovim、VSCode安裝cquery/vscode-client裏的插件)
* 爲你的C/C++/Objective-C項目生成`compile_commands.json`，參見下文。

cquery是一個C++ language server，它和編輯器端的LSP client協作流程如下：

當編輯器打開C++文件時，language client插件啓動language server進程(根據配置的language server可執行文件)，用[JSON-RPC 2.0](http://www.jsonrpc.org/specification)協議通過stdio通信，協議規範見<https://microsoft.github.io/language-server-protocol/specification>。

Language client插件用`initialize`請求告知language server(這裏是`build/release/bin/cquery`進程)自己支持的功能(ClientCapabilities)、項目路徑(rootUri)、初始化選項(initializationOptions，cquery需要知道`cacheDirectory`路徑)。之後各種語言相關功能都通過與language server通信實現：

+ 光標移動時向language server發送`textDocument/hover`請求，language server返回變量/函數聲明信息、註釋等。VSCode使用浮動窗口顯示，Emacs lsp-mode用eldoc顯示
+ 查找定義發送`textDocument/definition`請求，language server返回定義所在的文件、行列號。編輯器的可能行爲：單個結果時直接跳轉到目標文件的指定行列，如有多個候選則用菜單顯示
+ 查找引用發送`textDocument/references`請求，和查找定義類似
+ 查找當前文檔定義的符號(常用與查找頂層的outline)發送`textDocument/documentSymbol`請求，language server返回符號和行列號
+ 查找項目定義的符號(只查找outline的也很有用)發送`workspace/symbol`請求
+ 補全`textDocument/completion`，language server提供候選及排序方式，是否使用snippet，如何編輯文檔得到補全結果等
+ 文檔編輯操作發送`textDocument/didChange`，language server據此更新自己的模型
+ cquery還支持一些Language Server Protocol之外的擴展，比如`$cquery/derived`用於查找派生的類、方法等

## Emacs

參照<https://github.com/jacobdufault/cquery/wiki/Emacs>配置。需要安裝幾個插件：

* [lsp-mode](https://github.com/emacs-lsp/lsp-mode) Emacs裏的LSP客戶端庫，可用於多種language server。另有lsp-rust、lsp-haskell等，可以看作適配器，包含language server相關設置。
* cquery項目中的`emacs/cquery.el`。地位與lsp-rust、lsp-haskell等類似，把cquery適配到lsp-mode。另外支持cquery一些Language Server Protocol之外的擴展。
* [lsp-ui](https://github.com/emacs-lsp/lsp-ui) lsp-mode有計劃併入Emacs。其他UI相關或因協議等問題不適合在核心lsp-mode包的組件放在這裏。當前有：
  + lsp-ui-flycheck 用language server的diagnostics信息實現flycheck的checker
  + lsp-ui-sideline 即時顯示當前行所有標識符的`textDocument/hover`信息
  * lsp-ui-peek 基於[quick-peek](https://github.com/cpitclaudel/quick-peek)的`find-{definitions,references,apropos}`
  + 未來可能添加更多code lens功能
* [company-lsp](https://github.com/tigersoldier/company-lsp) company是一個補全引擎，company-lsp爲一個backend，用`textDocument/completion`信息提供補全

這些插件只有lsp-mode和cquery.el是必須的。

### [lsp-mode](https://github.com/emacs-lsp/lsp-mode)

* `(lsp-enable-imenu)`開啓，用imenu來顯示`textDocument/documentSymbol`信息。跳轉到當前檔案的符號很方便。
* 光標移動到標識符上會觸發`textDocument/hover`，顯示類型、變量、函數等的fully qualified name，有層層namespace嵌套時容易定位。對於auto specifier，能知道具體類型。
* `M-x lsp-format-buffer`發送`textDocument/formatting`。參見[cquery/wiki/Formatting](https://github.com/jacobdufault/cquery/wiki/Formatting)

![hover/documentHight，顯示fully qualified name](/static/2017-12-03-c++-language-server-cquery/hover.jpg)

### xref.el

xref.el是Emacs自帶的。lsp-mode設置`xref-backend-functions`，讓xref.el使用lsp後端。如果不安裝其他庫，也能用以下三個函數，結果由xref.el渲染。

* `xref-find-definitions` (默認`M-.`)，查找定義，發送`textDocument/definition`請求
* `xref-find-references` (默認`M-?`)，查找引用，發送`textDocument/references`請求
* `xref-find-apropos` (默認`C-M-.`)，查找項目符號，發送`workspace/symbol`請求

`xref-find-definitions`若只有一個結果會直接跳轉，有多個則彈出菜單供用戶選擇。而`xref-find-references`會觸發`xref--read-identifier`，在minibuffer中要求讀入一個串。這顯然和期望的查找當前光標位置引用的使用方式不符。另外，lsp-mode會讀取光標處標識符的text properties信息(其中編碼了buffer內位置信息)，而prompt讀入的串是不帶text properties的。`xref-find-references`會失敗。

要讓它工作，請閱讀`xref-prompt-for-identifier`文檔，把`xref-find-references`添加進`xref-prompt-for-identifier`。我提交了一個bug到Emacs(因爲`xref.el`是Emacs一部分)：<https://debbugs.gnu.org/cgi/bugreport.cgi?bug=29619>，但maintainer需要瞭解更多用戶的反饋才會修改`xref-prompt-for-identifier`默認值。

cquery `workspace/symbol`使用了一個sequence alignment結合詞結合性、camelCase等啓發因素的fuzzy matching算法，以`foo bar`爲模式會返回`fooBar foobar foozbar`等，`fooBar`排在前面。`xref-find-apropos`會自作聰明地把模式用空格分割後當作正規表達式轉義，考慮自定義。

## [company-lsp](https://github.com/tigersoldier/company-lsp)

提供LSP的補全支持。我用spacemacs的`(spacemacs|add-company-backends :backends company-lsp :modes c-mode-common)`。

tumashu寫了一個company-childframe.el，可能需要人推動一下[company-mode#745](https://github.com/company-mode/company-mode/issues/745)。

## `cquery.el`

cquery項目中的`cquery.el`適配cquery到lsp-mode，同時提供一些LSP協議未定義的功能。如inactive region，把preprocessor忽略掉的行用灰色顯示：

![inactive region和company-lsp](/static/2017-12-03-c++-language-server-cquery/inactive-region-and-company.jpg)

我用以下C/C++ mode hook在項目根目錄有`compile_commands.json`時自動啓用`lsp-cquery-enable。

```lisp
(defun my//enable-cquery-if-compile-commands-json ()
  (when
      (and (not (and (boundp 'lsp-mode) lsp-mode))
           (or
            (cl-some (lambda (x) (string-match-p x buffer-file-name)) my-cquery-whitelist)
            (cl-notany (lambda (x) (string-match-p x buffer-file-name)) my-cquery-blacklist))
           (or (locate-dominating-file default-directory "compile_commands.json")
               (locate-dominating-file default-directory ".cquery")))
    (setq eldoc-idle-delay 0.2)
    (lsp-cquery-enable)
    (lsp-enable-imenu)
    (when (>= emacs-major-version 26)
      (lsp-ui-doc-mode 1))))
```

另外一些不在LSP協議中的cquery擴展方法，如：

* `$cquery/base` 用於類型是查找base class，也可用於virtual function
* `$cquery/derived` 用於類型是查找derived classes，也可用於virtual function查找被哪些derived classes override
* `$cquery/vars` 查找一個類型的所有變量

另外有個`$cquery/typeHierarchyTree`，但還沒有人搬到Emacs，用空的話用個畫樹的庫造福其他人～

### [helm-xref](https://github.com/brotzeit/helm-xref)

[helm](https://github.com/emacs-helm/helm)用戶可以考慮安裝[helm-xref](https://github.com/brotzeit/helm-xref)，`(setq xref-show-xrefs-function 'helm-xref-show-xrefs)`即可。`xref-find-{definitions,references,apropos}`會用helm顯示，替代`xref.el`的界面。

helm-xref效果如圖![](/static/2017-12-03-c++-language-server-cquery/helm-xref.jpg)。

### lsp-ui-doc

使用了child frame，需要Emacs 26或以上。

Language client中命令行設置爲`cquery --language-server --enable-comments`可以索引項目中的註釋(文檔)。`textDocument/hover`信息除了提供類型簽名，還會提供註釋。
![lsp-ui-doc顯示註釋](/static/2017-12-03-c++-language-server-cquery/lsp-ui-doc.jpg)

```lisp
(setq lsp-ui-doc-include-signature nil)  ; don't include type signature in the child frame
```

### lsp-ui-flycheck

```lisp
(with-eval-after-load 'lsp-mode
   (add-hook 'lsp-after-open-hook (lambda () (lsp-ui-flycheck-enable 1))))
```

### lsp-ui-peek

[lsp-ui](https://github.com/emacs-lsp/lsp-ui)提供了不同於xref.el的另一套交叉引用。參見其主頁的demo。
```lisp
M-x lsp-ui-peek-find-definitions
M-x lsp-ui-peek-find-references
M-x lsp-ui-peek-find-workspace-symbol

# 不要隱藏非當前文件的匹配項的
(setq lsp-ui-peek-expand-function (lambda (xs) (mapcar #'car xs)))

(define-key lsp-ui-peek-mode-map (kbd "h") 'lsp-ui-peek--select-prev-file)
(define-key lsp-ui-peek-mode-map (kbd "l") 'lsp-ui-peek--select-next-file)
(define-key lsp-ui-peek-mode-map (kbd "j") 'lsp-ui-peek--select-next)
(define-key lsp-ui-peek-mode-map (kbd "k") 'lsp-ui-peek--select-prev)
```

以下三個cquery擴展協議也很有用，建議設置快捷鍵。
```lisp
(lsp-ui-peek-find-custom nil "$cquery/base")
(lsp-ui-peek-find-custom nil "$cquery/callers")
(lsp-ui-peek-find-custom nil "$cquery/derived")
```

### lsp-ui-sideline

```lisp
(setq lsp-ui-sideline-show-symbol nil)  ; don't show symbol on the right of info
```

### 其他

* LSP生態系統解決的一大痛點是以前對於不同語言，要使用不同工具，設置不同快捷鍵。用了language client就可以統一了。

注意[`textDocument/references`協議](https://microsoft.github.io/language-server-protocol/specification#textDocument_references)中定義返回結果爲`Location[] | null`，只含位置信息，不包含代碼行內容。顯示行內容是lsp-mode做的。

我的配置：<https://github.com/MaskRay/Config/blob/master/home/.emacs.d/layers/%2Bmy/my-code>

* 希望[spacemacs](http://spacemacs.org/)支持LSP。`reference-handler`(類似於跳轉到定義的`jump-handler`)也很有用：https://github.com/syl20bnr/spacemacs/pull/9911
* lsp-mode和[ggtags](https://github.com/leoliu/ggtags)都會`(setq-local eldoc-documentation-function ...)`，對於這類minor-mode衝突問題，如果能設置優先級就能優雅解決。

## Neovim

參照<https://github.com/autozimu/LanguageClient-neovim/wiki/cquery>。相關組件：

* [LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim) Neovim裏的LSP客戶端
* [denite](https://github.com/Shougo/denite.nvim) 提供UI支持，顯示LanguageClient-neovim獲取的信息
* [fzf](https://github.com/junegunn/fzf) 提供UI支持
* [deoplete](https://github.com/Shougo/deoplete.nvim) [nvim-completion-manager](https://github.com/roxma/nvim-completion-manager) 補全UI
* [neosnippet](https://github.com/Shougo/neosnippet.vim) 可以和deoplete配合，snippet
* 目前缺乏Emacs中`cquery.el`的對應物，提供LSP協議未定義的cquery特定功能

```vim
nn <leader>ji :Denite documentSymbol<cr>
nn <leader>jI :Denite workspaceSymbol<cr>
" 終端限制，<C-,>不可用。ord(`,`) & 64爲0無法表示
nn <M-,> :Denite references<cr>
nn <silent> <C-j> :MarkPush<cr>:call LanguageClient_textDocument_definition()<cr>
```

![`textDocument/workspaceSymbol`](/static/2017-12-03-c++-language-server-cquery/neovim-workspace-symbol.jpg)。

## 生成[compile_commands.json](https://clang.llvm.org/docs/JSONCompilationDatabase.html)

cquery這類Clang LibTooling工具和傳統tag-based工具的一大差別是瞭解項目中每個源文件和編譯方式。放在項目根目錄的[compile_commands.json](https://clang.llvm.org/docs/JSONCompilationDatabase.html)提供了這種信息。

### [CMake](https://cmake.org/)

```zsh
% mkdir build
% (cd build; cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=YES ..)
% ln -s build/compile_commands.json
```

### [Build EAR](https://github.com/rizsotto/Bear)

[Bear](https://github.com/rizsotto/Bear) is a tool that generates a compilation database for clang tooling. It can be used for any project based on `Makefile`.

```zsh
bear make
# generates compile_commands.json
```

### [Ninja](https://ninja-build.org/)

```zsh
ninja -t compdb rule_names... > compile_commands.json
```

## 深入

cquery使用Clang的C接口libclang parse/index文件。Clang C++ API不穩定，cquery使用C++ API可能會難以適配不同Clang版本。使用`--use-clang-cxx`編譯選項可以用Clang C++ API。但注意可能會顯著增加cquery構建時間。Windows releases.llvm.org的bundled clang+llvm不帶C++頭文件。

`--enable-comments`可以索引項目中的註釋，VSCode渲染完美，但Emacs [Vim](https://github.com/autozimu/LanguageClient-neovim/issues/224)的顯示還在改善中。註釋的排版，如何parse comment markers([Doxygen](https://www.stack.nl/~dimitri/doxygen/manual/docblocks.html),[standardese](https://github.com/foonathan/standardese))還有[很多爭論](https://github.com/jacobdufault/cquery/pull/187)。

`#include <algorithm>` 在include行也能跳轉，但如果是項目外的文件(系統頭文件)，你的LSP client可能不會把它和之前的LSP session關聯，你就無法在新打開的buffer中用LSP功能了。

`A a;` 對於聲明/定義，在`a`上`textDocument/definition`會跳到類型`A`的定義。在`a`旁邊的空格或分號會跳到constructor。因爲constructor標記爲implicit，代碼中讓implicit函數調用的範圍左右擴展一格，那麼就更容易觸發了。

`A a(3);` 在` (`或`);`上`textDocument/definition`會跳到類型constructor。

`A a=f();` 如果有隱式copy/move constructor，在`(`上能跳到它們。

`assert(1);` 在`assert`上會跳到`#define assert`，但在`(1)`上會跳到`__assert_fail`，`__assert_fail`來自`assert` macro的展開。libclang `IndexerCallbacks.indexEntityReference`回調會報告來自`__assert_fail`的引用，因此請不要驚訝。

`auto a = std::make_unique<A>(3);` `make_unique`會跳轉到constructor，因爲`src/indexer.cc`中對`make`開頭的模板函數有特殊邏輯，會跳到constructor而不是`make_unique`的定義。

function/class template裏有些東西有def/ref信息，但`A<int>::foo()`等引用跳轉不了，是因爲模板索引的困難[#174](https://github.com/jacobdufault/cquery/issues/174)。

有餘力～請更新<https://github.com/jacobdufault/cquery/wiki>～

## 問題

* Task lists <https://github.com/jacobdufault/cquery/issues/30> Polish before publishing (to GitHub Marketplace)
* 需要一個妥善的on-disk storage。很多輕量級數據庫不支持或有較難處理的問題(如果有需求把現在in-memory+JSON改成有其他存儲模型)。註記，`SQLITE_ENABLE_LOCKING_STYLE`、flock很難。

## 其他

索引Linux kernel

```zsh
wget 'https://git.archlinux.org/svntogit/packages.git/plain/trunk/config?h=packages/linux' -O .config
yes '' | make config
bear make -j bzImage modules
```

生成3GiB文件。

索引llvm，`du -sh => 1.1GB`，索引完內存佔用2G。

查看LSP requests/responses

```zsh
sudo sysdig -As999 --unbuffered -p '%evt.type %evt.buffer' "proc.pid=$(pgrep -fn build/app) and fd.type=pipe" | egrep -v '^Content|^$'
```

希望有朝一日Debug Protocol也能獲得重視，<https://github.com/Microsoft/vscode-debugadapter-node/blob/master/protocol/src/debugProtocol.ts>，讓[realgud](https://github.com/realgud/realgud/wiki/Debuggers-Supported)輕鬆一點。

和YouCompleteMe等項目一樣，cquery默認下載[prebuilt clang+llvm](http://releases.llvm.org/download.html)，即`.h .so .a`。用戶不需要編譯完整的llvm，開發門檻比clangd低。

哪些源文件處理不好：

* 多executable
* X macros，一份源碼多種編譯方式
* ODR violation
* self-modifying code
* dlopen
* weak symbol(不知道鏈接命令)

感謝ngkaho1234。
