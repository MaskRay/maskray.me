---
layout: post
title: ML編譯器Caml Featherweight——pretty printer
author: MaskRay
tags: [compiler, ocaml, ml]
mathjax: true
---

## 其他

项目中我们还实现了一些辅助工具。

### Pretty printer

Pretty-print指的是对文本文件(特别是标记语言和程序代码)采用语法高亮、调整换行空格、调整字体等手段使之易于阅读。项目中实现了一个pretty printer，用于生成进行换行空格排版调整后的代码和调试信息。

下面只考虑进行换行空格排版调整的pretty printer。Haskell中的pretty printer主要有两大流派，其一是John Hughes设计，由Simon Peyton Jones修改的，跟随GHC一起发行，称为Hughes-PJ库；其二是Philip Wadler在The Fun of Programming中设计的pretty printer@Wadler，后由Leijen修改，添加了对齐等扩展得到的，称为Wadler-Leijen库。

@prettiest-printer提出了另一种方法，是对Wadler/Leijen的一种改良。原始的Hughes-PJ和Wadler/Leijen对于并集文档，为了注重性能，使用了贪心算法进行选择。但这样得到的文档不一定是最短的。这一方法以行为单位搜索最短表示。我对它进行了改良，使得能保证返回适应宽度的文档。

@Swierstra04提出了一种线性算法，@Swierstra09提供了该算法的不依赖lazy evaluation的版本。我实现并扩展了这种方法，引入了对齐操作。

    h=5;a=let b=4;c=5 in (case 4 of <3> a -> 5;<4>->6;<4>-> case 3 of <2> A->2 * (let g = (let h = 5 in 5) + 8 in h)); h=6
