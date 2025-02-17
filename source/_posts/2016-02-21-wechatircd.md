---
layout: post
title: wechatircd——用IRC客戶端控制微信網頁版
author: MaskRay
tags: [irc,wechat,wechatircd,python,javascript]
---

內容可能過時，最新文檔參見：<https://github.com/MaskRay/wechatircd>。

# wechatircd

wechatircd在wx.qq.com裏注入JavaScript，用WebSocket與IRC server(`wechatircd.py`)通信，使得IRC客戶端可以收發微信朋友、羣消息、設置羣名、邀請刪除成員等。

```
           IRC               WebSocket                 HTTPS
IRC client --- wechatircd.py --------- browser         ----- wx.qq.com
                                       injector.user.js
                                       injector.js
```

## 安裝

需要Python 3.5或以上，支持`async/await`語法
`pip install -r requirements.txt`安裝依賴

### Arch Linux

- `yaourt -S wechatircd-git`。會在`/etc/wechatircd/`下生成自簽名證書。
- 把`/etc/wechatircd/cert.pem`導入到瀏覽器(見下文)
- `systemctl start wechatircd`會運行`/usr/bin/wechatircd --http-cert /etc/wechatircd/cert.pem --http-key /etc/wechatircd/key.pem --http-root /usr/share/wechatircd`

IRC服務器默認監聽127.0.0.1:6667 (IRC)和127.0.0.1:9000 (HTTPS + WebSocket over TLS)。

如果你在非本機運行，建議配置IRC over TLS，設置IRC connection password：`/usr/bin/wechatircd --http-cert /etc/wechatircd/cert.pem --http-key /etc/wechatircd/key.pem --http-root /usr/share/wechatircd --irc-cert /path/to/irc.key --irc-key /path/to/irc.cert --irc-password yourpassword`

可以把HTTPS私鑰證書用作IRC over TLS私鑰證書。使用WeeChat的話，如果覺得讓WeeChat信任證書比較麻煩(gnutls會檢查hostname)，可以用：
```
set irc.server.wechat.ssl on`
set irc.server.wechat.ssl_verify off
set irc.server.wechat.password yourpassword`
```

### 其他發行版

- `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -out cert.pem -subj '/CN=127.0.0.1' -days 9999`創建密鑰與證書。
- 把`cert.pem`導入瀏覽器，見下文
- `./wechatircd.py --http-cert cert.pem --http-key key.pem`

### 導入自簽名證書到瀏覽器

Chrome/Chromium

- 訪問`chrome://settings/certificates`，導入`cert.pem`，在Authorities標籤頁選擇該證書，Edit->Trust this certificate for identifying websites.
- 安裝Tampermonkey擴展，點擊<https://github.com/MaskRay/wechatircd/raw/master/injector.user.js>安裝userscript，效果是在<https://wx.qq.com>頁面注入<https://127.0.0.1:9000/injector.js>

Firefox

- 訪問<https://127.0.0.1:9000/injector.js>，報告Your connection is not secure，Advanced->Add Exception->Confirm Security Exception
- 安裝Greasemonkey擴展，安裝userscript

![](https://maskray.me/static/2016-02-21-wechatircd/2017-02-17.jpg)

HTTPS、WebSocket over TLS默認用9000端口，使用其他端口需要修改userscript，啓動`wechatircd.py`時用`--web-port 10000`指定其他端口。

## 使用

- 運行`wechatircd.py`
- 訪問<https://wx.qq.com>，userscript注入的JavaScript會向服務器發起WebSocket連接
- IRC客戶端連接127.1:6667(weechat的話使用`/server add wechat 127.1/6667`)，會自動加入`+wechat` channel

在`+wechat`發信並不會羣發，只是爲了方便查看有哪些朋友。
微信朋友的nick優先選取備註名(`RemarkName`)，其次爲`DisplayName`(原始JS根據暱稱等自動填寫的一個名字)

在`+wechat` channel可以執行一些命令：

- `help`，幫助
- `status [pattern]`，已獲取的微信朋友、羣列表，支持 pattern 參數用來篩選滿足 pattern 的結果，目前僅支持子串查詢。如要查詢所有羣，由於羣由 `&` 開頭，所以可以執行 `status &`。
- `eval $password $expr`: 如果運行時帶上了`--password $password`選項，這裏可以eval，方便調試，比如`eval $password client.wechat_users`

## 服務器選項

- Join mode，短選項`-j`
  + `--join auto`，默認：收到某個羣第一條消息後自動加入，如果執行過`/part`命令了，則之後收到消息不會重新加入
  + `--join all`：加入所有channel
  + `--join manual`：不自動加入
  + `--join new`：類似於`auto`，但執行`/part`命令後，之後收到消息仍自動加入
- 指定不自動加入的羣名，用於補充join mode
  + `--ignore 'fo[o]' bar`，channel名部分匹配正則表達式`fo[o]`或`bar`
  + `--ignore-topic 'fo[o]' bar`, 羣標題部分匹配正則表達式`fo[o]`或`bar`
- HTTP/WebSocket相關選項
  + `--http-cert cert.pem`，HTTPS/WebSocketTLS的證書。你可以把證書和私鑰合併爲一個文件，省略`--http-key`選項。如果`--http-cert`和`--http-key`均未指定，使用不加密的HTTP
  + `--http-key key.pem`，HTTPS/WebSocket的私鑰
  + `--http-listen 127.1 ::1`，HTTPS/WebSocket監聽地址設置爲`127.1`和`::1`，overriding `--listen`
  + `--http-port 9000`，HTTPS/WebSocket監聽端口設置爲9000
  + `--http-root .`, 存放`injector.js`的根目錄
- `-l 127.0.0.1`，IRC/HTTP/WebSocket監聽地址設置爲`127.0.0.1`
- IRC相關選項
  + `--irc-cert cert.pem`，IRC over TLS的證書。你可以把證書和私鑰合併爲一個文件，省略`--irc-key`選項。如果`--irc-cert`和`--irc-key`均未指定，使用不加密的IRC
  + `--irc-key key.pem`，IRC over TLS的私鑰
  + `--irc-listen 127.1 ::1`，IRC over TLS監聽地址設置爲`127.1`和`::1`，overriding `--listen`
  + `--irc-nicks ray ray1`，給客戶端保留的nick。`SpecialUser`不會佔用這些名字
  + `--irc-password pass`，IRC connection password設置爲`pass`
  + `--irc-port 6667`，IRC監聽端口
- 服務端日誌
  + `--logger-ignore '&test0' '&test1'`，不記錄部分匹配指定正則表達式的朋友/羣日誌
  + `--logger-mask '/tmp/wechat/$channel/%Y-%m-%d.log'`，日誌文件名格式
  + `--logger-time-format %H:%M`，日誌單條消息的時間格式

## IRC命令

- 標準IRC channel名以`#`開頭
- WeChat羣名以`&`開頭。`SpecialChannel#update`
- 聯繫人帶有mode `+v` (voice, 通常顯示爲前綴`+`)。`SpecialChannel#update_detail`
- 多行消息：`!m line0\nline1`
- 多行消息：`!html line0<br>line1`
- `nick0: nick1: test`會被轉換成`@GroupAlias0 @GroupAlias1 test`，`GroupAlias0`是`nick0`在頻道里的Group Alias，如果沒設置的話就是Alias。`GroupAlias0`是移動端用戶看到的名字
- 回覆12:34:SS的消息：`@1234 !m multi\nline\nreply`，會發送`「Re GroupAlias: text」text`
- 回覆12:34:56的消息：`!m @123456 multi\nline\nreply`
- 回覆朋友/羣的倒數第二條消息：`@2 reply`

若客戶端啓用IRC 3.1 3.2的`server-time`擴展，`wechatircd.py`會在發送的消息中包含 網頁版獲取的時間戳。客戶端顯示消息時時間就會和服務器收到的消息的時刻一致。參見<http://ircv3.net/irc/>。參見<http://ircv3.net/software/clients.html>查看IRCv3的客戶端支持情況。

WeeChat配置方式：
```
/set irc.server_default.capabilities "account-notify,away-notify,cap-notify,multi-prefix,server-time,znc.in/server-time-iso,znc.in/self-message"
```

支持的IRC命令：

- `/cap`，列出支持的capabilities
- `/dcc send $nick/$channel $filename`, send image or file。This feature borrows the command `/dcc send` which is well supported in IRC clients. See <https://en.wikipedia.org/wiki/Direct_Client-to-Client#DCC_SEND>.
- `/invite $nick [$channel]`, invite a contact to the group.
- `/kick $nick`，刪除羣成員，羣主纔有效。由於網頁版限制，可能收不到羣成員變更的消息
- `/list`，列出所有羣
- `/mode +m`, `--join new`模式下防止自動重新join。用`/mode -m`撤銷
- `/names`, 更新當前羣成員列表
- `/part $channel`的IRC原義爲離開channel，這裏表示當前IRC會話中不再接收該羣的消息。不用擔心，telegramircd並沒有主動退出羣的功能
- `/query $nick`，打開和`$nick`聊天的窗口
- `/summon $nick $message`，發送添加朋友請求，`$message`爲備註消息
- `/topic topic`修改羣標題。因爲IRC不支持channel改名，實現爲離開原channel並加入新channel
- `/who $channel`，查看羣的成員列表

![](https://maskray.me/static/2016-02-21-wechatircd/topic-kick-invite.jpg)

### 顯示

- `MSGTYPE_TEXT`，文本或是視頻/語音聊天請求，顯示文本
- `MSGTYPE_IMG`，圖片，顯示`[Image]`跟URL
- `MSGTYPE_VOICE`，語音，顯示`[Image]`跟URL
- `MSGTYPE_VIDEO`，視頻，顯示`[Video]`跟URL
- `MSGTYPE_MICROVIDEO`，微視頻?，顯示`[MicroVideo]`跟URL
- `MSGTYPE_APP`，訂閱號新文章、各種應用分享送紅包、URL分享等屬於此類，還有子分類`APPMSGTYPE_*`，顯示`[App]`跟title跟URL

QQ表情會顯示成`<img class="qqemoji qqemoji0" text="[Smile]_web" src="/zh_CN/htmledition/v2/images/spacer.gif">`樣，發送時用`[Smile]`即可(相當於在網頁版文本輸入框插入文本後點擊發送)。

Emoji在網頁上呈現時爲`<img class="emoji emoji1f604" text="_web" src="/zh_CN/htmledition/v2/images/spacer.gif">`，傳送至IRC時轉換成單個emoji字符。若使用終端IRC客戶端，會因爲emoji字符寬度爲1導致重疊，參見[終端模擬器下使用雙倍寬度多色Emoji字體](https://maskray.me/blog/2016-03-13-terminal-emulator-fullwidth-color-emoji)。

## JS改動

修改的地方都有`//@`標註，結合diff，方便微信網頁版JS更新後重新應用這些修改。增加的代碼中大多數地方都用`try catch`保護，出錯則`consoleerr(ex.stack)`。原始JS把`console`對象抹掉了……`consoleerr`是我保存的一個副本。

目前的改動如下：

### `webwxapp.js`開頭

創建到服務端的WebSocket連接，若`onerror`則自動重連。監聽`onmessage`，收到的消息爲服務端發來的控制命令：`send_text_message`、`add_member`等。

### Hook contactFactory#addContact以記錄聯繫人列表的變更

## Python服務端代碼

當前只有一個文件`wechatircd.py`，從miniircd抄了很多代碼，後來自己又搬了好多RFC上的用不到的東西……

```
.
├── Web                      HTTP(s)/WebSocket server
├── Server                   IRC server
├── Channel
│   ├── StandardChannel      `#`開頭的IRC channel
│   ├── StatusChannel        `+wechat`，查看控制當前微信會話
│   └── SpecialChannel       微信羣對應的channel，僅該客戶端可見
├── (User)
│   ├── Client               IRC客戶端連接
│   ├── SpecialUser          微信用戶對應的user，僅該客戶端可見
├── (IRCCommands)
│   ├── UnregisteredCommands 註冊前可用命令：NICK USER QUIT
│   ├── RegisteredCommands   註冊後可用命令
```

## FAQ

### 使用這個方法的理由

原本想研究微信網頁版登錄、收發消息的協議，自行實現客戶端。參考過<https://github.com/0x5e/wechat-deleted-friends>，仿製了<https://gist.github.com/MaskRay/3b5b3fcbccfcba3b8f29>，可以登錄。但根據minify後JS把相關部分重寫非常困難，錯誤處理很麻煩，所以就讓網頁版JS自己來傳遞信息。

### 用途

可以使用強大的IRC客戶端，方便記錄日誌(微信日誌導出太麻煩<https://maskray.me/blog/2014-10-14-wechat-export>)，可以寫bot。

## 微信數據獲取及控制

少量特殊賬戶的`UserName`不帶`@`前綴：`newsapp,fmessage,filehelper,weibo,qqmail,fmessage`等的；一般賬戶(公衆號、服務號、直接聯繫人、羣友)的`UserName`以`@`開頭；微信羣的`UserName`以`@@`開頭。不同session `UserName`會變化。`Uin`應該是唯一id，但微信網頁版API多數時候都返回0，隱藏了真實值。
羣的`OwnerUin`字段是羣主的`Uin`，但大羣用戶的`Uin`通常都爲0，因此難以對應。

自己的帳號
```javascript
angular.element(document.body).scope().account
```

所有聯繫人列表
```javascript
angular.element($('#navContact')[0]).scope().allContacts
```

刪除羣中成員
```javascript
var injector = angular.element(document).injector()
# 這裏獲取了chatroomFactory，還可用於獲取其他factory、service、controller等
var chatroomFactory = injector.get('chatroomFactory')
# 設置其中的`room`與`userToRemove`
chatroomFactory.delMember(room.UserName, userToRemove.UserName)`
```

名稱中包含`xxx`的最近聯繫人列表中的羣
```javascript
angular.element($('span:contains("xxx")')).scope().chatContact
```

當前窗口發送消息
```javascript
angular.element('pre:last').scope().editAreaCtn = "Hello，微信";
angular.element('pre:last').scope().sendTextMessage();
```

## 已知問題

```
Uncaught TypeError: angular.extend is not a function
    at Object.setUserInfo (index_0c7087d.js:4)
    at index_0c7087d.js:2
    at c (vendor_2de5d3a.js:11)
    at vendor_2de5d3a.js:11
    at c.$eval (vendor_2de5d3a.js:11)
    at c.$digest (vendor_2de5d3a.js:11)
    at c.$apply (vendor_2de5d3a.js:11)
    at l (vendor_2de5d3a.js:11)
    at m (vendor_2de5d3a.js:11)
    at XMLHttpRequest.C.onreadystatechange (vendor_2de5d3a.js:11)
```

```
Uncaught TypeError: angular.forEach is not a function
```

`injector.js`得在`vendor_*.js`後`index_*.js`前執行。但TamperMonkey無法精細控制script的運行時機。

## 參考

- [miniircd](https://github.com/jrosdahl/miniircd)
- [RFC 2810: Internet Relay Chat: Architecture](https://tools.ietf.org/html/rfc2810)
- [RFC 2811: Internet Relay Chat: Channel Management](https://tools.ietf.org/html/rfc2811)
- [RFC 2812: Internet Relay Chat: Client Protocol](https://tools.ietf.org/html/rfc2812)
- [RFC 2813: Internet Relay Chat: Server Protocol](https://tools.ietf.org/html/rfc2813)
