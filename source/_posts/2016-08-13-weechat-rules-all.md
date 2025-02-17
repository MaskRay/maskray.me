---
layout: post
title: WeeChat操作各種聊天軟件
author: MaskRay
tags: [irc]
---

## WeeChat

把Facebook Messenger、Google Hangouts、IRC、QQ、Telegram、Twitter、微信等整合起來，用同一個客戶端WeeChat控制。配置文件：[https://github.com/MaskRay/Config/tree/master/home/.weechat/]()，參考<https://pascalpoitras.com/my-weechat-configuration/>。

![效果圖](/static/2016-08-13-weechat-rules-all/weechat.jpg)

界面參考<https://gist.github.com/pascalpoitras/8406501>，UI顯示成這樣主要靠`autosort.py`和`buffers.pl`，以及文中提到的Bar input。

<!-- more -->

### autosort.py

把相同server的buffers順序編號

### bot2human.py

```
/set python.bot2human.bot_nicks "tg2arch tox2sync xmppbot teleboto teleboto_ Orizon NyaaTelegram tg2gzlug d3tgbot tg2linuxba NyaaCat aosc-telegram XiaoT_bot"
```

### buffers.pl

窗口左側sidebar顯示buffers

我的wechatircd/telegramircd使用`--join new`，`/part`一個channel後下次收到消息仍能自動加入，覺得一個羣煩了就用`/mode +m`屏蔽，下次收到消息就不會自動加入了。

### highmon.pl

參考<https://pascalpoitras.com/2013/08/09/weechat-highlight/>

```
/set weechat.look.highlight security,update
/set weechat.look.highlight_regex .*\bHi\b.*\bplease\b.*  # 對所有buffer有效
/autosetbuffer add irc.bitlbee.#twitter highlight_regex . # 安裝buffer_autoset.py，highlight所有bitlbee的`#twitter` buffer中內容。
```

匹配highlight規則的行會顯示在highmon窗口裏。

### `notify.py`

### `pastebin.py`

我的插件：<https://github.com/MaskRay/Config/blob/master/home/.weechat/python/autoload/pastebin.py>，用於上傳本地文件到pastebin並把鏈接貼到當前buffer。

`/paste /tmp/a.jpg`用img.vim-cn.com圖牀分享圖片，在當前buffer插入鏈接

`/paste /tmp/a.txt`用cfp.vim-cn.com pastebin分享文件，在當前buffer插入鏈接

## Slack

讓team owner開啓IRC gateway：<https://get.slack.help/hc/en-us/articles/201727913-Connecting-to-Slack-over-IRC-and-XMPP>

## Bitlbee

安裝aur/bitlbee-libpurple-unicode-channel(或原版aur/bitlbee-libpurple)

參考<https://www.bitlbee.org/user-guide.html#set_utf8_nicks>
在`&bitlbee` buffer裏輸入
```
set utf8_nicks true
set nick_format %full_name
```

```
set show_offline true
```

`/etc/bitlbee/bitlbee.conf`中`[settings]` section的`Proxy`可以設置proxy。

### 註冊

`&bitlbee` buffer:

- `register`
- `/oper any $password`

### Facebook Messenger

`aur/bitlbee-facebook-git`

<https://wiki.bitlbee.org/HowtoFacebookMQTT>

創建一個channel顯示所有聯繫人：
```
/j &facebook
channel facebook set account facebook
channel facebook set show_users 'online@,special%,away+,offline'
```

### Google Hangouts

<https://wiki.bitlbee.org/HowtoGtalk>

支持OAuth認證，不需要輸入密碼。

創建一個channel顯示所有聯繫人：
```
/j &gtalk
channel gtalk set account gtalk
channel gtalk set show_users 'online@,special%,away+,offline'
```

### Twitter

<https://wiki.bitlbee.org/HowtoTwitter>

Google Hangouts的聯繫人，`&bitlbee`中nick會優先使用Contacts裏設置的備註名。設置`utf8_nicks`後就不會顯示爲亂碼。

## 微信網頁版

<https://github.com/MaskRay/wechatircd>

如果需要定製命令行選項的話，把`/usr/lib/systemd/system/wechatircd.service`複製到`/etc/systemd/system/wechatircd.service`，添加`--ignore 不想看到的羣名子串1 不想看到的羣名子串2`選項

```zsh
systemctl enable --now wechatircd
```

```
/server add wechat 127.1/6667 -autoconnect
```

## QQ網頁版

<https://github.com/MaskRay/webqqircd>

```zsh
systemctl enable --now webqqircd
```

```
/server add qq 127.1/6668 -autoconnect
```

如果需要定製命令行選項的話，把`/usr/lib/systemd/system/webqqircd.service`複製到`/etc/systemd/system/webqqircd.service`，添加`--ignore 不想看到的羣名子串1 不想看到的羣名子串2`選項

## Telegram網頁版

<https://github.com/MaskRay/telegramircd>

```zsh
systemctl enable --now telegramircd
```

如果需要定製命令行選項的話，把`/usr/lib/systemd/system/telegramicd.service`複製到`/etc/systemd/system/telegramicd.service`，添加`--ignore 不想看到的羣名子串1 不想看到的羣名子串2`選項

## 服務器上運行WeeChat

WeeChat不支持以WeeChat为客户端的relay。浏览器desktop notification不方便定制，显示效果也差，快捷键不够方便，因此不想用Glowing Bear。

把wechatircd和瀏覽器運行在服務器上，可以持續運行無需每日掃碼，參見<https://maskray.me/blog/2016-07-06-wechatircd-webqqircd-without-scanning-qrcode-daily>。

我讓WeeChat也運行在服務器的一個名爲weechat的tmux session，本地用ssh訪問：
```
SSH_AUTH_SOCK= ssh -R 9010:0:9010 -tX linode-ca 'tmux a -t weechat'
```

由於`notify-send`不能通過網絡傳輸，自制簡易的notification服務：

* 修改服務器`notify.py`插件，往127.0.0.1:9010發數據
* `ssh -R 9010:0:9010`把服務器9010重定向到本地9010
* 本地啓動<https://github.com/MaskRay/Config/blob/master/home/bin/notify-server>監聽9010，收到數據後執行`notify-send`
