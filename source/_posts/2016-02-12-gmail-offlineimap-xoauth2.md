---
layout: post
title: Gmail的OfflineIMAP XOAUTH2認證
author: MaskRay
tags: [email]
---

Gmail現在似乎不再允許IMAP的AUTHENTICATE PLAIN了。2016年1月的OfflineIMAP 6.7.0-rc1對XOAUTH2支持較好，仍可訪問Gmail IMAP。

<!-- more -->

## Gmail網頁版啓用IMAP

<https://mail.google.com>中Settings->Forwarding and POP/IMAP，開啓IMAP Access，When a message is marked as deleted and expunged from the last visible IMAP folder:最好選擇默認的Archive the message，被刪除的郵件在網頁版中會丟失所有標籤，但仍能在“[Gmail]/All Mail”中找到。

- 訪問[Google Developers Console](https://console.developers.google.com/apis/credentials)
- 選擇或新建一個project
- 訪問https://console.developers.google.com/apis/credentials，點擊Create credentials->OAuth client ID，type選擇other，記錄client ID與client secret

Gmail的label大致被映射成IMAP的folder，以下是一些默認label的IMAP folder名稱：
```
INBOX
[Gmail]/Trash
[Gmail]/Important
[Gmail]/Sent Mail
[Gmail]/Starred
[Gmail]/Spam
[Gmail]/All Mail
[Gmail]/Personal
```

## OfflineIMAP XOAUTH2配置

編輯`~/.offlineimaprc`：
```
[general]
ui = ttyui
accounts = Ray
fsync = False

[Account Ray]
localrepository = Ray-Local
remoterepository = Ray-Remote
status_backend = sqlite
#要用代理的话得pip install pysocks
#proxy = HTTP:127.0.0.1:1111
#proxy = SOCKS5:127.0.0.1:1112

[Repository Ray-Local]
type = Maildir
localfolders = ~/Maildir

[Repository Ray-Remote]
type = Gmail
ssl = yes
sslcacertfile = /etc/ssl/certs/ca-certificates.crt
remoteuser = youremail@example.com
auth_mechanisms = XOAUTH2
oauth2_client_id = 7738xxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com
oauth2_client_secret = xxxxxx-xxxxxxxxxxxxxxxxx
oauth2_refresh_token = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
#oauth2_access_token = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

nametrans = lambda folder: {'[Gmail]/Drafts':    'Drafts',
                            '[Gmail]/Sent Mail': 'Sent',
                            '[Gmail]/Starred':   'Starred',
                            '[Gmail]/Trash':     'Trash',
                            '[Gmail]/All Mail':  'Archive',
                            }.get(folder, folder)
folderfilter = lambda folder: folder in ['INBOX', 'Label1', 'Label2', 'Work']
```

默認配置文件<https://github.com/OfflineIMAP/offlineimap/blob/master/offlineimap.conf>有詳細的描述。

```bash
cd /tmp
git clone https://github.com/google/gmail-oauth2-tools
cd gmail-oauth2-tools
# 獲取refresh token，填入~/.offlineimaprc的oauth2_refresh_token
# 這一步需要藉由各種代理途徑，比如proxychains等
python2 python/oauth2.py --generate_oauth2_token --client_id=xxx --client_secret=xxx
```

## systemd user

OfflineIMAP的quick sync很不靠譜，SIGINT後也不能有效退出，往往要停滯很長時間，因此我用systemd user每個一段時間運行一次offlineimap，並設置較短的`TimeoutStopSec`(`man 5 systemd.service`)。編輯`~/.config/systemd/user/offlineimap@.service`：
```
[Unit]
Description=OfflineIMAP IMAP/Maildir Synchronization
Documentation=man:offlineimap

[Service]
ExecStart=/usr/bin/offlineimap -o -u quiet -a %i
Restart=always
RestartSec=600
TimeoutStopSec=10

[Install]
WantedBy=default.target
```

```bash
# Ray爲~/.offlineimaprc中配置的賬戶名
systemctl --user enable offlineimap@Ray
systemctl --user start offlineimap@Ray
```

## SMTP client

沒找到支持XOAUTH2、可用作Mail Submission Agent的SMTP client，暫時仍用msmtp的SMTP AUTH PLAIN，用[pass](https://www.passwordstore.org/)記錄Gmail密碼，用gpg的passphrase保護。編輯`~/.msmtprc`：

```
defaults
auth           on
#proxy_host     127.0.0.1
#proxy_port     1111
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt

account        Gmail
host           smtp.gmail.com
port           587
from           youremail@example.com
user           youremail@example.com
passwordeval   pass show Gmail
```

## 易犯錯誤

如果沒有用全局代理，使用`offlineimaprc`中的`proxy`選項，得確保安裝Python 2的`PySocks`包，不然代理設置會被忽略。

`imapserver.py`中`__xoauth2handler`用`urllib.urlopen`獲取access token，2016年1月的實現沒有用上`proxy`選項設置。`offlineimap-6.7.0`已經集成了該補丁。
