---
layout: post
title: 用Pike's VM實現的非回溯正則表達式引擎
author: MaskRay
tags: [algorithm, regex]
---

Parser之外的部分參考http://swtch.com/~rsc/regexp/regexp2.html ，代碼都模仿自http://code.google.com/p/re1/source/browse 。注意到正則表達式是operator-precedence grammar，可以用一個擴展的Shunting-Yard算法來解析，其中用了一些特殊構造處理後綴操作符和括號。

<!-- more -->

## 實現

[github上](https://github.com/MaskRay/Regex)

## 推薦閱讀

[Parsing Expressions by Recursive Descent](http://www.engr.mun.ca/~theo/Misc/exp_parsing.htm)
