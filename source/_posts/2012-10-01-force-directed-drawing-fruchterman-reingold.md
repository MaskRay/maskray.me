---
layout: post
title: Force-directed算法(1)——Fruchterman-Reingold
author: MaskRay
tags: [algorithm, graph drawing]
mathjax: true
---

## 簡介

Graph drawing即根據頂點和邊的拓撲關係，將這張圖展現出來。很明顯，表現的形式種類是非常多的，如果精確到每個頂點的座標，那麼方案有無窮多種。Graph drawing 目標是畫出一張美觀的圖的佈局來。美是個見仁見智的概念，何謂美？“每個本質在於旺盛的生命力，美的形象能夠感染人的情感和鼓舞人心”，展現出來的圖是否美每個人的看法可能都不一樣，但有一些通用的要素是大家一般共同認可的，比如邊的兩端不能靠得太近也不能離得太遠，相交邊的數目要儘量少，有對稱性等等。

<!-- more -->

Graph drawing的應用大家應該不陌生，常用的工具Graphviz就是這個領域一款成熟的開源軟件。

佈局算法中一類通用、擴展性好、易於實現的方法是 force-directed drawing algorithm。這類算法在 _William T. Tutte. How to draw a graph. Proc. London Math. Society, 13(52):743–768, 1963._ 時已經有一個雛形，
_Eades, Peter (1984). "A Heuristic for Graph Drawing". Congressus Numerantium 42 (11): 149–160._ 中加入了彈簧的因素，force-directed算法就成型了。

## Fruchterman-Reingold算法

下面討論Fruchterman-Reingold算法，每個頂點在展現的圖中表現爲一個質點或者球(下面爲方便期間犧牲嚴謹性稱之爲頂點)，任意兩點間有斥力，有邊關連的兩點間有引力，引力可以想像成由頂點間的彈簧提供的，也就是說每一條邊都表示一根彈簧。精確地，引力：

$f_a(d) = d^2/k$

斥力：

$f_r(d) = -k^2/d$。

另外假定理想狀況下所有畫出來的邊的長度都是確定的，上式中的 $k$ 就是描述邊最優長度的常數，$d$ 是兩點間距，可以取每個頂點平均分得的畫布面積的算數平方根的常數倍。

算法初始時給每個頂點分配一個隨機位置(可以組成圓形，可以是網格，也可以是其他佈局算法的輸出結果，但不能排在一條直線上(想一想爲什麼))，核心是個迭代過程，計算出所有點對間的斥力，再對於每個頂點，考慮和它關連的彈簧對它產生的引力。每一輪迭代枚舉每個頂點，根據它受到的合力向量讓它的位置發生改變。當所有頂點位置不發生改變或者迭代次數超過預設的某個閾值後算法結束。

僞代碼如下：

    f_a(d) = \d -> d^2 / k # attractive force
    f_r(d) = \d -> k^2 / d # repulsive force
    for i in [0..iterations)
      u.displacement = zero vector
      # regard repulsive forces
      for (u, v) in E
        d = u.pos - v.pos // vector
        u.displacement += d / |d| * f_r(|d|)
      # regard attractive forces
      for (u, v) in E
        d = u.pos - v.pos
        u.displacement += d / |d| * f_a(|d|)
        v.displacement -= d / |d| * f_a(|d|)
      # tune positions
      for u in V
        u.pos += u.displacement

讓頂點位置發生改變的時候可以用上類似模擬退火算法中的溫度參數，限制位置的改變量。最後還要注意放縮佈局以適應畫布。

一輪迭代的時間複雜度爲：$O(n^2+m)$，其中 $n$ 和 $m$ 分別爲點數和邊數。其中 $O(n^2)$ 的斥力計算是可以優化的，方法類似 n-body 問題中的 Barnes-Hut simulation，即把遙遠頂點團的斥力合力視作頂點團質心處的一個質點，可以用quad-tree、k-d tree之類數據結構優化。

注意到斥力的引入的一個緣由是防止坍塌到一點這種退化情況發生，所以斥力其實是夠用就好的，所以可以設置一個和 $k$ 相關的常數 $R$，直接忽略距離超過 $R$ 的頂點。所以類似 n-body 問題，還可以用 spatial hashing 等方法優化。

## 效果

執行腳本：

```ruby
##!/usr/bin/env ruby
n = 8
puts "#{n*n} #{n*(n-1)*2}"
1.upto n-1 do |i|
  n.times do |j|
    puts "#{n*i+j} #{n*i+j-n}"
  end
end
1.upto n-1 do |j|
  n.times do |i|
    puts "#{n*i+j} #{n*i+j-1}"
  end
end
```

生成8*8的網格 ``/tmp/i``，採用 von Neumann neighborhood。

執行``scripts/gen-svg.rb -i 2000 < /tmp/i | display svg:-``迭代2000次計算處這樣一張圖：

![](/static/fruchterman-reingold-mesh.svg)

## 源代碼

[在github上](https://github.com/MaskRay/ForceDirected)
