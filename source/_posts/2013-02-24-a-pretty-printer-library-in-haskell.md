---
layout: post
title: TUNA技術沙龍及A Pretty Printer Library in Haskell
author: MaskRay
tags: [haskell, presentation]
---

## 技術沙龍準備

三天前王康提議開學前的週末搞個活動，之後敲定由Cheer Xiao、heroxbd、Aron Xu和我做演講。得到消息很詫異，因爲以上幾位都有豐富的系統管理經驗，我就不知道能幹什麼了……但畢竟被逼上梁山了，最後Cheer Xiao讓我講Haskell。

<!-- more -->

我考慮了幾種講法：

- 純粹講語法。大家可能不會有什麼興趣。
- 和其他語言做比較，找設計獨到的地方。以前做過類似的事，不太想重複。
- 找一個function pearl。思考再三，覺得以前看過的Philip Wadler的pretty printer可能是最合適的了，用不到Haskell很多特性，需要瞭解的語法很少就能上手。在開頭簡單地過一下語法中的一個子集就能滿足後面講pretty printer的需求。

## 幻燈片工具jmpress.js

要是以前我肯定會糾結選擇org-mode還是用pandoc風味的markdown，生成`.tex`文件後還要折騰`xelatex`的各種宏包，特別是得處理`beamer`的很多麻煩問題。由於我對`LaTeX`不熟，用起來勢必捉襟見肘。儘管`LaTeX`號稱排版功能強大，但我這樣的入門用戶是用起來就帶腳鍊跳舞。另外幻燈片缺乏顯示以外的其他控制功能。索引、進度條、現代化一點的動畫等，我知道`LaTeX`當然都能做，但做起來肯定不方便，它使用的語言看着就讓人倒胃口了。

現在學了點網頁前端領域的皮毛，`html`、`css`、`javascript`，一個個就其本身來說，都是我沒法接受的。但引入了一層抽閒：各種現代化的模板引擎和`coffeescript`，`html`這些東西就不再讓人倒胃了。

我選擇了[jmpress.js](http://jmpressjs.github.com/jmpress.js/)作爲演示工具。內容用`jade`編寫，樣式用`stylus`生成，腳本使用`coffeescript`。

構建工具選用`grunt-0.4.0`，其`grunt-contrib-watch`插件能自動檢測源文件是否修改並更新相關目標文件。源文件保存後，在瀏覽器裏刷新就能立刻看到結果。而用`beamer`寫的幻燈片，內容多了之後就得等上半分鐘，很影響體驗。

使用了`coffeescript`，幾行代碼就能實現其他幻燈片工具裏難以實現的功能：

- 在底部添加進度條，用彩虹色表示時間
- 項目列表（還包括其他元素）的增量式顯示
- 操作DOM實現內容的複製

我把這次演講當成了前端知識的一個極佳鍛鍊機會。

## 現場

今天14:00活動開始。來現場的人好多……遠遠超過預期了，TUNA的潛水會員還真有這麼多啊。碰到了好多久仰卻從未謀面的人如heroxbd、smilekzs，神祕的curimit，高中+大學學長Gene……還有好多熟人就不講了啊，不認識的同學還是不認識……以後是不是考慮結束後聚餐什麼的。

首先Cheer Xiao和heroxbd給大家介紹了Google Summer of Code的事項，然後Debian Developer Aron Xu介紹了systemd，至於他們講了什麼……很抱歉，我一直在調css，設置字體大小，臨到沙場才發現了好多之前沒發現的bug，沒怎麼認真聽……抱歉啊。

最後的遺憾是大家沒有合影留念。

## TUNA

另外還有很多事情懵懵懂懂。在TUNA打了一年醬油卻對TUNA的前生今世沒有什麼瞭解，包括但不限於活躍成員、目標計劃、任務、指導教師、經費、活動場地、管理服務器、校內地位等。

平時在郵件列表裏經常能看到“出口 QoS 策略討論”、“rootkit 事件”等高深問題，可惜都沒有找到老會員問清。

先偏一下話題。在一些問答社區裏提問時，提問前先要自己搜尋相關資料做嘗試，提問時要提供間斷、具體、清晰的描述，熱心回答者有疑問時要注意補充說明，任務完成後還要注意總結。作爲TUNA郵件列表用戶，我們是不是也應該注意這些？

## A Pretty Printer Library in Haskell

[HTML幻燈片](/static/2013-02-22-a-pretty-printer-library-in-haskell/index.html)

[tarball下載](/static/a-pretty-printer-library-in-haskell.tar.gz)
