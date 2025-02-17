---
layout: post
title: 用Makefile搭建博客-2
author: MaskRay
tags: [makefile, shell, website]
---

## 緣起

前一篇[用Makefile搭建博客](./2011-07-12-blogging-with-makefile.html)說到我用`inotifywait`監控目錄下文件的寫操作來更新目錄的修改時間，以後只要把`make inotify`放到啓動腳本中就行了。但當時就發現了一個問題，一直拖到今天才解決掉。

<!-- more -->

### GoboLinux Scripts包中的ColorMake

先從/ColorMake/說起。/GNU Make/的顏色是很單調的，我一般用GoboLinux的包Scripts裏的`ColorMake`來給`make`上色。而這個`ColorMake`實際上是寫了一個`mtail`的配置文件`ColorMake.mtailrc`，把`make`的輸出管道給`mtail`來上色。[可以參看](http://forum.ubuntu.org.cn/viewtopic.php?f56&t285953)。我之前則是用的<http://bre.klaki.net/programs/colormake/>裏的`colormake.pl`腳本來上色的。

我把`ColorMake.mtailrc`保存爲 ~/bin/ColorMake.mtailrc ，另外寫了個 wrapper，保存到 ~/bin/mk ，內容如下：

    #!/bin/sh
    /usr/bin/make "$@" 2>&1 | mtail -q --config ~/bin/ColorMake.mtailrc

意思就是把`mk`的參數全部傳遞給`make`，`make`的 stdout stderr 全部管道給`mtail`來上色。

另外， ~/bin 在我的環境變量`PATH`中。

## 產生問題的命令

先來看我的 [[/Makefile][Makefile]]，只要看 inotify 僞目標，其他的可以忽略掉。之前 inotify 的規則是 inotifywait -e modify -m -r . --format %w | xargs -I % sh -c "touch \`dirname %\`" &

make inotify 運行得非常正常，`inotify`和`xargs`在後臺執行；但如果執行`mk inotify`，問題就來了，終端會被佔着，無法再執行其他命令了。

## 分析

### `make inotify`

先來看執行`make inotify`會發生什麼，不妨假設交互用的shell是Zsh，這裏用Bash效果也是一樣的。

-`zsh`進程產生一個`make`進程
-`make`進程執行重建 inotify 的規則，即產生一個進程執行``inotifywait -e modify -m -r . --format %w | xargs -I % sh -c "touch \`dirname %\`" &``

不妨用 /Lisp/ 來表示進程樹，那麼這些進程的關係如下： (zsh (make (inotifywait) (xargs))) 。
一對*圓括號*代表*進程*，圓括號第一個元素是*進程名*，其餘元素代表*子進程*。

接着，

-`make`退出，因爲規則執行完了
-`zsh`檢測到它的子進程`make`退出，又可以執行其他命令了

### `mk inotify`

-`zsh`進程產生`mk`，其實是用`/bin/sh`解釋`mk`，這裏就簡寫成`mk`
-`mk`產生`make`和`mtail`，其中管道的一端是`make`的fd1、fd2，另一端是`mtail`的fd0
-`make`產生`inotifywait`和`xargs`

第三步中，`make`的文件描述符被`inotifywait`和`xargs`繼承，
由於`inotifywait`和`xargs`用另一根管道而把 fd1 關閉了，
所以現在原管道的兩端分別是：

-`make`的fd1、fd2；`inotifywait`的fd2；`xargs`的fd1、fd2
-`mtail`的fd0

現在的進程樹是： (zsh (mk (make (inotifywait) (xargs)) (mtail)))

接着，

- `make`退出，因爲規則執行完了
- 因爲管道的寫端描述符沒有全部關閉，`mtail`不會讀到EOF退出，而是等待管道讀端的數據
- `mk`也不會退出，因爲它的某個子進程`mtail`沒有退出
- `zsh`未檢測到`mk`的退出，所以終端被佔用了，沒法執行其他命令

## 解決方案

由前面的分析可以看出，只要讓`mtail`退出，那麼`mk`會跟着退出，終端就不會被佔用了。
而要讓`mtail`退出，就要讓它讀到 eof 退出，我們只要讓`make`產生進程時不要把管道的寫端描述符
傳遞給`inotifywait`和`xargs`，但是這個據我所知是做不到的。
但我們可以讓`inotifywait`和`xargs`立刻把相應寫端描述符關閉，
這個很簡單，用

```
inotifywait -e modify -m -r . --format %w 2>&- | xargs -I % sh -c "touch \`dirname %\`" >&- 2>&- &
```

代替原來的命令就行了。
