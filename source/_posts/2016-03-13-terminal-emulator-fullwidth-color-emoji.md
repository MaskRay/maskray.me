---
layout: post
title: 終端模擬器下使用雙倍寬度多色Emoji字體
permalink: 2016-03-13-terminal-emulator-fullwidth-color-emoji
tags: [terminal,vte,emoji]
---

2017年8月更新。

## 多色Emoji字體

### Cairo支持

較新的FreeType支持多色，但cairo-1.14.6沒有默認開啓支持。[hexchain](https://hexchain.org)指出修改cairo源碼一行代碼即可：

<!-- more -->

```patch
--- a/src/cairo-1.14.6/src/cairo-ft-font.c	2016-03-13 09:36:42.618325503 +0800
+++ b/src/cairo-1.14.6/src/cairo-ft-font.c	2016-03-13 09:38:24.194288159 +0800
@@ -2258,7 +2258,7 @@
      * Moreover, none of our backends and compositors currently support
      * color glyphs.  As such, this is currently disabled.
      */
-    /* load_flags |= FT_LOAD_COLOR; */
+    load_flags |= FT_LOAD_COLOR;
 #endif

     error = FT_Load_Glyph (face,
```

然後編譯安裝cairo。

2017年8月更新，cairo-1.14.10依然需要這個patch……

#### Arch Linux普通用戶

可以安裝`aur/cairo-coloredemoji`。

#### Arch Linux infinality-bundle用戶

huiyiqun指出，infinality-bundle用戶應該：

1. 安裝<https://aur.archlinux.org/packages/cairo-infinality-ultimate-with-colored-emoji/>
2. <del>刪除`/etc/fonts/conf.d/82-no-embedded-bitmaps.conf`</del>。按<https://gist.github.com/huiyiqun/9f20f177655946263a48170ee662cea9>配置fontconfig。

### Fontconfig配置

安裝Noto Color Emoji字體，Arch Linux可以裝`extra/noto-fonts-emoji`。

推薦使用hexchain的配置：<https://gist.github.com/hexchain/47f550472e79d0805060>。把Noto Color Emoji的配置放到`~/.config/fontconfig/conf.d/`或全局的`/etc/fonts/conf.d/`，使用`fc-cache`或`sudo fc-cache`更新。

可以用`fc-match -s monospace`、`fc-match -s sans`、`fc-match -s sans-serif`確定字體選擇順序。我這裏Noto Color Emoji只能排到第二，不明原因。

DejaVu Serif/Sans包含部分黑白emoji glyphs，如果fc-match順序在Noto Color Emoji前面，就看不到後者的相應字形。可以考慮安裝aur/ttf-dejavu-emojiless等移除emoji glyphs的包。

## Emoji字符寬度

經過以上兩步配置，理應可以使用Noto Color Emoji字體了。但對於Monospace字體，emoji字符顯示寬度爲1，會與其後的字符重疊。

### vte中設置emoji字符寬度

2017年8月更新，較新的vte3-ng或依賴的glib終於修好了寬度計算，不需要這麼patch了。

在終端模擬器中，字符是按列顯示的，因此有寬度的概念。vte把emoji字符當作1，因此顯示時會與之後的字符重疊。

以終端模擬器[termite](https://github.com/thestinger/termite)使用的`community/vte3-ng`爲例。修改源碼`src/vte.cc:_vte_unichar_width`，讓該函數對於某些Noto Color Emoji提供的Unicode block返回2。此處我硬編碼了幾個用到的Unicode block，不全：
```patch
--- a/src/vte.cc        2016-03-12 23:44:04.157720973 +0800
+++ b/src/vte.cc        2016-03-13 00:20:45.592623311 +0800
@@ -206,6 +206,10 @@
                 return 0;
         if (G_UNLIKELY (g_unichar_iswide (c)))
                 return 2;
+        if (G_UNLIKELY(0x25a0 <= c && c < 0x27c0 || // Geometric Shapes, Miscellaneous Symbols, Dingbats
+                       0x2b00 <= c && c < 0x2c00 || // Miscellaneous Symbols and Arrows
+                       0x1f300 <= c && c < 0x1f700 || // Miscellaneous Symbols and Pictographs ... Geometric Shapes Extended
+                       0))
+                return 2;
         if (G_LIKELY (utf8_ambiguous_width == 1))
                 return 1;
         if (G_UNLIKELY (g_unichar_iswide_cjk (c)))
```

Arch Linux可以裝`aur/vte3-ng-fullwidth-emoji`。該patch另外修復了canonical mode下退格寬字符，終端模擬器只移動一格光標的問題(其實應該是kernel的bug，不妨頭痛醫腳)。

### 設置`wcwidth`寬度

對於ncurses應用，會用`wcwidth`計算待擦除字符的寬度。僅經過上面的配置，`wcwidth`仍認爲emoji字符寬度爲1，擦除時寬度計算不對，可能導致一些字符殘餘在屏幕上。

`wcwidth`對字符寬度的計算由locale決定，比如對於常用的`en_US.UTF-8`等，glibc提供的`/usr/share/i18n/charmaps/UTF-8.gz`中`WIDTH`、`END WIDTH`區塊給出了字符寬度信息。但其中沒有列出Emoji字符，因此寬度將用缺省值1。

我用<https://gist.github.com/MaskRay/86b71b50d30cfffbca7a>重新生成一個`UTF-8`，gzip壓縮後覆蓋`/usr/share/i18n/charmaps/UTF-8.gz`，然後執行`locale-gen`。修改後，可以用<https://gist.github.com/MaskRay/8042e39dc822a57c217f>確定`wcwidth`計算出來的寬度確實變更了。

[glibc 2.26](https://sourceware.org/ml/libc-alpha/2017-08/msg00010.html)支持Unicode 10.0.0的character encoding, character type info, transliteration tables，很多emoji字符的寬度應該正確了。

### Emoji ligature

很多emoji是由幾個codepoints拼湊出來的，比如Regional Indicator Symbol Letter C(U+1F1E8)和Regional Indicator Symbol Letter n(U+1F1F3)組合得到中國國旗(🇨🇳)，skin tone modifier可以切換膚色。這類emoji寬度在終端裏就很難弄對。

#### Arch Linux用戶

`/etc/pacman.d/hooks/update-charmaps-UTF-8.hook`:

```
[Trigger]
Operation = Upgrade
Type = File
Target = usr/share/i18n/charmaps/UTF-8.gz

[Action]
When = PostTransaction
Exec = /etc/pacman.d/hooks/update-charmaps-UTF-8.py
```

`/etc/pacman.d/hooks/update-charmaps-UTF-8.py`:

從<https://gist.github.com/MaskRay/86b71b50d30cfffbca7a>下載

## 效果

```zsh
echo 🚂🚊🚉🚞🚆🚄🚅🚈🚇🚝🚋🚃🚟
```

![](/static/2016-03-13-terminal-emulator-fullwidth-color-emoji/trains.jpg)

爲了這張笑臉真是一把辛酸淚……

![](/static/2016-03-13-terminal-emulator-fullwidth-color-emoji/smile.jpg)

## 感想

Monospace實現很困難。要畫出好看的Emoji，勢必要用fullwidth，這個屬性如果能由字體提供是再好不過的。對於底層的`glibc、glib`，它們並不能知道字體，但又不得不規定字符寬度，一不小心即與字體的實際寬度產生衝突。翻了翻<http://www.unicode.org/Public/UCD/latest/ucd/>等，好多地方分散地提到了fullwidth，比如Halfwidth and Fullwidth Forms、East Asian Width等，不同實現對這些概念的理解會有偏差……

在neovim裏寫這篇文章時又發現neovim對字符寬度也有問題，`src/nvim/mbyte.c:utf_char2cells`看上去是無辜的，所以誰壞掉了呢？

## 參考

- <https://pixelambacht.nl/2014/multicolor-fonts/>
- Noto Emoji字體：<https://github.com/googlei18n/noto-emoji>
- Emoji One Color字體：<https://github.com/eosrei/emojione-color-font>
- <http://stackoverflow.com/questions/23526353/how-to-get-ncurses-to-output-astral-plane-unicode-characters/>
