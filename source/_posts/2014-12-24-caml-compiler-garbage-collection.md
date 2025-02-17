---
layout: post
title: ML编译器Caml Featherweight——垃圾收集
author: MaskRay
tags: [compiler, ocaml, ml]
---

#### 垃圾收集

Caml Light使用的垃圾收集算法可以参考@Sestoft94，结合了stop-and-copy和分代mark-sweep。我们的实现则使用了一个非常简单的方法，基于Schorr-Waite graph marking算法。该算法不使用栈，只需要在节点上分配极少量信息($\lceil\log_2{n+1}\rceil$ bits，其中$n$是该节点的指针域数目)。在一些资料中被称为pointer reversal。基本思想是在节点信息里维护一个域$s$，表示遍历过程中的访问次数，并把遍历过程中祖先节点下溯的指针翻转，再设置一个表示当前节点父节点的指针$p$以把祖先节点组织为一个链。
