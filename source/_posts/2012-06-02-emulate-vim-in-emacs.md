---
layout: post
title: Evil--在Emacs中模擬Vim
author: MaskRay
tags: emacs
---

这篇文章过时了，參見[皈依Emacs](/blog/2015-09-18-conversion-to-emacs)，现在我對操作方式有較大調整。

### Vim模擬

平心而論，Vim的modal editing確實比Emacs強，而Emacs默認的按鍵綁定設計不好，不適合使用，要讓它適合做工作環境往往要改大量按鍵。“一千個讀者心中有一千個哈姆雷特”，Emacs的配置也確實大相徑庭。

<!-- more -->

但和打字配套的附件功能始終不如Emacs，比如repl環境、文件管理、筆記管理、minor mode 等。相當一部分major mode(淺薄地說，和Vim的filetype對應)，都是Emacs的比Vim的對應物好用，比如[AUCTeX](http://www.gnu.org/software/auctex/)，[haskell-mode](http://www.haskell.org/haskellwiki/Haskell_mode_for_Emacs#Haskell-mode)，[tuareg-mode](http://www.emacswiki.org/emacs/TuaregMode)。

### Evil

[Evil](http://emacswiki.org/emacs/Evil)是又一個Vim模擬器，是Vimpulse和vim-mode的接替者。

Pandoc和Gitit的作者John MacFarlane也使用Evil，他也是個能折騰工具的人，他的選擇也能說明Evil確實不錯：[http://john.macfarlane.usesthis.com/]()。

[這篇文章](http://dnquark.com/blog/2012/02/emacs-evil-ecumenicalism/)介紹了一些Evil使用的技巧，另外強烈推薦Michael Markert的[Evil配置](https://github.com/cofi/dotfiles/blob/master/emacs.d/config/cofi-evil.el)。

### 插件

目前似乎有四個插件，個人認爲最有用的是這兩個：

[evil-surround](https://github.com/timcharper/evil-surround)。用來模擬[surround.vim](https://github.com/tpope/vim-surround) 的。這項功能一直是Emacs的軟肋，因爲缺乏normal mode，即使有text object的移植，換成key sequence再加上control meta之類的鍵就變得比surround.vim笨拙很多了。

[evil-leader](https://github.com/cofi/evil-leader)。用來模擬Vim的mapleader。我一般把`mapleader`設爲逗號，添加`ido-find-file save-buffer compile`等綁定。

## 2015

大概從2012年下半年開始，我基本只用Vim，Emacs僅用於看info。一晃三年過去了，今天因爲要用Proof General而又折騰了下Emacs。三年前的配置其實還能用，但還是借這個機會大刀闊斧改掉。 Emacs的生態還是很棒，很多包都轉移到GitHub了(Vim的生態轉到GitHub感覺更早些)。包管理系統逐漸改善，helm、org-mode、auctex和衆多FP語言的major-mode都是Vim對應插件無可比擬的。有Evil在，編輯功能不至於比Vim差太多，繼續啓用Emacs作爲主力編輯器！

- <http://nathantypanski.com/blog/2014-08-03-a-vim-like-emacs-config.html>
- <http://www.emacswiki.org/emacs/Evil>
- <http://juanjoalvarez.net/es/detail/2014/sep/19/vim-emacsevil-chaotic-migration-guide/>
