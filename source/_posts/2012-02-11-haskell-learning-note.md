---
layout: post
title: Haskell學習筆記
author: MaskRay
tags: [haskell, ocaml]
---

機緣巧合，寫了兩個用到`Parsec`的`Haskell`程序。一個是[Untyped lambda calculus](https://github.com/MaskRay/hsnippet/tree/master/calculus/untyped)，另一個是 [po2db](https://github.com/MaskRay/po2db)。`adam8157`對神器`Pandoc`大爲讚賞，這也是`Parsec`的應用。`Parsec` 也有[不少其他語言的復刻版](http://www.haskell.org/haskellwiki/Parsec#Parsec_clones_in_other_languages)，不過不少語言都缺乏`Haskell`自定義中綴操作符的功能，實現一個parser所寫出來的代碼可能會冗長很多。

<!-- more -->

一直以來對`Haskell`的record沒有first-class syntax耿耿於懷，直到發現了`data-lens`，結合`Template Haskell`，對`record`的操作也能像一般的函數那樣，方便了許多。

`OCaml`也是門不錯的語言，它的實現(似乎只有一個)性能也不錯，開發工具也非常齊全(甚至有調試器，相比之下，`ghci`顯得非常初級)。學語言最好的方式還是動手，網上的99 Prolog problems不錯，也有Lisp版的，我做了不少，放在[這裏](https://github.com/MaskRay/99-problems-ocaml)。

之前make+m4生成的網站雖然好用，但顯得太“野蠻”了，正好有個Haskell的模仿Jekyll的項目[Hakyll](http://jaspervdj.be/hakyll/index.html)，就遷徙到Hakyll吧。
