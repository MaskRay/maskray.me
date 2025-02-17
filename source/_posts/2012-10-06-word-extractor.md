---
layout: post
title: 自然語言處理之詞語抽取
author: MaskRay
tags: [algorithm, data structure, stringology, nlp]
---

多日之前看到了`Matrix67`的《互聯網時代的社會語言學：基於SNS的文本數據挖掘》，文中提到的方法是無監管的，而且無需詞典就能提取詞語，要素概括起來有兩點：詞的凝聚力，以及左右鄰字的信息熵。今天把這個方法實現了一下。

對於凝聚力，我的理解是可以用詞前後兩部分的`pointwise mutual information`來描述，比如對於“博物館”一詞，考慮“博”與“物館”之間，以及“博物”與“館”之間的`pointwise mutual information`，兩者取較小值作爲“博物館”這個詞的凝聚力。

<!-- more -->

左右鄰字的信息熵按照`Matrix67`原文方法計算。

計算`pmi`和信息熵都需要使用詞頻，直觀的想法是先枚舉詞的長度$len$，之後枚舉所有連在一起的長爲$len$的字數組作爲候選詞。計算詞頻需要用到類似Rabin-Karp算法的`string hash`，或者`trie`，或者一些`binary search tree`。

另一種實現方式是使用`suffix array`，以相同詞作爲前綴的後綴在`suffix array`中處於連續的一段，這樣一遍掃描就能依次得到每個詞的頻度，空間佔用較小。另外一個好處是共享同一個右鄰字的後綴也是連續的一段，一遍掃描可以得到每個右鄰字的頻度。之後翻轉字符串再做一次得到左鄰字的信息，或者把`suffix array`改造成`prefix array`。

## 實踐

模仿[BYVoid](http://byvoid.com/)對《笑傲江湖》進行了分析。

- 使用`poppler`中的`pdftotext`把pdf轉成txt。
- `./WordExtractor < in > out`，兩三秒就運行完了。
- 人工分析 `out`，設置信息熵以及 `pmi` 的閾值。
- 修改`analysis.rb`並執行`ruby analysis.rb < out`得到結果。

    令狐沖 自己 甚麼 咱們 嶽不羣 倘若 說道 師父 盈盈 田伯光 林平之 儀琳 如何 武功
    劍法 嶽靈珊 一個 長劍 左冷禪 任我行 如此 華山 突然 弟子 今日 他們 恆山派 向問天
    東方不敗 餘滄海 江湖 不知 華山派 登時 跟着 之中 笑道 出來 教主 心想 雖然 當真
    只是 心中 也不 魔教 二人 小師妹 此刻 劉正風

## 源代碼

[github gist](https://gist.github.com/3844497)
