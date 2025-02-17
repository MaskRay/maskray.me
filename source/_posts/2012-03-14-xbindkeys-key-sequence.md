---
layout: post
title: 給xbindkeys添加key sequence支持
author: MaskRay
tags: [xbindkeys, scheme]
---

`xbindkeys`不支持`emacs`風格的`key sequence`，不過由於它可以調用`guile`來支持`scheme`的配置文件。`xbindkeys`的`tarball`裏有一個配置文件，支持`key sequence`的特殊形式：兩個鍵的序列。

最近正好學了些`scheme`，就好好折騰了一把，寫了一份配置文件，可以支持任意長的`key sequence`(當然太長的一般用不到)。

<!-- more -->

```lisp
(define (delete-duplicates l)
  (cond ((null? l) '())
        ((memq (car l) (cdr l)) (delete-duplicates (cdr l)))
        (else (cons (car l) (delete-duplicates (cdr l))))
        )
  )

(define global-map '())

(define (register keys action)
  (define (loop getmap setmap keys)
    (if (null? keys)
        (setmap action)
        (let ((k (car keys)))
          (if (eq? #f (assoc k (getmap)))
              (setmap (assoc-set! (getmap) k '())))
          (loop (lambda () (cdr (assoc k (getmap))))
                (lambda (m) (setmap (assoc-set! (getmap) k m)))
                (cdr keys))
          )
        )
    )
  (loop (lambda () global-map) (lambda (m) (set! global-map m)) keys)
  )

(define (grab keymap)
  (define (proc k)
    (define action (cdr (assoc k keymap)))
    (ungrab-all-keys)
    (remove-all-keys)
    (if (string? action)
        (begin
          (run-command action)
          (reset-first-binding)
          )
        (grab action)
        )
    (grab-all-keys)
    )
  (map (lambda (k)
         (xbindkey-function k (lambda () (proc k)))
         ) (delete-duplicates (map car keymap)))
  (if (not (eq? global-map keymap))
      (xbindkey-function '(control g) (lambda () (reset-first-binding)))
      )
  )

(define (first-binding)
  (grab global-map)
  )

(define (reset-first-binding)
  (ungrab-all-keys)
  (remove-all-keys)
  (first-binding)
  (grab-all-keys))

(define (simple s)
  (string-concatenate `("wmctrl -xa " ,s "||" ,s))
  )

(register '((control semicolon) x) (simple "xterm"))
(register '((control semicolon) u) "wmctrl tmux || -T tmux -e tmux attach -t default")
(register '((control semicolon) e) "wmctrl -xa Emacs || emacsclient -c -n")
(register '((control semicolon) v) "wmctrl -xa Vim || gvim")
(register '((control semicolon) f) (simple "firefox"))
(register '((control semicolon) i) (simple "evince"))
(register '((control semicolon) (control semicolon)) "xdotool send ctrl+semicolon")

(first-binding)
```

用法很簡單，模仿代碼中的`(register ...)`，在這串鍵綁定裏添刪你自己的。`xbindkeys`還支持`shift` `mod3` `release`等修飾符。`release`指的是鍵釋放時而不是`press`時執行動作。`xbindkeys -d`還有其他一些關於修飾符的例子。

我的鍵綁定中`wmctrl -xa`是一個用外部命令搭建的簡易`jump-or-exec`，`wmctrl`會檢查窗口`title`和`class`，如果不存在則執行，否則給相應窗口焦點。`(simple ...)`針對的是`title`和`class`相同的情況。

兩次`control semicolon`被我映射爲產生一個`control semicolon`，`key sequence`按到一半時按`control g`是取消。
