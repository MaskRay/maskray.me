---
layout: post
title: 無需每日掃碼的IRC版微信和QQ：wechatircd、webqqircd
author: MaskRay
tags: [irc,wechat,qq,wechatircd,webqqircd,javascript,python]
---

- <https://maskray.me/blog/2016-02-21-wechatircd>
- <https://maskray.me/blog/2016-04-11-webqqircd>

之前ftiasch提到webqqircd運行在服務器上，似乎可以持續運行無需每日掃碼。最近聽bigeagle推薦了x2go。於是實驗了一下headless wechatircd & webqqircd。PhantomJS不可用，存儲圖片麻煩，會拋出SIGFPE異常，自帶webkit舊不支持ES6。故棄用，嘗試Firefox。

<!-- more -->

服務器上安裝x2goserver，Arch Linux配置較爲容易：
```zsh
sudo pacman -S --needed firefox socat
sudo pacman -S --needed x2goserver
sudo x2godbadmin --createdb
sudo systemctl enable --now x2goserver
```

```zsh
yaourt -S wechatircd-git webqqircd-git
sudo systemctl enable --now wechatircd
sudo systemctl enable --now webqqircd
```

wechatircd、webqqircd的IRC server默認監聽127.0.0.1，本地無法直接連接，也沒有作身份認證。可以考慮用socat、iptables等方法限制只有自己能訪問。

- 本地安裝x2goclient，New session，Session type選Single application，Command:爲firefox。連接後稍等片刻(網速)會看到本地出現firefox窗口，該進程運行在服務器上。
- 安裝<https://addons.mozilla.org/en-US/firefox/addon/redirector/>擴展
- 地址欄找到該擴展的圖標，點擊配置
- 根據<https://github.com/MaskRay/wechatircd>與<https://github.com/MaskRay/webqqircd>上的說明重定向qq.com域名上的js到127.0.0.1下wechatircd、webqqircd的修改版本，注意設置Applies to: Main window (address bar), Scripts
- 訪問兩個js連接，Firefox提示Your connection is not secure，添加例外
- 直接關閉x2goclient，效果似乎和suspend等效，進程仍在服務器上運行

![Firefox Redirector配置](/static/2016-07-06-wechatircd-webqqircd-without-scanning-qrcode-daily/firefox.jpg)

微信網頁版只要synccheck不間斷(約30秒一次)，不會斷開連接要求重新掃碼二維碼。QQ網頁版類似。

QQ網頁版不能顯示圖片。
微信網頁版能顯示圖片、視頻、音頻、文件、位置等，wechatircd會把它們轉成qq.com域名的鏈接。本地瀏覽器沒有cookies無法訪問這些鏈接，我寫了一個腳本把服務器Firefox的cookies(sqlite3)同步到本地，把qq.com相關域名的cookies導入到Chrome(也是sqlite3)：<https://gist.github.com/MaskRay/fae75a66f707d774b2335f61701221e8>。

![](/static/2016-07-06-wechatircd-webqqircd-without-scanning-qrcode-daily/wechatircd.jpg)

某些客戶端可以預覽圖片。

使用微信是無奈，警惕它打造的局域網。這種方式可以用來做普通帳號的微信機器人。

## 附錄

systemd service文件`wechatircd.service`:
```systemd
[Unit]
Description=IRC server capable of controlling Web WeChat
Documentation=https://github.com/MaskRay/wechatircd
After=network.target

[Service]
ExecStart=/usr/bin/wechatircd --join new --http-key /etc/wechatircd/key.pem --http-cert /etc/wechatircd/cert.pem --http-root /usr/share/wechatircd --password a --logger-mask '/home/ray/irclogs/wechatircd/$channel/%%Y-%%m-%%d.log' --ignore 不想自動加入的羣組名子串 另一個不想自動加入的羣組名子串

[Install]
WantedBy=multi-user.target
```
