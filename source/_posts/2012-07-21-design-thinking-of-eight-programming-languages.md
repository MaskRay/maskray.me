---
layout: post
title: 8門編程語言的設計思考
author: MaskRay
tags: [haskell]
---

今天參加了SHLUG的[月度技術分享會](http://www.shlug.org/?p=1550)，見到了三個同學的同學(呃~精確描述的話可能得寫成這樣：其中一個是同學A的同學，兩個是同學B的同學)。

我就不同編程編程語言的設計思考爲主題做了一個分享。之前在學校裏做過類似的分享，這次把它搬到SHLUG上來了，對之前的幻燈片又修修補補，增添了一些內容(把之前Prolog、OCaml紙上談兵的部分加了些代碼，豐富了內容……當前，也可以把整張幻燈片看作紙上談兵)。

之前“80分鐘8語言”這個標題感覺有些誤導人，畢竟讀者的第一反應很可能是認爲這是在介紹語法，第二反應是80分鐘甚至無法把一門語言的語法給講清楚，如何能把八門語言交代清楚。而且之前在學校分享時和另一位同學肖騏一共講了兩個半小時的樣子，遠遠超出了80分鐘。題目的用語想了一會兒，決定改爲設計思考([design thinking](http://en.wikipedia.org/wiki/Design_thinking))。

<!-- more -->

關於8門語言的選取，Perl是用來吐槽的，就不足爲奇……imperative、class-based object-oriented的語言已經氾濫了，C系(語法上的，C、C++、Java、C#這些)沒什麼亮點，對主流編程思想的“毒害”也深，可以稱道的地方不多，沒什麼可談的(而且都不會……)Objective-C另一半繼承自Smalltalk，Smalltalk這個史上第二門object-oriented語言對後世影響還是非常深的，雖然只看了相關的一本書，但還是冒昧地決定談及它。

Lisp系中自然得選一個，從應用多寡來看選項其實只有三種：Common Lisp、Scheme、Clojure，第三個一無所知，第一個看了半本書覺得設計上歷史包袱較重，所以就選設計更爲清晰的Scheme了，而且minimalism也投我所好。Lua風格亦是minimalism，而且是prototype-based，所以也列進去了。class-based中Ruby設計實在是優秀，所以也選了(可惜那些基於jvm的如Clujure、Scala、Groovy都沒接觸過)。

functional的首選自然是Haskell，但非pure的也得選一個，於是自然想到了經典的ML系列，選取了接受程度最大也是最實用的OCaml。這樣一來還缺一個dynamic typing的functional，所以就選了Erlang。另類範式logic的只有Prolog可選了。遺憾是沒接觸過Scala和Forth，後者concatenative必有可圈點之處，順便也能吐槽shell設計得如何之爛。

沒有學過APL(或者後裔J)從而沒有把它列進去着實是個遺憾。Joel Moses有云："APL is like a beautiful diamond - flawless, beautifully symmetrical. But you can't add anything to it. If you try to glue on another diamond, you don't get a bigger diamond. Lisp is like a ball of mud. Add more and it's still a ball of mud - it still looks like Lisp."

參加活動的朋友二十有餘，初時我還有些緊張，而後卻是越講越起興，加者以爲時間不能超過1小時，而內容挺多的，語速較快。最後大概講了一個半小時。

幻燈片還是挺翔實的，2000多字……

[幻燈片下載](/static/design-thinking-of-eight-programming-languages.pdf)
