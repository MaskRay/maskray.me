---
layout: post
title: 異於Make的另一種構建系統
description: 不同於傳統構建系統(如Make)的系統，很多源文件修改場合下時間複雜度低於線性
author: MaskRay
tags: [makefile, build system]
---

## 以Make爲例的傳統構建系統

傳統的構建系統如Make，用戶指定項目中所有目標文件(非源文件)的產生方式，儲存在makefile文件中。

考慮如下的僞makefile：

```makefile
main: a.o b.o
a.o: a.c main.h
b.o: a.c main.h
```

<!-- more -->

請忽略Make的缺省規則(main的實際依賴會有所不同)。例子中也省略了recipe(`info '(make) Rule Syntax'`)。

每次需要構建整個項目的終極目標或者中間目標時，比如執行`make main`，Make會檢查
構建`main`的規則，發現`main`依賴`a.o`和`b.o`，而`a.o`依賴`a.c`和`main.h`，`b.o`依賴`b.c`和`main.h`。假設`a.c`修改了，那麼`a.o`和`main`都會“過期”，更新`main`的方式是先重建`a.o`再重建`main`。

依賴關係可以表示成這樣一張圖(directed acyclic graph)：

![依賴關係的 directed acyclic graph](/static/alternative-build-system/make.png)

Make這一類傳統構建系統每次執行時都會根據配置文件(比如makefile)中記錄的規則建立所有文件的依賴關係，在內部產生一個圖(類似上圖)。然後取出最終重建目標的所有直接、間接的依賴，按照拓撲序的逆序依次考慮每個節點對應的文件，判斷是否過期，如果過期則執行一條recipe重建這個文件。

而判斷目標是否過期的方式則是用`stat(2)`之類的系統調用獲取目標及其依賴的時間戳，判斷目標的時間戳是否小於依賴的。

重建一個目標執行的重建操作數等於目標的依賴樹中“過期”節點的個數。(實際上make可能會重啓，導致重建操作數變多。這裏不考慮這些)看上去非常優異，但是獲取時間戳操作的次數是和依賴樹的大小同階的，達到了線性的時間複雜度。

## 另一種構建系統

### 判斷是否過期

傳統的構建系統對文件的修改狀態實際上一無所知，所以每次執行重建操作都得通過檢查所有文件時間戳的方式來瞭解文件是否被修改，從而判定哪些目標需要重建，瓶頸就在於這些時間戳獲取操作。

假如有一種方式能夠得到文件的修改通知，那麼我們就能省去這些時間戳獲取操作。

![依賴關係的 directed acyclic graph](/static/alternative-build-system/alternative.png)

考慮上圖，我們修改了`a.c`，`a.c`、`a.o`、`main`都“過期”了。可以看出，變動`a.c`之後，只有它的祖先節點對應的文件會變成“過期”狀態，其他文件仍然會是新的。

設想這樣一個構建系統，以daemon形式運行，讀取配置文件產生依賴圖，維護一份文件是否“過期”的列表。用戶執行構建操作時，客戶端會向daemon傳遞構建任務，daemon根據這份列表判斷目標是否需要重建。注意，傳統的構建系統是通過檢查時間戳來判斷是否需要重建的。

我們需要根據文件的修改信息來維護這份列表。當有文件發生修改時，在DAG中找到這個文件對應的節點，將它及其所有祖先標記爲“過期”。

### 高效執行重建操作

能判斷目標是否過期還不夠，我們需要知道如何重建它。顯然不能像傳統構建系統那樣遍歷目標所在的子樹。

一種解決方案是讓directed acyclic graph**只包含“過期”文件對應的節點**，那麼一開始這張圖爲空，如下圖：

![空 DAG](/static/alternative-build-system/alternative-0.png)

灰色節點實際上不在依賴圖中，這裏顯示出來是爲了展示出整張圖的全貌，即如果所有文件都過期時圖的形態。另外注意一下箭頭方向，讀完後可以想一下這麼做的理由。

修改了`a.c`之後，更新`a.c`、`a.o`、`main`的過期狀態：

![修改了`a.c`之後的DAG](/static/alternative-build-system/alternative-1.png)

這時如果需要構建某個目標，在依賴圖中找到目標對應的節點，後續遍歷這個節點，對每個訪問到的節點執行重建操作。如果依賴圖中不存在目標對應的節點，那麼就說明目標沒有“過期”，不需要做任何重建操作。比如用戶要構建`a.o`，那麼就會依次重建`a.c`和`a.o`，`a.c`不是任何規則的目標所以不執行重建操作，`a.o`是某條規則的目標，重建之。如果要構建`main`則會依次構建`a.c`、`a.o`、`main`。**執行完所有重建操作後刪除重建過的節點**，仍舊變成空圖：

![空 DAG](/static/alternative-build-system/alternative-0.png)

修改`main.h`之後，`main.h`、`a.o`、`b.o`、`main`都過期了：

![DAG](/static/alternative-build-system/alternative-2.png)

重建`b.o`。需要刪除`b.o`及其所有直接間接依賴：

![DAG](/static/alternative-build-system/alternative-3.png)

小結一下有兩條規則：

- **得到文件修改通知後把對應節點及所有祖先(這個祖先是針對全圖來說的)加入到依賴圖中。**
- **重建過的節點都從依賴圖中刪去。**

## 實際應用

[Tup](http://gittup.org/tup)是一個採取了這個技術的構建系統。

文件修改通知可以用Inotify、FAM、Gamin等。

## 參考文獻

Tup項目首頁給出的鏈接：[Build System Rules and Algorithms](http://gittup.org/tup/build_system_rules_and_algorithms.pdf)
