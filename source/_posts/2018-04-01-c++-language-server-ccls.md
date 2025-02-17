---
layout: post
title: "ccls: a fork of the C++ language server cquery"
author: MaskRay
tags: [c++, emacs, lsp]
---

感覺很矛盾，一方面希望自己做的東西能有廣泛受衆(比如一個學長因此找到了我)，另一方面，項目關注多了issue也多了(沒有建設性/用戶錯誤也多了，我真的很想叫這類issue愚蠢)，之前三個月每天看cquery issues很勞累。

現在更多地考慮是自己用着舒服，當然如果能不費太大工夫幫到其他Linux/FreeBSD用戶，或是我自己也想嘗試知道的一些奇異的配置如cross compiling，我也會願意做的。有近500個commits加做了大力推廣，我確實是居功自傲，但原作者傷人的不僅僅是改變了協作模型，不多提。想到這篇文章會被別人翻譯着看挺高興的呢~LanguageClient-neovim cquery方法、lsp-ui一些東西(主要作者sebastiencs，我早期也做了一點修補)、Wiki、Reddit推廣、spacemacs +tools/lsp加已放棄的+tools/cquery(已經轉投doom-emacs~)、langserver.org、推動Arch Linux/FreeBSD包、……

Good Friday

如果是今年一月cquery還有很多問題時發生這個變故，我肯定會毫不猶豫fork自立門戶再試圖超過它。但現在我想不出更加革新性的功能增強，libclang索引的上限我也有所瞭解，我也沒那麼多精力在空閒時間再做更多事。前些時候commits多是因爲對交叉引用功能不滿+拿這個項目學習C++，懂得了一些犄角旮旯的知識。比如clang 3.5推導lambda return type的缺陷，copy initialization用move structor clang 3.9之前的缺陷……玩generic lambda等。

clangd主要開發的人面向超大代碼庫和Chrome，但他們的方向感覺不對勁，加之code review，一些簡單功能也非要用大量代碼實現。到現在還在用AST的方法蠻幹，沒有合適的內存內交叉索引數據結構。現在開始又多了些workspace/symbol等功能，未來重構這些添加的方法只會增加重複勞動。

* <https://github.com/MaskRay/ccls>
* <https://github.com/MaskRay/emacs-ccls>
* 請求添加到Melpa <https://github.com/melpa/melpa/pull/54053>

自己做主，主要爲自己服務，就能作出很多cquery難以做的選擇：

* `-std=c++17`，向telegram-desktop看齊，可以刪除`variant optional string_view`的third-party依賴。用clang++ 6.0.0和系統libstdc++ (gcc 7.3左右)需要這個補丁使得`std::get<...>(std::variant<...>)`可用。<https://gcc.gnu.org/viewcvs/gcc/trunk/libstdc%2B%2B-v3/include/std/variant?view=markup&pathrev=258854>
* 拋棄`textDocument/documentLink textDocument/rangeFormatting $cquery/wait`等幾乎沒用的功能。把.h函數定義放到.cc裏這類refactor功能`clang-*`等外部工具能做的，不值得在language server中實現。這些功能未來會形成命令行/clangd refactor框架，libclang可能永遠做不了這些事。
* 刪除默認的`-fms-compatibility -fdelayed-template-parsing -fms-extensions`。這是原作者revert我的一個commit
* 刪除`"command"/"arguments"`中對compiler scheduler `goma clang`的特判。提供給libclang的命令行應該做儘可能少的變換，用戶自行添加選項適配自己的項目。
* `src/`下文件實在太多了，很多東西幾十行就寫成一個新文件。不必要的東西我刪除了一些。
* 修了一個`/usr/include/c++/7.3.0/`是指向`/usr/include/c++/7` symlink時，跳轉到系統頭文件沒有索引信息的問題。
* `#include ""`不補全STL。`""`中包含系統頭文件合法，但是不符合規範。
* 可以用C++ optional TS filesystem代替各類hack的文件系統函數。
* 放棄`#include <c*>`轉用C風格`#include <*.h>`。我比較懶惰，想省`std::`等幾個字。
* 用`stdio`，拋棄`iostream`

`import_pipeline file_contents file_consumer`等文件，我也比較茫然，每次要改都得重新理論下。這裏流程太複雜了，不清楚能怎麼簡化。

不知道這個項目我能堅持到什麼時候，使用請注意風險~

Resurrection Sunday

## Building trunk libclang

* `CXSymbolRole`: read/write/addr references
* `clang_File_tryGetRealPathName`: on Arch Linux, system header include paths extracted from `clang -E -v -xc++ /dev/null` have several `../` path components, which confuse `clang_getFileName`. You'll need `clang_File_tryGetRealPathName` to correctly resolve paths of system C++ headers.

They require `#if CINDEX_VERSION >= 48`, while `CINDEX_VERSION` shipped with clang 6.0.0 is 45.

Here are instructions to build trunk libclang(see <https://llvm.org/docs/CMake.html> for LLVM cmake options). *Note*, please use a powerful workstation to build LLVM.
```zsh
git clone https://git.llvm.org/git/llvm.git
cd llvm
git clone https://git.llvm.org/git/clang.git tools/clang
cmake -G Ninja -DCMAKE_BUILD_TYPE=MinSizeRel -DCMAKE_EXPORT_COMPILE_COMMANDS=On -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_C_COMPILER=clang -DLLVM_OPTIMIZED_TABLEGEN=On -DLLVM_USE_LINKER=lld -Bstatic-release -H.
ninja -C static-release libclang
# Built static-release/lib/libclang.*
```

If you want to copy built libclang from `workstation` to local:
```zsh
rsync -a -f '+ lib' -f '+ lib/clang**' -f '+ lib/libclang.*' -f '- *' \
  workstation:~/Dev/llvm/static-release/ ~/Dev/llvm/static-release/
# If you want to use trunk clang as the compiler,
# ninja -C static-release clang
# and add -f '+ bin' -f '+ bin/clang' -f '+ bin/clang++' -f '+ bin/clang-7' on the rsync command line
```

The libclang headers reside in `Dev/llvm/tools/clang` while built libraries are in `Dev/llvm/static-release`. Locate them with `CMAKE_PREFIX_PATH`.
```zsh
cmake -G Ninja -DCMAKE_CXX_COMPILER=/usr/bin/clang++ -DCMAKE_EXE_LINKER_FLAGS=-fuse-ld=lld \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_EXPORT_COMPILE_COMMANDS=On -DSYSTEM_LIBCLANG=On \
  -DCMAKE_PREFIX_PATH="$HOME/Dev/llvm/static-release;$HOME/Dev/llvm/tools/clang" -Brelease -H.
cmake --build release
# Built release/ccls whose path can be used to set emacs-ccls variable `ccls-executable`
```
