---
layout: post
title: 皈依Emacs
author: MaskRay
tags: [emacs,vim]
---

這應該算第三次使用Emacs了。第一次是在2009年，NOI 2009醬油後下決心好好學習Linux，Philip Xu前輩我指引了兩天，發現Linux這個未知世界竟然有這麼多好玩的軟件。編輯器之神Vim用了一陣子了，NOIP 2009前若干天開始探索神的編輯器Emacs，使用Emacs Lisp配置，實現與Vim相似的設定效果需要多出好多代碼，因此不久又回到Vim。大概2010下半年又用起Emacs，這次好好折騰了一把。當時Vim的插件生態還圍繞vim.org(現在逐步轉戰GitHub了)，Emacs倒是有個<http://www.emacswiki.org>，嘗試了Mew、ERC、AUCTeX、haskell-mode、[org-mode](http://org-mode.org)等好多東西。很多文件類型的解析、自動縮進支持比Vim對應物好不少，插件質量感覺也稍高。

<!-- more -->

Emacs有時被人詬病爲一個缺乏好的編輯器的操作系統。誠然，收郵件、上IRC、內置終端模擬器、各種網站客戶端，網上能找到很多插件賦予它做各式各樣任務的能力。我2012年第二次棄用Emacs是因爲我認爲作爲“操作系統”的Emacs扮演了很多窗口管理器的角色，我已經配置好了xmonad，這些任務應該交給xmonad。Emacs扮演的窗口管理器提供了不錯的REPL能力，buffer間通信的方便程度不是不同進程間通信可比的。

Emacs的很多major mode會與外部程序通信，提供快捷鍵把代碼傳遞給inferior process執行，再把結果傳輸回來，並用適當形式在Emacs中顯示。如haskell-mode、tuareg-mode、erlang-mode等，這些major mode往往都提供REPL功能，而且詞法高亮、縮進也都優於Vim下的對應物。另外minor mode的設計着實高明，功能模塊可以方便地載入和卸載，比Vim使用autocommand和filetype檢測方便。Emacs能作爲操作系統的競爭力來自於它是個優秀的Lisp交互式編程環境，活着的Lisp Machine。求值光標前的代碼、查詢快捷鍵綁定、查閱函數文檔，有相當多功能通過Vim的插件很難做好。

下圖是j-mode，使用快捷鍵把當前行傳輸給inferior process (jconsole)並獲取執行結果。
![j-mode](/static/2015-09-18-conversion-to-emacs/j-mode.gif)

下圖是Proof General，theorem prover的界面，錄色底色部分爲已經執行過的命令，階段性結果顯示在下方buffer中。Proof General提供了快捷鍵執行下一條命令或者撤銷上一條命令。
![Proof General](/static/2015-09-18-conversion-to-emacs/proofgeneral.gif)

web-mode可以在一個buffer中同時支持HTML、CSS、JavaScript等多種詞法高亮，根據當前major mode提供的菜單也非常方便。
![web-mode](/static/2015-09-18-conversion-to-emacs/web-mode.jpg)

感謝最近的awesome-*系列項目，全方位介紹各種領域的插件，通過<https://github.com/emacs-tw/awesome-emacs>，發現Emacs的插件生態還挺好的。這是再次皈依Emacs的重要原因，另一方面是感覺Emacs固有的設計優於Vim。GTK+的Emacs是個富文本編輯器，每個字符都可以有不同的樣式，而GVim的字符大小是統一的，在UI表現能力上弱於Emacs。[AUCTeX](https://www.gnu.org/s/auctex/)就充分利用了Emacs的這個優勢，解析TeX中粗體、斜體、大字號、小字號等標記，用相應的樣式顯示，一目瞭然。Google Images中搜索AUCTeX有很多圖片例子。另外Emacs也可以內嵌圖片，公式片段作爲圖片渲染，這可是個殺手級特性。再譬如[prettify-symbols-mode](http://www.emacswiki.org/emacs/PrettySymbol)，Vim也有conceal，但是由於字體顯示的缺陷，不能很好發揮。

自package.el誕生以來Emacs的包管理生態也逐漸好轉。第二次放棄Emacs時還有很多插件散落在網上各地，如今逐步被ELPA等倉庫收納。現在包管理的選擇太多，[el-get](https://github.com/dimitri/el-get)、[cask](https://github.com/cask/cask)、[quelpa](https://github.com/quelpa/quelpa)……選擇太多有些時候不是好事。

## Evil

Vim的modal editing非常方便，我認爲Vim勝過Emacs的最主要的地方在於normal mode下的一些操作，特別是光標移動和operator-pending mode。Emacs默認的光標移動快捷鍵主要有`C-f`、`C-b`、`C-p`、`C-n`、`M-f`、`M-b`等，使用頻率很高，但卻需要兩個鍵，令人惱火。優於缺少模式，Emacs默認提供的很多快捷鍵使用了很長的key sequence，比如`C-x r j`映射爲`jump-to-register`，後面跟字符`a`就類似於Vim的`` `a``，但是太不方便了。

所幸插件[evil-mode](http://www.emacswiki.org/Evil)可以在Emacs裏模擬Vim，使Emacs具備modal editing的功能。爲了更好的一致性，我不使用insert mode中的Vim特有快捷鍵，只用Emacs默認的：

```emacs
(setcdr evil-insert-state-map nil)
(define-key evil-insert-state-map (read-kbd-macro evil-toggle-key) 'evil-normal-state)
(define-key evil-insert-state-map [escape] 'evil-normal-state)
```

Evil另外附帶了一些Vim中以插件形式提供的功能。下圖是Evil整合的ace-jump(Vim的easymotion)，跳轉到指定字符或以指定字符開頭的單詞、行等：
![evil-ace-jump](/static/2015-09-18-conversion-to-emacs/evil-ace-jump.gif)

## Helm

和Vim的[Unite](https://github.com/Shougo/unite.vim)類似但強大得多的必備插件，參見<http://tuhdo.github.io/helm-intro.html>。

helm-ag，在當前目錄下用[ag](https://github.com/ggreer/the_silver_searcher)搜索，顯示匹配項。
![helm-ag](/static/2015-09-18-conversion-to-emacs/helm-ag.gif)

[helm-imenu](https://github.com/emacs-helm/helm/blob/master/helm-imenu.el)，[Imenu](http://emacswiki.org/emacs/ImenuMode)的增量搜索版：
![helm-imenu](/static/2015-09-18-conversion-to-emacs/helm-imenu.gif)

[helm-swoop](https://github.com/ShingoFukuyama/helm-swoop)，增量搜索匹配行，類似於`helm-occur`，但會即時顯示匹配的部分。
![helm-swoop](/static/2015-09-18-conversion-to-emacs/helm-swoop.gif)

## Global的前端helm-gtags

[helm-gtags](https://github.com/syohex/emacs-helm-gtags)，Global的前端。`helm-gtags-dwim`、`helm-gtags-find-rtag`很有用，值得爲之綁定簡單的快捷鍵。
![helm-gtags](/static/2015-09-18-conversion-to-emacs/helm-gtags.gif)

## Org-mode

一款日程安排、項目管理、literate programming、筆記軟件，神器。參考<http://doc.norang.ca/org-mode.html>。

## 鍵綁定與衝突

fcitx默認以`C-SPC`爲輸入法切換鍵，與Emacs默認的`set-mark-command`衝突，我把切換鍵改爲`C-S-SPC`。

fcitx默認綁定了`C-M-h`用於拼寫檢查，與Emacs默認的`mark-defun`衝突，我刪除了fcitx的綁定：Configure-Input Method，雙擊Keyboard，點擊Toggle to word hint，按ESC。

## 其他

- `M-x winner-mode`，內置，global minor mode。可以用`C-c left`和`C-c right`在撤銷或重做對窗格佈局的變更。
- `M-x ansi-term`，內置，可顯示顏色的終端模擬器。
- [dired](http://emacswiki.org/emacs/DiredMode)，內置的文件管理器。
- [expand-region](https://github.com/magnars/expand-region.el)，基於語義單位增量式選擇區域。視頻<http://emacsrocks.com/e09.html>。
- [neotree](https://github.com/jaypei/emacs-neotree)，樹狀文件管理器，類似Vim的[NERD Tree](https://github.com/scrooloose/nerdtree)。
- flycheck，即時語法檢查，類似Vim的syntastic。
- [projectile](https://github.com/bbatsov/projectile)，提供一些以項目爲單位的實用功能。
- [smartparens](https://github.com/Fuco1/smartparens)，處理廣義括號的minor mode。
- [which-key-mode](https://github.com/justbur/emacs-which-key)，在彈出的buffer中顯示當前按下的前綴鍵的所有可能綁定。
  ![which-key-mode](/static/2015-09-18-conversion-to-emacs/which-key-mode.jpg)

目前採用的GIF錄製方法：

`xwininfo`獲取Emacs窗口位置大小信息，用`byzanz-record -x 1 -y 33 -w 960 -h 640 -d 10 a.gif`錄製GIF動畫，再用`gifsicle --scale 0.75 --delete '#33-' a.gif > b.gif`裁剪開頭部分。

## 我的配置

<https://github.com/MaskRay/Config/tree/master/home/.emacs.d>

## 進一步瞭解

Vim到Emacs的遷移：

- <http://juanjoalvarez.net/es/detail/2014/sep/19/vim-emacsevil-chaotic-migration-guide/>
- <http://ceyes.github.io/2015-01/from-Vim-to-Emacs/>

聚合、Wiki：

- <http://emacsrocks.com/>
- <http://www.emacswiki.org/>

一些配置：

- <http://pages.sachachua.com/.emacs.d/Sacha.html>
- <http://www.wisdomandwonder.com/wordpress/wp-content/uploads/2014/03/C3F.html>
- <https://github.com/codahale/emacs.d>
- <https://github.com/bbatsov/prelude>
