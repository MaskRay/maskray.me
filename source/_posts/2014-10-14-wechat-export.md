---
layout: post
title: Android微信數據導出
author: MaskRay
tags: [wechat, forensics]
---

在Nexus 5(Android 4.4)+WeChat 5.4，和Nexus 5(Android 5.0)+Wechat 6.0上測試可用。

## 獲取加密的sqlite3數據庫`EnMicroMsg.db`

如果已經root過，可以下載`/data/data/com.tencent.mm/MicroMsg/*/EnMicroMsg.db`。

若沒有root，則`/data/data/com.tencent.mm`下多數目錄都不可讀，可以使用下面的方法：

- 開啓“開發人員選項”，選上“USB偵錯”
- 電腦上執行`adb backup -noapk com.tencent.mm`
- 在手機上彈出對話框提示是否允許備份
- 不要設置密碼，點備份，電腦會收到`backup.ab`
- 解壓`backup.ab`：`dd if=backup.ab bs=24 skip=1 | openssl zlib -d > backup.tar`
- 解壓`backup.tar`得到數據庫`apps/com.tencent.mm/r/MicroMsg/*/EnMicroMsg.db`

<!-- more -->

## 獲取用於生成密鑰的信息

- uin：訪問`/data/data/com.tencent.mm/MicroMsg/*/system_config_prefs.xml`，獲取其中`name="default_uin" value="([0-9]+)"`的`value`字段值`uin`。也可以打開wx.qq.com網頁版，查找`.wx.qq.com`域的cookie，其中`wxuin`字段的值就是`uin`。也可以用`backup.tar`裏的`apps/com.tencent.mm/sp/system_config_prefs.xml`。
- IMEI：在撥號盤輸入`*#06#`獲取IMEI，或者開啓“USB偵錯”後使用`adb shell dumpsys iphonesubinfo`得到15個十進制數字組成的`imei`。網上查到有些機型可能使用不同於IMEI的其他字段用於生成密鑰。

## 使用sqlcipher解密

把上面兩步得到的`imei`和`uin`拼接起來計算MD5。執行`echo -n "$imei$uin" | md5sum | cut -c -7`獲取sqlcipher使用的加密密鑰，下面用`abcdefg`指代。

執行`sqlcipher EnMicroMsg.db`，輸入：

```sql
PRAGMA key='abcdefg';
PRAGMA cipher_use_hmac = off;
ATTACH DATABASE "decrypted_database.db" AS decrypted_database KEY "";
SELECT sqlcipher_export("decrypted_database");
DETACH DATABASE decrypted_database;
```

解密得到可用sqlite3打開的`decrypted_database.db`。

注意，sqlcipher不同版本使用的加密方式不同，我嘗試使用3.8.4.3版本打開數據庫文件，得到如下錯誤信息：

```
sqlite> PRAGMA key='abcdefg';
sqlite> .schema
Error: file is encrypted or is not a database
```

目前發現2.1.1版本的sqlcipher可以解密。可以下載[https://github.com/CovenantEyes/sqlcipher-windows/releases](https://github.com/CovenantEyes/sqlcipher-windows/releases)提供的2.1.1的Windows用exe，用wine運行，或者在[https://launchpad.net/ubuntu/+source/sqlcipher/2.1.1-2/+build/4642377](https://launchpad.net/ubuntu/+source/sqlcipher/2.1.1-2/+build/4642377)上下載`libsqlcipher0_2.1.1-2_amd64.deb`和`sqlcipher_2.1.1-2_amd64.deb`，執行：

```bash
# /tmp/sqlcipher_2.1.1-2_amd64.deb
# /tmp/libsqlcipher0_2.1.1-2_amd64.deb
cd /tmp
# get /tmp/usr/bin/sqlcipher
ar x sqlcipher_2.1.1-2_amd64.deb && tar xf data.tar.gz --no-overwrite-dir
# get /tmp/usr/lib/x86_64-linux-gnu/libsqlcipher.so.0.8.6
ar x libsqlcipher0_2.1.1-2_amd64.deb && tar xf data.tar.gz --no-overwrite-dir
```

解壓後執行：

```bash
cd /tmp/usr && LD_LIBRARY_PATH=lib/x86_64-linux-gnu bin/sqlcipher /tmp/EnMicroMsg.db
```

## 解析`message`表並導出消息

`message`表儲存消息。目前瞭解到從`fmessage_conversation`、`rcontact`和`chatroom`表中可以得到一些聯繫人和聊天室的信息。

暫時使用一個比較粗糙的Ruby腳本導出信息，需要先`gem install sqlite3`：

```ruby
require 'sqlite3'

begin
  talker2name = {}
  username2name = {}

  db = SQLite3::Database.open '/tmp/decrypted_database.db'
  db.results_as_hash = true
  db.execute('SELECT talker,displayName FROM fmessage_conversation').each {|row|
    talker2name[row['talker']] = row['displayName']
  }
  db.execute('SELECT username,nickname FROM rcontact').each {|row|
    username = row['username']
    nickname = row['nickname']
    if nickname != ''
      if username =~ /@chatroom$/
        talker2name[username] = nickname == '' ? username : nickname
      else
        username2name[username] = nickname == '' ? username : nickname
      end
    end
  }

  db.execute('SELECT createTime,talker,content FROM message').each {|row|
    time,talker,content = row.values_at 'createTime','talker','content'
    next unless content
    if talker =~ /@chatroom$/
      content.sub!(/^(\w+):\n/) {|x| "#{username2name.fetch($1,'xx')}: " }
    end
    #next if content =~ /^~SEMI_XML~|</
    next if content =~ /^~SEMI_XML~/
    name = talker2name.fetch talker, talker
    puts "#{Time.at(time/1000).strftime('%FT%R')}\t#{name}\t#{content}"
  }
rescue SQLite3::Exception => e
  puts e
ensure
  db.close if db
end
```

## Refenrences

- <https://gist.github.com/shellexy/7246798>
- <http://nelenkov.blogspot.jp/2012/06/unpacking-android-backups.html>
