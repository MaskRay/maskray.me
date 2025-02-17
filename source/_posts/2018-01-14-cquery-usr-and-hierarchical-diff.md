---
layout: post
title: cquery USR優化和分層diff
author: MaskRay
tags: [emacs, lsp]
---

* [使用cquery：C++ language server](2017-12-03-c++-language-server-cquery)
* [cquery最近改動與libclang.so一字節補丁](2017-12-25-cquery-updates-and-libclang-one-byte-patch.md)

<!-- more -->

## cquery最近改動

* 原作者創建了organization，我據此把幾個插件分離了：[emacs-cquery](https://github.com/cquery-project/emacs-cquery/)和[vscode-cquery](https://github.com/cquery-project/vscode-cquery/)，等候吉日再來一次`git filter-branch --index-filter`減小主[repo](https://github.com/jacobdufault/cquery/)的體積
* 檢索Doxygen註釋標記，及可選的普通(行註釋`//`塊註釋`/**/`)。在initialization options裏配置`"enableComments": 2,`
* cache文件(`cacheDirectory`目錄下的文件)原來爲JSON，現在支持序列化成MessagePack，請在initialization options裏配置**"cacheFormat": "msgpack",`
* 感謝bennyyip添加了[archlinuxcn/cquery-git](https://github.com/archlinuxcn/repo/tree/master/cquery-git)，Arch Linux用戶要體驗cquery更加方便了
* 光標處符號的類型信息改進了，根據romix的提議，對於函數，找出函數類型和返回值類型字串形式第一個不同的位置，多半是插入函數名的位置。但參數名插入仍然困難，都怪C的inside-out類型表示語法

參見wiki中的[Initialization options](https://github.com/jacobdufault/cquery/wiki/Initialization-options)，邀請大家多編輯wiki爲後人提供方便。

![lsp-ui-doc](/static/2018-01-14-cquery-usr-and-hierarchical-diff/lsp-ui-doc.jpg)

## Unified Symbol Resolution優化

Clang Index使用Unified Symbol Resolution標識程序中的function/variable/type等。cache文件中一個函數可能存儲爲這樣：
```json
{
  "id": 0,
  "is_operator": false,
  "usr": "c:@FT@>2#T#pTMakeUnique#P&&t0.1#*t0.0#",
  "short_name": "MakeUnique",
  "detailed_name": "T *MakeUnique(Args &&... args)",
  "kind": 12,
  "declarations": [],
  "definition_spelling": "4:4-4:14",
  "definition_extent": "4:1-6:2",
  "base": [],
  "derived": [],
  "locals": [],
  "callers": ["2@14:3-14:13", "2@15:3-15:13", "2@16:3-16:13"],
  "callees": []
},
```

對於C++程序，嵌套幾個namespace、class後USR就會很長，佔用不少空間。原作者提起clangd用了SHA-1 (`std::array<uint8_t, 20>`)來省空間，我感覺160-bit太浪費了，想找個64-bit non-cryptographic hash，經quininer推薦選取了SipHash 64-bit，能用`uint64_t`代替原來的`std::string usr`，省空間。

<del>悲傷</del>
```c++
  const uint8_t k[16] = {0xd0, 0xe5, 0x4d, 0x61, 0x74, 0x63, 0x68, 0x52,
                         0x61, 0x79, 0xea, 0x70, 0xca, 0x70, 0xf0, 0x0d};
  (void)siphash(reinterpret_cast<const uint8_t*>(usr), strlen(usr), k, out, 8);
  return ret;
```

[rapidjson](https://github.com/Tencent/rapidjson)也能序列化`int64_t/uint64_t`，而不會因爲使用了`double`導致精度丟失到53-bit。JavaScript的Number類型真是一個陷阱。

用hash做主鍵，在這裏非常合適。因爲假使兩個鍵衝突，觀察到的後果也就是兩個變量/函數(他們的數量遠多於類型)合併了，所有功能都能照常使用。不會有blockchain hash衝突的災難性後果。

在LLVM上觀察cache空間佔用，`stat -c %s .vscode/cquery_cached_index/**/*.mpack | awk '{s+=$1}END{print s}'`，從720055341字節降到265784325字節。

根據khng300索引Linux kernel的結果：

```
# QueryFunc/QueryFile/...
db.funcs.size() = 346583
db.files.size() = 25553
db.symbols.size() = 2891931
```

## 層級diff

因爲性能問題，cquery當前只在用戶保存文檔時調用libclang重新索引用到當前文件的translation units，對於`.cc/.cpp`通常只會編譯一個translation unit。打開文件後進行修改，文件的buffer版本就和索引時版本不同了。類型/函數/變量的交叉引用信息仍爲索引時的版本，怎麼對應到buffer版本呢？index版本和buffer版本的映射是雙向的：

* buffer到index。用戶在buffer某處使用definition/references等功能，需要把buffer光標位置映射到index版本，查找符號信息
* index到buffer。對於hover/references等功能，找到一個符號所有出現的位置後，需要把index映射到buffer版本，顯示出來

對於一個編輯中的文件，很可能無法編譯通過，甚至難以保證能語法分析成功，我今天加了一些文檔diff算法，啓發式地處理這個問題。以按buffer 光標行列找index對應行列爲例。

* 先用Paul Heckel's diff algorithm找出index與buffer的映射關係，實質上把文檔分塊了
* 對於一個行文本，用Eugene Wimberly Myers' O(ND) diff algorithm找塊間最相似的行
* 對於行文本和列位置，在找到的最相似的行中找最合適的列

這樣就把index的一個行列位置轉換到buffer的行列位置。

### Paul Heckel's diff algorithm

大意是如果一個行在index中只出現一次，在buffer中也只出現一次，可以看作一個錨點，認爲buffer中出現的行號映射到index中出現的行號。

之後處理錨點鄰接的行，如果在index和buffer中相同，那麼也認爲buffer中出現的行號到index出現的行號有一個映射關係。據此把所有鄰接行都標記上映射關係。對於這種類型的flood fill，只需要從上到下掃描和從下到上掃描兩遍文檔。

### 塊間找最相似的行

如果詢問的buffer行`i`恰好有個到index的Paul Heckel映射關係`buffer_to_index[i]`，直接返回。否則就上下找到最接近的含有映射關係的行`up,down`，在映射後的區間`[buffer_to_index[up],buffer_to_index[down]]`內用edit distance找和`buffer[i]`最相似的行`j`。

### 把buffer第i行第c列映射到index第j行最優列

再用一次edit distance，在`index[j]`中找一個列`w`，使得`align(buffer_line[0:c], index_line[0:w]) + align(buffer_line[c:], index_line[w:])`最小。可以認爲index行的第`w`列即對應buffer行的第`c`列。這實際上是Hirschberg's algorithm的一步。

取到最小值的可能有多個列，我用了一個簡單的方法。一般詢問是關於buffer一個區間對應index中的哪一個區間。對於區間起點，我希望映射後的位置儘可能靠後；對於區間終點，希望映射後的位置儘可能靠前。

實現了以上buffer到index的雙向映射後，可以用index版本給正在編輯中的buffer版本提供信息了。效果見下圖，編輯了光標處的函數名，`textDocument/documentHighlight`依然可用：

![documentHighlight](/static/2018-01-14-cquery-usr-and-hierarchical-diff/diff.jpg)

## 其他

Emacs用戶請關注這些repo和issue：

* [emacs-cquery](https://github.com/cquery-project/emacs-cquery)，添加到[Melpa](https://github.com/melpa/melpa/pull/5235)
* [lsp-mode](https://github.com/emacs-lsp/lsp-mode)
* [lsp-ui](https://github.com/emacs-lsp/lsp-ui)
* 對於spacemacs用戶，請關注我呼籲[支持lsp、添加cquery到c-c++ layer的issue](https://github.com/syl20bnr/spacemacs/issues/10134)

Vim用戶還沒有很好看的UI，需要人拯救。

另外，[把字符單位從UTF-16換成Unicode codepoint](https://github.com/Microsoft/language-server-protocol/issues/376)，Microsoft和JavaScript受害者真是對UCS-2/UTF-16孜孜不倦。感謝[Riatre的分析](https://github.com/jacobdufault/cquery/issues/57#issuecomment-357021182)。這個問題修不好，我們在Emacs或VSCode裏使用emoji會有很大阻礙。

cquery中Language Server Protocol中物件的序列化仍挺麻煩的。TypeScript中interface可以繼承，我也在Emacs lsp-mode裏看到了有些`TextDocumentIdentifier`帶有子類`VersionedTextDocumentIdentifier`的版本字段。這個在C++裏序列化就不方便了。

如果要進一步優化啓動時cache加載速度，需要考慮lmdb之類存儲。

<https://microsoft.github.io/language-server-protocol/specification#versionedtextdocumentidentifier>

`interface VersionedTextDocumentIdentifier extends TextDocumentIdentifier`
