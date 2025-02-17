---
layout: post
title: 在Makefile中自動生成依賴
author: MaskRay
tags: [makefile]
---

[論壇](http://forum.ubuntu.org.cn/viewtopic.php?f=21&t=341048&p=2440061#p2440061)裏有人寫了一個用於自動生成C/C++依賴的腳本。但是他的腳本處理不同目錄的源文件時會有些問題，`.o` 會生成在當前目錄，而不是和 `.cc` 同一個目錄。

`gcc` 其實有一些生成用於 `Makefile` 規則的選項，`-M` 等，說明如下：

  - `-MM`，忽略依賴中的系統頭文件
  - `-MF`，指定生成的規則文件的路徑
  - `-MT`，指定規則的目標
  - `-MP`，對每一個依賴創建一個 `force target`，典型輸出：

        test.o: test.c test.h
        test.h:

    沒有這個選項的話是不會有 `test.h:` 的

<!-- more -->

我寫的 `Makefile` 如下，注意要把規則中的8格替換成tab。

    PROG := main
    SRCS := $(PROG).c

    $(PROG): $(SRCS:.c=.o)
            $(LINK.c) $^ -o $@

    sinclude $(SRCS:.c=.d)

    %.o: %.c
            gcc -MM -MP -MT $@ -MF $(@:.o=.d) $<
            $(COMPILE.c) $< -o $@

比較難說明每個規則的作用，所以還是用實例好了。

  - 一開始，`*.o` `*.d` 都不存在，只有一個 `main.c`
  - 執行 `make`，`make` 沒法讀取 `sinclude` 那行表示的文件 `main.d`，但由於是
    `sinclude`，忽略這個錯誤
  - 嘗試重建 `main`，發現需要的 `main.o` 不存在
  - 因此嘗試先重建 `main.c`。執行規則 `%.o: %.c`，生成 `main.o` 的同時也
    生成了 `main.d`。假設 `main.c` 包含一個頭文件 `a.h`，那麼 `main.d` 的內容將會是：
        main.o: main.c foo/a.h
        foo/a.h:

之後如果修改 `main.c` 或者 `foo/a.h`，因爲 `main.d` 中的規則都會重建 `main.o`。

再看添加新依賴 `bar/b.h` 的情況，沒錯，這時我們的 `main.d` 是過期的，不能反映 `main.c` 的真實依賴情況。但注意到 `main.d` 的過期必然蘊含：`main.c` 被修改了。根據規則 `%.o: %.c`，再次重建 `main.d` 和 `main.o`。

假設我們刪除 `main.c` 中的 `#include "foo/a.h"`，同樣的，`main.d` 過期了，不能反映 `main.c` 的真實依賴情況。但注意到 `main.d` 的過期必然蘊含：`main.c` 被修改了。根據規則 `%.o: %.c`，再次重建 `main.d` 和 `main.o`。

如果我們不改動 `main.c` 而是直接刪除 `foo/a.h`，那麼根據 `main.o: main.c foo/a.h`，無法重建 `main.o`，`make` 報錯。注意：`foor/a.h` 是 `force target`，它不存在的話不會報錯，只是讓依賴它的目標強制重建。這也是我們之所以使用 `-MP` 的原因，否則如果不存在目標爲 `foo/a.h` 的規則，`make` 會報錯：無法重建 `foo/a.h`。

還有種生成依賴的方式是讓 `main.o` 依賴 `main.d`，`main.d` 依賴 `main.c`，但是這種方法可能會導致 `make` 的重啓，當 `make` 很複雜的時候性能不如前文的方法。
