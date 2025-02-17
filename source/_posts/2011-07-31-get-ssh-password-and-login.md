---
layout: post
title: 自動獲取SSH密碼並登錄
author: MaskRay
tags: [ssh, expect]
---

## 給別人打廣告

_tusooa_ 的腳本，主要用 Bash 和 Perl 寫的，用了類似 GoboLinux 的組織方式，大家可以去看看。圍觀地址：[[https://github.com/tusooa/tusooa]] 。

<!-- more -->

## 正經事

一直以來都是厚臉皮蹭着朋友的 vps ，在家裏連得上，不過其他有些地方沒法用，所以還是要備些免費 ssh 帳號，否則萬一不能用了就……
從 _tusooa_ 的腳本中瞭解到一個提供免費 ssh 帳號的服務器，服務器會定期更新密碼。從這個網頁可以獲取密碼：[[http://vastars.info/free-ssh-source]]。

正如標題所說，我們要做的就是寫一個腳本，自動獲取密碼並登錄。

## Expect 腳本

要自動登錄 ssh ，由於沒法使用 AuthorizedKeysFile ，不難想到用 Expect 來自動
輸入密碼。服務器、帳號、密碼這幾項，可以用 *w3m -dump* 之後用 shell utilities 比如 *sed* 處理。不過既然用了 Expect ，大可以把這些都寫到 Expect 腳本里去。

    #!/bin/sh                                             (ref:shebang)
    # -*- tcl -*- \
    exec tclsh "$0" "$@"
    package require Expect

    spawn w3m -dump http://vastars.info/free-ssh-source   (ref:w3m)
    expect -re "\r\n服務器：(\[0-9\\.]+)" {
        set server $expect_out(1,string)
        exp_continue
    } -re "\r\n帳號：(\[\\w\\.]+)" {
        set user $expect_out(1,string)
        exp_continue
    } -re "\r\n密碼：(\[\\w\\.]+)" {
        set password $expect_out(1,string)
    }

    send_user "服務器: $server\n帳號: $user\n密碼: $password\n"

    spawn ssh -nND 7777 $user@$server                     (ref:ssh)
    expect "(yes/no)" {
        send "yes\r"
        exp_continue
    } "assword" {
        send "$password\r"
    }

    if {[fork]} exit                                      (ref:fork)
    disconnect
    set timeout -1
    expect

## 下面我們來分析這個程序

### [[(shebang)]] 這一段

這是 Expect 腳本常用的寫法。因爲通常無法預料 *tclsh*
會裝在哪個目錄，而我們幾乎可以斷定 *sh* 會在 */bin/sh* ，所以可以設法讓 *sh* 找到 *tclsh* 。

第二行做了兩件事情，首先是讓 Emacs 和 Vim 都能把這個文件
認作 Tcl 腳本（ Expect 腳本是一種 Tcl 腳本）；
對於行尾的反斜槓， *tclsh* 會把下一行 exec tclsh 當作註釋而 *sh*
則不會。

第四行是讓 *tclsh* 加載 Expect 模塊。

注意，這個腳本沒有用到命令行選項，所以把 shebang 改爲 #!/usr/bin/env tclsh
更簡單。

### [[(w3m)]] 這一段
這一段用 expect 分析 *w3m -dump* 抓到的網頁，求出服務器、帳號等信息。
expect 的三個模式都差不多，這裏就看“服務器”那個模式。

首先用 -re 標誌表示模式是個正則表達式，正則表達式是回車換行後的“服務器：”，後面跟任意數字或點。這裏之所以用 \r\n 是門學問，你或許看到了，很多 Expect
腳本都是用的 \r\n ，原因是 Expect 操作了一個僞終端。程序實際上只輸出了 \n ，但通常情況下，終端的 opost 和 onlcr 標誌都是開啓的，輸出到終端時，會把 \n
轉換成 \r\n 。而當程序的輸出重定向到文件時，就不會被轉換，這就是爲什麼重定向到文件
的話不會出現 \r 。 expect 在分析文本前， *w3m* 的輸出已經被終端給轉換了，所以我們要用 \r\n 。

$expect_out(1,string) 代表第一個捕獲括號得到的字符串。我們可以期待
(\[0-9\\.]+) 能捕獲 /IP地址/ ，結果存放在變量 server 中。這裏其實還涉及到
Tcl 的正則表達式流派問題，很難講清楚，需要注意的是 Tcl 的雙引號中 []
有特殊含義，一般需要把第一個 [ 用反斜槓轉義。

### [[(ssh)]] 這一段

幾個標誌的說明：

- -N ，我們 *ssh* 後不需要執行命令
- -n ， *ssh* 既然不需要執行命令，那也不需要 stdin
- -D ，在本地 7777 端口開設 SOCKS 5 代理服務器

`exp_continue`命令類似於 C 的`continue`，它會重新執行當前`expect`。

### [[(fork)]] 這一段

我們的 *tclsh* 進程實際上佔着終端，我們要設法把它轉到後臺去。

一般 fork 是不會失敗的，如果 fork 返回非0代表父進程，否則代表子進程。我們讓 *tclsh* 父進程退出，注意到 *tclsh* 父進程的父進程是 shell ，它注意到 *tclsh* 父進程退出後就會顯示提示符，讓我們繼續輸入命令。

另一方面， *tclsh* 子進程用 disconnect 命令脫離控制終端（就是 shell 的控制終端）。之後它把 timeout 設置爲 -1 即不會超時，用 expect 命令等待 *ssh* 進程退出。這裏也有個注意點：如果 *tclsh* 先於 spawn 出來的進程退出的話，它會殺死那些 spawn
出來的進程。所以我們不能讓 *tclsh* 退出，要讓它等待 spawn 出來的 *ssh* 進程先退出。
