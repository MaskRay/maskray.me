---
layout: post
title: webqqircd——用IRC客戶端控制WebQQ
author: MaskRay
tags: [irc,qq,webqqircd,python,javascript]
---

代碼：<https://github.com/MaskRay/webqqircd>

<!-- more -->

# webqqircd

webqqircd類似於bitlbee，在WebQQ(SmartQQ)和IRC間建起橋樑，可以使用IRC客戶端收發消息。大部分代碼來自[wechatircd](https://github.com/MaskRay/wechatircd)，爲適配QQ做了一些修改，去除了wechatircd中的token，因此只支持單客戶端。

## 原理

修改WebQQ(<http://w.qq.com>用的JS，通過WebSocket把信息發送到服務端，服務端兼做IRC服務端，把IRC客戶端的命令通過WebSocket傳送到網頁版JS執行。未實現IRC客戶端，因此無法把QQ羣的消息轉發到另一個IRC服務器(打通兩個羣的bot)。

## WebQQ侷限

- WebQQ不支持發送圖片，也無法獲悉別人發送了圖片
- 消息發送後不知道成功與否，`mq.model.chat`中`sendMsg(h)`的`onSuccess`爲空函數
- 無法獲知羣信息變化(如成員變化等)`mq.model.chat`中`addGroup(x)`只判斷羣存在與否，不判斷信息變化

## 安裝

需要Python 3.5或以上，支持`async/await`語法
`pip install -r requirements.txt`安裝依賴

Arch Linux可以安裝<https://aur.archlinux.org/packages/webqqircd-git>，會自動在`/etc/webqqircd/`下生成自簽名證書(見下文)，導入瀏覽器即可。

## 運行

### HTTPS、WebSocket over TLS

推薦使用TLS。

- `openssl req -newkey rsa:2048 -nodes -keyout a.key -x509 -out a.crt -subj '/CN=127.0.0.1' -dates 9999`創建密鑰與證書。
- Chrome訪問`chrome://settings/certificates`，導入a.crt，在Authorities標籤頁選擇該證書，Edit->Trust this certificate for identifying websites.
- Chrome安裝Switcheroo Redirector擴展，把<http://pub.idqqimg.com/smartqq/js/mq.js>重定向至<https://127.0.0.1:9002/mq.js>。若js更新，該路徑會變化。
- `./webqqircd.py --tls-cert a.crt --tls-key a.key`，會監聽127.1:6668的IRC和127.1:9002的HTTPS與WebSocket over TLS

![](https://maskray.me/static/2016-04-11-webqqircd/demo.jpg)

### HTTP、WebSocket

如果嫌X.509太麻煩的話可以不用TLS，但Chrome會在console裏給出警告。

- 執行`./webqqircd.py`，會監聽127.1:6668的IRC和127.1:9002的HTTP與WebSocket，HTTP用於伺服項目根目錄下的`mq.js`。
- 把<http://pub.idqqimg.com/smartqq/js/mq.js>重定向至<http://127.0.0.1:9002/mq.js>。若js更新，該路徑會變化。
- 把`mq.js` `var ws = new MyWebSocket('wss://127.0.0.1:9002')`行單引號裏面的部分修改成`ws://127.0.0.1:9002`

### IRC客戶端

- IRC客戶端連接127.1:6668(weechat的話使用`/server add qq 127.1/6668`)，會自動加入`+qq` channel
- 登錄<http://w.qq.com>
- 回到IRC客戶端，可以看到QQ朋友加入了`+qq` channel，在這個channel發信並不會羣發，只是爲了方便查看有哪些朋友。
- QQ朋友的nick優先選取備註名(`RemarkName`)，其次爲`DisplayName`(原始JS根據暱稱等自動填寫的一個名字)

在`+qq` channel可以執行一些命令：

- `help`，幫助
- `status`，已獲取的QQ朋友、羣列表
- `eval $password $expr`: 如果運行時帶上了`--password $password`選項，這裏可以eval，方便調試，比如`eval $password client.uin2qq_user`

若服務端或客戶端重啓，刷新WebQQ。

## IRC命令

webqqircd是個簡單的IRC服務器，可以執行通常的IRC命令，可以對其他客戶端私聊。

以下命令會有特殊作用：

- 程序默認選項爲`--join auto`，收到某個QQ羣的第一條消息後會自動加入對應的channel，即開始接收該QQ羣的消息。
- `/join [channel]`表示開始接收該QQ羣的消息
- `/list`，列出所有QQ羣
- `/names`，更新當前羣成員列表
- `/part [channel]`的IRC原義爲離開channel，轉換爲QQ代表在當前IRC會話中不再接收該QQ羣的消息。不用擔心，webqqircd並沒有主動退出羣的功能
- `/query nick`打開與`$nick`的私聊窗口，與之私聊即爲在QQ上和他/她/它對話
- `/who channel`，查看羣成員列表

## JS改動

原始文件`mq.js`在Chrome DevTools裏格式化後得到`orig/mq.pretty.js`，可以用`diff -u orig/mq.pretty.js mq.js`查看改動。

修改的地方都有`//@`標註，結合diff，方便WebQQ更新後重新應用這些修改。增加的代碼中大多數地方都用`try catch`保護，出錯則`consoleerr(ex.stack)`。

目前的改動如下：

### `mq.js`開頭

創建到服務端的WebSocket連接，若`onerror`則自動重連。監聽`onmessage`，收到的消息爲服務端發來的控制命令：`send_text_message`等。

### 定期把通訊錄發送到服務端

獲取所有聯繫人(朋友、訂閱號、羣)，`deliveredContact`記錄投遞到服務端的聯繫人，`deliveredContact`記錄同處一羣的非直接聯繫人。

每隔一段時間把未投遞過的聯繫人發送到服務端。

### 收到QQ服務器消息`messageProcess`

原有代碼會更新未讀標記數及聲音提醒，現在改爲若成功發送到服務端則不再提醒，以免瀏覽器的這個標籤頁造成干擾。

## Python服務端代碼

當前只有一個文件`webqqircd.py`，從miniircd抄了很多代碼，後來自己又搬了好多RFC上的用不到的東西……

```
.
├── Web                      HTTP(s)/WebSocket server
├── Server                   IRC server
├── Channel
│   ├── StandardChannel      `#`開頭的IRC channel
│   ├── StatusChannel        `+qq`，查看控制當前QQ會話
│   └── QQRoom               QQ羣對應的channel，僅該客戶端可見
├── (User)
│   ├── Client               IRC客戶端連接
│   ├── QQUser               QQ用戶對應的user，僅該客戶端可見
├── (IRCCommands)
│   ├── UnregisteredCommands 註冊前可用命令：NICK USER QUIT
│   ├── RegisteredCommands   註冊後可用命令
```

## 我的配置

<https://wiki.archlinux.org/index.php/Systemd/User>

`~/.config/systemd/user/webqqircd.service`:
```
[Unit]
Description=webqqircd
Documentation=https://github.com/MaskRay/webqqircd
After=network.target

[Service]
WorkingDirectory=%h/projects/webqqircd
ExecStart=/home/ray/projects/webqqircd/webqqircd.py --tls-key a.key --tls-cert a.crt --password a --ignore 不想自動加入的羣名0 不想自動加入的羣名1

[Install]
WantedBy=multi-user.target
```

WeeChat:
```
/server add qq 127.1/6668 -autoconnect
```
