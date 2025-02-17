---
layout: post
title: Emacs helm-kythe与Haskell交叉引用
author: MaskRay
tags: [emacs,haskell,kythe]
---

一直以来Haskell沒有好用的工具支持goto-references，`hasktags`、`fast-tags`等能提供ctags風格的goto-definition，但不支持goto-references。最近Google開源的一個工具<https://github.com/google/haskell-indexer>(主要作者爲Robin Palotai)提供了基於[Kythe](https://github.com/google/kythe)的交叉引用實現。

<!-- more -->

## [haskell-indexer](https://github.com/google/haskell-indexer)

### 依賴

- [stack](https://www.haskellstack.org)，Arch Linux的話可以使用[felixonmars](https://felixc.at)打包的`community/stack`(感謝肥貓辛勤勞作，喵！)。
- [Google Kythe](https://kythe.io)。把<https://github.com/google/kythe/releases>下載的壓縮包裝到`/opt/kythe`之類的地方。如果不想成爲Bazel工程師的話，不要從源碼編譯，編譯需要一大堆編譯時依賴。

### 索引Hackage

```zsh
git clone https://github.com/google/haskell-indexer
cd haskell-indexer

# 構建索引。如果需要編譯不同Stackage lts版本，請修改`stack.yaml`。
./build-stack.sh /tmp/logs lens mtl
./build-stack.sh /tmp/logs mtlparse cpu

# http_server
./serve.sh /tmp/logs 127.0.0.1:8080
```

打開<http://127.0.0.1:8080>能看到Kythe的簡陋網頁前端。

注意事項：

- Kythe v0.0.26裏的`/opt/kythe/tools/http_server --listen localhost`不會監聽IPv6 ::1。
- haskell-indexer使用GHC API，需要ghc編譯的命令行。當前找到所需源碼及編譯選項、依賴的方式是讓`build-stack.sh`修改`PATH`環境變量，讓`stack build --system-ghc`時使用自己指定的ghc wrapper。但有些時候`stack build`會複製`~/.stack/precompiled/`下的包而不是重新構建。需要一個更加可靠的獲取ghc命令行的方式。目前如果發現某個包`mtl-2.2.1: using precompiled package`的話，可以刪除`~/.stack/precompiled/x86_64-linux/ghc-8.0.2/1.24.2.0/mtl-2.2.1/`目錄後再執行`build-stack.sh`。

## [helm-kythe.el](https://github.com/MaskRay/emacs-helm-kythe)

我寫了一個Emacs Helm擴展，調用`tools/http_server`的HTTP API實現交叉引用。

### 安裝

建議用某個Emacs包管理器加載<https://github.com/MaskRay/emacs-helm-kythe>到`load-path`。`helm-kythe`依賴`dash`和`helm`。

如果只有一個vanilla Emacs的話，可以執行下面代碼，添加`melpa-stable`源並安裝這兩個包。

```lisp
(add-to-list 'package-archives '("melpa-stable" . "https://stable.melpa.org/packages/"))
(package-refresh-contents)
(package-list-packages)  ;; 安裝dash和helm

(load "/path/to/emacs-helm-kythe/helm-kythe.el")
```

### 使用

```lisp
(require 'helm-kythe)
(add-hook 'haskell-mode-hook 'helm-kythe-mode)
```

假設`./build-stack.sh /tmp/logs mtl`時安裝了`mtl-2.2.1`。

- `cabal get mtl-2.2.1`
- `emacs mtl-2.2.1/Control/Monad/Cont/Class.hs`

可以在mode line看到`Kythe`字樣，如果顯示爲~~Kythe~~的話說明`helm-kythe-apply-decorations`沒有執行成功，無法訪問`http://127.0.0.1:8080`或者沒有索引。

- 設置了`eldoc-documentation-function`，把point移動到標識符上可以在minibuffer看到其定義的`snippet`
- `helm-kythe-find-definitions`，默認綁定到`C-c k d`，跳轉到定義
- `helm-kythe-find-references`，默認綁定到`C-c k r`，跳轉到引用
- `helm-kythe-imenu`，默認綁定到`C-c k i`，顯示當前文件頂層定義
- `helm-kythe-resume`，默認綁定到`C-c k l`，打開最近訪問的一個helm-kythe buffer

假設當前文件爲`/tmp/kan-extensions-5.0.2/src/Data/Functor/Contravariant/Coyoneda.hs`，如果要跳轉到的文件在當前Cabal包外，比如要跳到`contravariant-1.4/`目錄，會嘗試找`kan-extensions-5.0.2/`的兄弟目錄，不存在的話再考慮`helm-kythe-filesystem-path-to-corpus-root`。如果你把Cabal `.tar.gz`解壓到其他地方了，請設置這個變量。

## 效果

![效果](/static/2017-07-01-emacs-helm-kythe-and-haskell-xrefs/helm-kythe-haskell.gif)

## C/C++

`helm-kythe`對於C/C++也適用。Emacs做如下配置：

```lisp
(add-hook 'c-mode-hook 'helm-kythe-mode)
(add-hook 'c++-mode-hook 'helm-kythe-mode)

;; 把文件系統路徑與Kythe path雙向轉換需要用的search paths
;; 对于 kythe:?path=proj/a.c 将会在 /tmp/c /tmp/d 下找 /tmp/c/proj/a.c 或 /tmp/d/proj/a.c，选择第一个存在的文件
;; /tmp/d/proj2/b.c 则会转换为 kythe:?path=proj2/b.c
;; 建立起双向映射
(setq helm-kythe-filesystem-path-to-corpus-root '(("/tmp/d" "corpus" "root") ("/tmp/c" "" "")))
```

目前Kythe 0.0.26的命令行工具非常难用，下面是索引`/tmp/c/proj/a.c`的例子。

```zsh
#!/bin/zsh
export KYTHE_OUTPUT_DIRECTORY=kythe/compilations
export ENTRIES_DIR=kythe/entries
export SERVING_DIR=kythe/serving

cd /tmp/c
mkdir -p $KYTHE_OUTPUT_DIRECTORY $ENTRIES_DIR $SERVING_DIR

# extractor
/opt/kythe/extractors/cxx_extractor gcc proj/a.c -o proj/a
/opt/kythe/extractors/cxx_extractor gcc proj/b.c -o proj/b

# indexer
for i in $KYTHE_OUTPUT_DIRECTORY/*.kindex; do
  if [[ $i =~ ([[:xdigit:]]+)\.kindex$ ]]; then
    /opt/kythe/indexers/cxx_indexer --ignore_unimplemented $i > $ENTRIES_DIR/$match[1].entries
  fi
done

# dedup_stream + write_tables
for i in $ENTRIES_DIR/*.entries; do
  #export GS_DIR=kythe/gs
  #/opt/kythe/tools/dedup_stream < $i | /opt/kythe/tools/write_entries --graphstore leveldb:$GS_DIR
  /opt/kythe/tools/dedup_stream < $i > $i.dedup
  /opt/kythe/tools/write_tables --entries $i.dedup --out $SERVING_DIR
done

# http_server
/opt/kythe/tools/http_server --serving_table $SERVING_DIR --listen :8080 --public_resources /opt/kythe/web/ui
```

不運行`tools/http_server`，用命令行客戶端`tools/kythe`檢查生成得到的`kythe/serving`目錄可用：

```
% /opt/kythe/tools/kythe -api serving ls 'kythe:?'
proj/
% /opt/kythe/tools/kythe -api serving ls 'kythe:?path=proj'
a.c
% /opt/kythe/tools/kythe -api serving node 'kythe:?path=proj/a.c'
kythe:?path=proj/a.c
  /kythe/node/kind      file
    /kythe/text
```

可以參考我的Emacs配置<https://github.com/MaskRay/Config/blob/master/home/.emacs.d>。

```lisp
;; spacemacs|define-reference-handlers 是仿照 spacemacs|define-jump-handlers 寫的
(spacemacs|define-reference-handlers haskell-mode)

(add-to-list 'spacemacs-jump-handlers-haskell-mode 'helm-kythe-find-definitions)
(add-to-list 'spacemacs-reference-handlers-haskell-mode 'helm-kythe-find-references)

(define-key evil-motion-state-map (kbd "C-,") 'spacemacs/jump-to-reference)
(define-key evil-motion-state-map (kbd "C-j") 'spacemacs/jump-to-definition)
(define-key evil-motion-state-map (kbd "C-t") 'my-xref-jump-backward)

(defun my-xref-jump-backward ()
  (interactive)
  (if (eq major-mode 'haskell-mode)
      (helm-kythe-jump-backward)
    (helm-gtags-pop-stack)))
```

一晃到七月了，不能再墮落了，開啓CTF模式。
