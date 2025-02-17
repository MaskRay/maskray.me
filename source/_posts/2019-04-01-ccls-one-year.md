---
layout: post
title: C++ language server ccls一周年
author: MaskRay
tags: [c++, emacs, lsp]
---

2018年4月1日我寫了[ccls: a fork of the C++ language server cquery](https://maskray.me/blog/2018-04-01-c++-language-server-ccls)，宣告ccls誕生。如今一週年，有必要記錄下這一年的點點滴滴。

<!-- more -->

4月1日我fork出來，原因是我需要一個滿足自己口味的C++ language server。修改cquery幾個月，期間也曾學習過rtags ycmd clang_completer等clang-based工具，對如何打造language server頗有心得。由於和原作者理念差異，在cquery引入很多改動是掣肘的。因爲理念不合被取消commit權限，就決定自己新建一個項目。另外cquery作者也在聊天室裏提到他一直在等待clangd。而我的感覺是，clangd採用的並非最佳模型，或至少，不應該是用戶唯一的選擇。其實當時我也試探性想貢獻一下clangd的fuzzy matcher，發現想法不合做貢獻就要耽誤很多時間，很多地方的設計理念也不合。所以還是決定從cquery改，看如何闖出一番天地。當時覺得cquery已經很成熟了(看着它的star漲到超過1000，在LSP、emacs-china等地被人越來越多地討論，也會有很有成就感(<del>虛榮心</del>))，我另起爐竈也就是自娛自樂而已。

說幹就幹，從一個描述爲“Resurrection of ccls”的commit開始，3月31日提交了8個描述只有一個點的commits(它們改了116個文件！)。接下來在cquery的代碼庫上做了大量清理工作：刪除第三方庫、精簡代碼、刪除過度的抽象、合併拆得過碎的文件，當然還有改名、刪除waf構建系統、刪除無用blob、……自己用得舒服了再推己及人，希望別人也能用上<del>滿足自己的虛榮心。其實說穿了就是和cquery/clangd搶奪用戶</del>。精簡並不能改變用戶習慣，帶來用戶。我很快瞄準了cquery用戶的一個痛點：auto-index pipeline的穩定性。最大的問題是，保存文件後容易導致重複或丟失的references。如果`.h`和`.cc`沒有放在同一個目錄，更容易出問題。

這個組件我一直知道被設計得過分複雜，但我自己對這部分理解也不透徹。cquery把索引信息分成並行索引時用的`IndexFile`和加載到全局數據庫裏的`QueryFile`是個非常不錯的設計。一個文件修改，觸發索引，獲得`IndexFile`，合併回主線程的全局數據庫的過程其實很脆弱，出過很多問題。在交叉索引信息的基本單元(標識符，由Clang Unified Symbol Resolution表示)的表示從字符串改爲8字節hash後，pipeline中的幾個步驟已經沒有太大意義了。query.cc用的MergeableUpdate也是有問題的。能改善一點性能(indexer threads分擔少量main thread任務)，但cquery的全局索引更新模型是不滿足交換律的，這麼做也會在一些情況下會導致索引丟失，這裏還有個merge sort用的merge的bug，直到7月才修好。

五月初我把pipeline翻新了。初時內存佔用會稍高，但穩定性提升很多，不會發生保存多次導致索引確實或重複的問題。到9月已經非常穩定了。

等到逐漸有人關注了，有時候不得不回答一些討厭的問題：和cquery的區別是什麼、有cquery/clangd(clang官方項目)爲什麼還要弄ccls。我現在仍然討厭這類簡短而內容空洞問題(除非提問者能證明自己思考過了)。在此引用eglot作者João Távora的話，當別人問起有lsp-mode爲什麼還要再造一個client時，他回答：“You see, I added those paragraphs in the first place because some people ask, sometimes indignantly, why I write new things and don't contribute to an existing project. Apparently "because I want to" and "to have fun" aren't acceptable answers. But for me, they are still totally great answers, so if you are having fun writing lsp-mode.el I can only wish you well.”

ccls對於我的價值：

* 最重要的是給了我好好學習clang的理由。2017年給llvm全家桶貢獻了幾個patches，到年底依然只有10個不到，後來慢慢在這個領域找到興趣。維護ccls能逼迫我認真鑽研clang工作機制，以及多灌水(commit)。到現在，在clang裏灌了69個，llvm其他地方都加起來灌到400多。
* 迫使我瞭解Language Server Protocol各個方面的細節。說實話，我不喜歡這個協議，很多地方設計得過於臃腫、浪費，爲了所謂的<del>VSCode</del>兼容性做了很多讓客戶端、服務端都痛苦的設計。其實無可厚非，這個協議本來就是Microsoft用於VSCode的，由於用戶的請求才被開源出來，這和LLVM、Rust那些社羣的管理模式是很不一樣的。話雖如此，比起之前幾代產品還是好不少，匯聚起來了一個真正幫用戶改進編程體驗的鬆散的社羣。
* 一個展示平臺。
  + 被隊友、對手、主辦方(??)自發關注到(尤其是DEF CON 26 CTF上)。在LLVM Dev Meeting上一些閒聊的人談到。
  + 有廠商想給我donation但我沒要。
  + 有廠商想用ccls做離線分析。
  + 被spacemacs doom-emacs採用。進了v8的`.gitignore`
  + 有人給AOSC、Mac OS X Homebrew、FreeBSD、Nix、Void Linux、Fedora等打包。另外還有我自己給Arch User Repository貢獻的。
  + 被GNU make維護者Paul Smith使用了。help-make是我訂閱過最久的郵件列表，看完過Managing Projects with GNU Make。OI比賽時我也是用GNU make編譯程序、運行測試的。這個帶我的感覺很不一樣：一直以來寫的軟件影響着你的人，現在也被你影響了。
* 有朋友真的會關心我這個東西做了怎麼樣，和我探討實現。

2018年7月初從“First draft: replace libclang indexer with clangIndex”開始，用Clang C++ API重寫了indexer。接下來把`project.cc`中用的libclang `compile_commands.json`解析換成clangTooling。隨後用Clang C++ API重寫了completion/diagnostics(從clangd和ASTUnit學了很多)。最後終於把所有libclang組件移除。

用了libclang很久，發現了它的很多問題。它在早年clang API不夠穩定時確實立下了功勞，最初應該是Apple Xcode用，後來催生了clang\_completer、rtags、irony-mode、iycmd clang_completer等工具。但它的限制也很明顯，Clang C++ API茫茫多，libclang作爲穩定的C API包不完API，因此它只實現了一些較高層次的API，某些API如(find declaration, find definition)實際上包含了很多邏輯，如果使用場景和API設計者不同，就會發現很不靈活。

10月把以前移除的`textDocument/documentLink` `textDocument/rangeFormatting`等小衆功能都加了回來，在功能上也不輸了。

最近幾個月都挺頭疼的，日均一個新增issue。不少issue提問的人隨手一寫，好在最近引入了GitHub issue template情況有所改觀，但依然能注意到非常多不懂腦筋的人，讓我很反感。我對於issue管理比較粗暴，通常回覆一下就關閉，不會像一些其他項目會等一段時間，關閉時再回覆一句話撫慰提問人。我不願意浪費這些時間。別人也許會這麼評價：“It did spawn a fork: https://github.com/MaskRay/ccls, which is actively developed, but the lead is explicitly not interested in supporting community requests, which IMO bodes well for its longevity.”我覺得這挺貼切。我也很感謝即時的反饋，比如某個改動造成了某某發行版編譯失敗或運行錯誤，現在都能較爲迅速的得到反饋及時改正。有時用戶會注意到clang bugs，調試它們的過程也挺有趣。

一些有趣的issues：

* https://github.com/MaskRay/ccls/issues/30 `-fno-rtti`導致的ABI mismatch。終於查明了`aur/ccls-git`會segfault的原因。有個相關的commit讓我成了GCC contributor:)
* https://github.com/MaskRay/ccls/issues/71 GCC 5 dual ABI導致的ABI mismatch。
* https://github.com/MaskRay/ccls/issues/186 調整wiki結構。新的結構確實更清晰了，不過我吐嘈剛改完的樣子有些羅嗦……現在比較簡潔了。很多項目的文檔我都覺得太長不想讀。
* https://github.com/MaskRay/ccls/issues/355 根據`lib{clang,LLVM}*.a`獲得`.so`。

20180513的“Congratulations to Tea Deliverers”(恭賀Tea Delievers再度入圍DEF CON CTF決賽)，20180521的“Maker Faire”，20180731的“Real World CTF”(給本來宣傳力度就很大的活動造勢)。

