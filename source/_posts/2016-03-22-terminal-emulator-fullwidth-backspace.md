---
layout: post
title: 終端模擬器下寬字符退格
author: MaskRay
tags: [terminal,vte,unicode]
---

折騰完<https://maskray.me/blog/2016-03-13-terminal-emulator-fullwidth-color-emoji>後發現canonical mode下emoji字符退格只後退了一列，後發現所有寬字符都有問題，因此做了一番調研。

## Canonical/noncanonical mode

早期Unix有cooked/cbreak/raw mode三種模式，raw mode和cbreak模式區別在於signal和輸入輸出處理，輸入都是以字符爲單位，即`read(STDIN_FILENO, buf, 1)`在鍵入一個字符後即返回。Cooked mode與它們差別較大，最重要的區別是終端輸入以行爲單位進行，並自帶一個基礎行編輯器，可以使用退格和WERASE(默認爲`^W`)刪除光標前的單詞。

<!-- more -->

termios引入後對輸入輸出行爲(input/output/local modes)有了更精細的控制，通過一些選項可以定製出原始的cooked/cbreak/raw mode三種模式。Local modes中的`ICANON`最爲重要，區分canonical/noncanonical mode，canonical mode與早期cooked mode類似，帶行編輯器，一些字符如CR EOF EOL ERASE KILL NL WERASE等有特殊含義，下面介紹幾個比較重要的。更多介紹參見The Linux Programming Interface 62.4 Terminal Special Characters。

### EOF

通常爲`^D`，可以用`ctrl d`輸入，作用是使得`read()`立即返回該行所有字符，若位於行首則返回0。很多地方把它視爲到達文件結束位置的信號，並不再繼續讀入。但實際上終端輸入並沒有被關閉，仍可以繼續讀取字符。

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
  char buf[9];
  for(;;) {
    printf("%d\n", read(0, buf, 9));
  }
}
```

編譯運行上面C程序，試試輸入若干字符後按`^D`的輸出。

### ERASE

`stty -a`中`erase = `顯示當前ERASE字符設置，通常爲`^?`或`^H`。終端模擬器vte是`^?`，退格鍵發送`^?`；xterm是`^H`，退格鍵發送`^H`。倘若終端ERASE字符與之不同則可能導致退格不刪除字符反而輸入了一個`^?`或`^H`。

### INTR

通常爲`^C`，若local modes中`ISIG`開啓，則前臺進程組會收到SIGINT。

### QUIT

通常爲`^\`，若local modes中`ISIG`開啓，則前臺進程組會收到SIGQUIT。

### SUSP

通常爲`^Z`，若local modes中`ISIG`開啓，則前臺進程組會收到SIGTSTP，默認會停止成爲後臺任務，shell回到前臺。

對於readline等使用noncanonical mode的應用程序，它們會檢測`TERM`環境變量獲取terminfo信息，從中找出不同功能對應的輸出字符序列。

## 退格

`\b`字符的解析方式是光標左移一格。

在canonical mode下當前行僅有一個兩個寬字符時，按下退格，光標左移一格並擦除了該字符，但繼續按退格也無法回到行首，產生顯示問題。

原因是內核tty驅動似乎沒有考慮字符寬度信息，只給pseudoterminal master發送一個`\b`，終端模擬器收到`\b`後將光標左移了一格。再次按退格時，內核tty驅動判斷該行已空，因此不再發送`\b`，光標也就無法退回到行首。

介紹一個測試方式：在pseudoterminal的slave端運行canonical mode的`cat`程序，輸入退格，可以在master端看到內核發來`"\b \b"`三個字符，即後退一格，空格擦除，再後退一格。可以用下面的方法查看：

`termite -e cat`創建一個termite終端運行`cat`，然後找出termite在pseudoterminal pair的master端的fd：
```zsh
% lsof -p $(pgrep -n termite) -Fn | sed -n '/^n\/dev\/ptmx/{g;p};s/^f//;h'
13
```

之後用`strace -e read -p $(pgrep -n termite) |& grep 13`，在termite窗口鍵入退格，觀察master端fd讀到的數據。

這很可能是內核的問題，簡易的修復方式是頭痛醫腳，修改使用pseudoterminal的程序(如終端模擬器、tmux)的代碼。對於canonical mode並開啓`IUTF8`時，從pseudoterminal master處讀到`\b`時，判斷左側字符是否爲寬字符，是則左移2格(目前尚無更寬的字符)。我做了兩個patch：

- <https://aur.archlinux.org/packages/vte3-ng-fullwidth-emoji>
- <https://aur.archlinux.org/packages/tmux-fullwidth-backspace>

於是這是我第一次和第二次創建PKGBUILD……

## 參考

- <https://blog.nelhage.com/2009/12/a-brief-introduction-to-termios-termios3-and-stty/>
- `info '(libc) Terminal Modes'`
