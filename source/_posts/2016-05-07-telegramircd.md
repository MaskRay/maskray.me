---
layout: post
title: telegramircd——用IRC客戶端控制Telegram
author: MaskRay
tags: [irc,telegram,telegramircd,python,javascript]
---

代碼：<https://github.com/MaskRay/telegramircd>

<!-- more -->

# telegramircd

telegramircd類似於bitlbee，在web.telegram.org和IRC間建起橋樑，可以使用IRC客戶端收發朋友、羣消息。

## 原理

修改<https://web.telegram.org>用的JS，通過WebSocket把信息發送到服務端，服務端兼做IRC服務端，把IRC客戶端的命令通過WebSocket傳送到網頁版JS執行。未實現IRC客戶端，因此無法把羣的消息轉發到另一個IRC服務器(打通兩個羣的bot)。

## 安裝

需要Python 3.5或以上，支持`async/await`語法
`pip install -r requirements.txt`安裝依賴

### Arch Linux

安裝<https://aur.archlinux.org/packages/telegramircd-git>，會自動在`/etc/telegramircd/`下生成自簽名證書(見下文)，導入瀏覽器即可。

### 其他發行版

- `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -out cert.pem -subj '/CN=127.0.0.1' -dates 9999`創建密鑰與證書。
- Chrome訪問`chrome://settings/certificates`，導入cert.pem，在Authorities標籤頁選擇該證書，Edit->Trust this certificate for identifying websites.
- Chrome安裝Switcheroo Redirector擴展，把<https://web.telegram.org/js/app.js>重定向至<https://127.0.0.1:9003/app.js>。
- `./telegramircd.py --tls-cert cert.pem --tls-key key.pem`，會監聽127.1:6669的IRC和127.1:9003的HTTPS(兼WebSocket over TLS)

## IRC客戶端

- IRC客戶端連接127.1:6669
- 刷新<https://web.telegram.org>頁面
- 回到IRC客戶端，會發現自動加入了`+telegram` channel

在`+telegram`發信並不會羣發，只是爲了方便查看有哪些朋友。

## IRC命令

telegramircd是個簡單的IRC服務器，可以執行通常的IRC命令，可以對其他客戶端私聊，創建standard channel(以`#`開頭的channel)。另外若用token與某個微信網頁版連接的，就能看到微信聯繫人(朋友、羣聯繫人)顯示爲特殊nick、羣顯示爲特殊channel(以`&`開頭，根據羣名自動設置名稱)

這些特殊nick與channel只有當前客戶端能看到，因此一個服務端支持多個微信帳號同時登錄，每個用不同的IRC客戶端控制。另外，以下命令會有特殊作用：

- 程序默認選項爲`--join auto`，收到某個羣的第一條消息後會自動加入對應的channel，即開始接收該羣的消息。
- `/dcc send nick/channel filename`，給mutual friend或羣發圖片/文件。參見<https://en.wikipedia.org/wiki/Direct_Client-to-Client#DCC_SEND>
- `/list`，列出所有羣
- `/names`，更新當前羣成員列表
- `/part [channel]`的IRC原義爲離開channel，轉換爲微信代表在當前IRC會話中不再接收該羣的消息。不用擔心，telegramircd並沒有主動退出羣的功能
- `/query nick`打開與`$nick`的私聊窗口，與之私聊即爲在微信上和他/她/它對話
- `/who channel`，查看羣成員列表

## 顯示

![](https://maskray.me/static/2016-05-07-telegramircd/telegramircd.jpg)

vte終端模擬器支持URL選擇，但不能識別`filesystem:https://`。我修改的`aur/vte3-ng-fullwidth-emoji`添加了該類URL支持。

termite `C-S-Space` URL選擇也不支持，可以用<https://gist.github.com/MaskRay/9e1c57642bedd8b2b965e39b2d58fc82>添加該類URL支持。感谢张酉夫的ELF hack指导。
