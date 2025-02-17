---
layout: post
title: 用Makefile搭建博客
author: MaskRay
tags: [makefile, shell, website]
---

## 緣起

我幾乎不會 HTML，不會使用 Perl、Python 的庫，不會 PHP 和 Ruby，不懂 JavaScript，但也想搭個博客。

### 嘗試 WordPress

首先選擇的是 WordPress，基本上是照着 [這篇文章](http://forum.ubuntu.org.cn/viewtopic.php?f43&t317219)
做的，在 tusooa 和 phoenixlzx 的大力幫助下，註冊了個 .tk 的域名和一個免費服務器，基本上算是搭起來了。不過有些不滿意的地方，最討厭的是它不是用標記語言寫的，有些繁瑣。

<!-- more -->

### 嘗試 DokuWiki

MeaCulpa 推薦使用 DokuWiki，看了它的一些介紹，覺得這個標記語言還不錯。但要讓它變成 blog，還是有不少工作要做。要裝一個名爲 BlogTNG 的插件，我裝了之後就只顯示空白頁了，用 findbad.php 也找不出原因。也用了 lainme 的可以正常工作的
BlogTNG 插件，但在我這兒就是不行。無奈之下還是放棄了。

## 自己寫 Makefile

作爲一個 Emacs 用戶，得用 Org Mode 做標記語言，其實通過配置 org-publish 應該也能實現的。但我不大會 Elisp，也不熟悉 Org Mode，所以要找其他工具來做發佈的任務。我發現了一個叫做 blorgit 的博客系統，跟着教程一步步做，默認頁面不怎麼樣。而且它是 Ruby 寫的，我配置不來，於是這個也放棄了。最後選擇了用 Makefile 自己寫。

### 基本思路

.org 存放實際寫的文章，用於生成 .html，首頁 index.html 是文章列表。還要使用第三方的評論系統。

### 設計

目錄設計如下：
- 創建目錄 2011/07/12/ 來表示當天寫的文章（當前還有其他日期，這裏使用 2011/07/12/ 只是爲了方便闡述）
- 2011/07/12/ 下創建 blogging-with-makefile.org ，用於生成 blogging-with-makefile.html
- 2011/ 下創建 titles-07 緩存2011年7月所有文章的標題

文章部分就用如上形式儲存。所有文章都有共用的頁頭、頁腳（我不知道準確的術語），比如本頁上方的的鏈接和下方 disqus 的評論系統，它們分別儲存爲 header.org 和 footer.org 。當要把某篇文章的 .org 導出爲 .html 時，在前面包含 header.org ，後面包含 footer.org 。
disqus 要求每個網頁要包含幾個特定的變量，所以 footer.org 需要做下修改再包含。

首頁是所有文章的列表，需要用腳本生成出來，還要注意的是它不需要評論系統，所以不包含 footer.org ，

### Makefile

Makefile 的主要規則如下：

- 2011/titles-07 依賴 2011/07/ ，如果該目標過期（需要重建），
  那麼嘗試重建 2011/07/12/ 下所有日期的所有 .html 。然後把這些 .html 的標題抽取出來，
  寫入 2011/titles-07
- %.html 依賴 %.org 以及頁頭頁腳，需要用 Org Mode 根據 .org 生成 .html。
  這個規則也對首頁 index.org 有效。

當 2011/07/12/ 創建後， 2011/titles-07 就會過期，重建
2011/07/12/ 下所有日期的所有過期的 .html ，並且生成新的 2011/titles-07 ，進而
重建 index.org ，最後重建 index.html 。

注意到這種設計把目錄作爲了依賴，我們要知道目錄的修改時間是不會因爲被它包含的文件的修改而改變的。所以當 2011/07/12/blogging-with-makefile.org 修改之後， 2011/07/12/
和 年/月/ 的修改時間都是不會變化的。這樣就會導致 2011/07/12/blogging-with-makefile.org
修改後，執行 make 時 2011/07/12/blogging-with-makefile.org 不會得到更新。

爲了處理這種情況，我們需要讓目錄隨着被它包含的文件的修改而更新修改時間。我想到的辦法是用 inotify，當 2011/07/12/blogging-with-makefile.org 修改之後，自動 touch 年/月/ 。
