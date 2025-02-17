---
layout: post
title: 我是這樣使用微信的——wechatircd最近更新
author: MaskRay
tags: [irc,wechat,wechatircd,python,javascript]
---

既然你誠心誠意地問了，我就大發慈悲地告訴你～

最近給wechatircd(讓IRC客戶端控制微信網頁版，收發文本圖片、邀請刪除羣成員、添加朋友請求等)做大手術，折騰了很多東西……可以看[這段視頻](https://asciinema.org/a/636dkay05bpzci1idf3e84y1y)以及最新的[README.md](https://github.com/MaskRay/wechatircd/blob/dev/README.zhs.md)。

<!-- more -->

<script type="text/javascript" src="https://asciinema.org/a/636dkay05bpzci1idf3e84y1y.js" id="asciicast-636dkay05bpzci1idf3e84y1y" async></script>

- server建立和微信網頁版的綁定，而不是之前的client登錄後和網頁綁定
- 全面修繕多clients支持，多個clients可以操作一個微信個人帳號，可以外接bot
- 命令行選項`--irc-nicks ray ray1`，給客戶端保留的nick。防止微信用戶佔用這些名字
- 新的join mode：`--join new`。收到某個羣第一條消息後自動加入對應的channel。WeeChat裏`/close`命令關閉窗口會自動`/part`。原來的`--join auto`是收到`/part`命令後，收到消息不會重新加入。
- 現在通過設置channel mode來防止自動JOIN一個惱人的微信羣：`/mode +m`
- `/motd`，查看本repo最近5個commits
- `/squit $any`，log out微信網頁版
- 微信朋友帶有mode `+v` (voice, 通常顯示爲前綴`+`)。IRC客戶端nicklist裏把這些nick會用前綴+顯示。`SpecialChannel#update_detail`
- `nick0: nick1: test`會被轉換成`@GroupAlias0 @GroupAlias1 test`，`GroupAlias0` 是那個用戶自己設置的名字，不是你設置的`Set Remark and Tag`，對應移動端的`On-screen names`
- 回覆12:34:SS的消息：`@1234 !m multi\nline\nreply`，會發送`「Re GroupAlias: text」text`
- 回覆12:34:56的消息：`!m @123456 multi\nline\nreply`
- 回覆朋友/羣的倒數第二條消息(自己的消息不計數)：`@2 reply`
- 粘貼檢測。PRIVMSG行會被延遲0.1秒，期間發送的所有行會被打包成一個多行消息發送
- `!m `, `@3 `, `nick: `可以任意安排順序。
- IRC 3.1 3.2的`server-time`，讓客戶端顯示消息時和服務器收到的消息的時刻一致。
- 订阅号文章统一由BrandServ虚拟用户发送
- -c/--config可以接配置文件(代替命令行选项)
- `--irc-nicks ray ray1`保留给IRC客户端用的nick，不会被虚拟的微信用户占用


對於WeeChat，默認的anti-flood機制會讓發出去的兩條消息間隔至少2秒。禁用該機制使粘貼檢測生效：
```
/set irc.server.wechat.anti_flood_prio_high 0
```

如果想用這種方式操作Telegram，请移步<https://github.com/MaskRay/telegramircd>
我的WeeChat配置參見[WeeChat操作各種聊天軟件](/blog/2016-08-13-weechat-rules-all)

去年折騰這個就是嫌微信操作麻煩，這兩年各種微信bot方案層出不窮，比如最近活躍的<https://github.com/wechaty/wechaty>、<https://github.com/littlecodersh/ItChat/>，我更喜歡的還是sjdy521的<https://github.com/sjdy521/Mojo-Weixin>，也提供了一個IRC插件(可惜我自己有很多其他需求，這個項目IRC插件部分比較簡單)。然而他們從頭實現微信網頁版協議，代碼量巨大……我不懂這些前端技術，也沒有足夠精力持續擁抱微信網頁版變化，最初無奈下載源碼後加一些自己的patch，演化到現在抄electronic-wechat注入angular，用Userscript來驅動。目前`injector.js`只有600多行，很大一部分是從微信網頁版挖出來的。

> 對於黑客來說，有這麼一條鐵律：從頭開發個軟件只是小兒科；要想在別人面前展現自己的聰明才智（並讓別人對自己留下印象），就得做點更有意思的事：改進一個程序讓它體積更小、跑得更快、功能更強，代碼更優雅，這纔是真本事。——《若爲自由故：自由軟件之父理查德·斯托曼傳》

現在支持多客戶端後，可以一邊用IRC聊天，一邊用一個bot自動化一些操作。IRC bot非常多，如Sopel等，能找到豐富的玩法。

另外，建議使用`README.zhs.md`中提到的headless瀏覽器方案，可以避免每日掃碼。

- 創建一個新Chromium profile：`chromium --user-data-dir=$HOME/.config/chromium-wechatircd`，配置瀏覽器(`injector.js`用的證書, Tampermonkey, `injector.user.js`)，關閉瀏覽器
- 安裝xvfb (Arch Linux裏叫`xorg-server-xvfb`)
- `xvfb-run -n 99 chromium --user-data-dir=$HOME/.config/chromium-wechatircd https://wx.qq.com`
- 等數秒待QR碼加載。`DISPLAY=:99 import -window root /tmp/a.jpg && $your_image_viewer /tmp/a.jpg`，截圖後用移動端應用掃碼

另一種方案是x2go，參見[無需每日掃碼的IRC版微信和QQ：wechatircd、webqqircd](https://maskray.me/blog/2016-07-06-wechatircd-webqqircd-without-scanning-qrcode-daily)。

討論羣：

- freenode channel #wechatircd
- Telegram <https://t.me/wechatircd>
- Gitter <https://gitter.im/wechatircd/wechatircd>

祝wechatircd一歲生日快樂。就是這樣，喵喵喵，喵喵喵～
