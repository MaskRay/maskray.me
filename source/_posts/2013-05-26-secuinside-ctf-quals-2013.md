---
layout: post
title: SECUINSIDE CTF Quals 2013雜記
author: MaskRay
tags: [security, ctf]
---

## Secuinside CTF Quals 2013

Secuinside是韓國人辦的信息安全比賽，在ctftime.org上排名很靠前。這次應該算是我第二次參加類似的比賽，不過這類做題形式的還是頭一次。

<!-- more -->

我對安全這塊幾乎什麼都不瞭解，本來得知這個比賽的訊息後打算在兩週內好好突擊學習一下的。無奈俗務纏身，困於各個deadline、物理實驗、數電實驗等。不過這學期活動倒是不少，Google GDG兩次活動（一次是Google I/O直播之夜，都是簽到就有T恤衫，相當不錯），Lisp中國地區愛好者技術沙龍、編程之美、Yahoo Hack Day等。直到比賽前兩天才有時間開始學習Web安全方面的知識，[https://pentesterlab.com/web_for_pentester.html]()提供了非常不錯的學習資源，我開始接觸SQL injection、directory traversal這類東西。下載到的iso是可引導的，可以用qemu啓動。圖方便我用了tap方式讓虛擬機接入網絡。另外Linux下有個工具lxc，增強版的chroot，可以構建單獨的網絡環境，對於這個應用來說應該會更方便，以後再瞭解了。

週五21:00多比賽開始，binary類題目遠多餘web類題目，web類題目我也幾乎處於矇昧狀態，在強力的隊友面前我能做的就是打雜了。不過在三天的學習過程中倒是瞭解到了不少工具：

- sqlmap
- ettercap
- whatweb
- knocker

賽後反思一下，我對Web安全方面的知識儲備嚴重匱乏，也沒有接觸過MySQL和PHP，以後要好好瞭解一下。這段時間要抽空看一下這幾本書：《SQL Injection Attacks and Defense》、《Web 前端黑客技術揭祕》、《白帽子講WEB安全》。

比賽持續48個小時，今天21:00結束。我們blue-lotus戰隊最後是第7名，據說要製作戰隊隊衫了，好期待啊！

## Shell utilities技巧

- plain hexdump: `xxd -p`
- reverse operation of plain hexdump: `xxd -r -p`。
  我想到的是個比較笨的方法：`ruby -ne 'print $1.scan(/../).map{|x|x.to_i(16).chr}.join`。
  不得不說邏輯稍煩雜的one-liner得數Ruby最好用，Smalltalk的cascading函數調用風格比Perl還好用不少。
- `tar`的選項`--to-command`，這個是第一次見。
- `curl`的`{}[]`有特殊含義，`-g`可以關掉這一特性，`-r`可用於指定`Range:` HTTP首部。
