---
layout: post
title: 建立清華大學Node Packaged Modules鏡像
author: MaskRay
tags: [npm, node.js, mirror]
---

## 緣由

Hacker News最近有篇文章[How to create a private npm.js repository](http://http://clock.co.uk/tech-blogs/how-to-create-a-private-npmjs-repository)，看完後打算給[http://mirror.tuna.tsinghua.edu.cn/]()搭個源。

## 搭建

- `sudo aptitude install couchdb`
- `sudo vim /etc/couchdb/local.ini`修改`admin`密碼
- `sudo install -d -o couchdb -g couchdb /var/run/couchdb`

<!-- more -->

然後配置CouchDB：

```
$ sudo vim /etc/couchdb/local.ini
[httpd]
secure_rewrites = false

[couchdb]
database_dir = /mirror/npm/couchdb # 設置couchdb數據庫存放路徑
```

這裏把數據庫目錄設爲`/mirror/npm/couchdb`了。
2013年8月17日，同步完成後這個目錄下面的`registry.couch`有72G。

- `sudo /etc/init.d/couchdb start`

然後開始同步，CouchDB的管理是通過HTTP請求來進行的：

```bash
curl -X POST http://127.0.0.1:5984/_replicate -d \
  '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry", "create_target":true, "continuous":true}' \
  -H "Content-Type: application/json"
```

注意其中的name/value對：`"continuous":true`，就是說這個replication過程(同步)是持續的，而非一次性的，不需要像rsync鏡像那樣用cron job定期執行同步命令。

<div style="display: none;">
2013年8月17日更新，以下操作似乎不需要：

```bash
git clone git://github.com/isaacs/npmjs.org.git
cd npmjs.org
sudo npm install -g couchapp
npm install couchapp
npm install semver
couchapp push registry/app.js http://localhost:5984/registry
couchapp push www/app.js http://localhost:5984/registry
```
</div>

## 獲取同步狀態

使用以下命令：

```bash
curl -s localhost:5984/_active_task | jq .
```

顯示同步狀態。其中[`jq`](http://stedolan.github.io/jq/)是個強大的JSON數據的命令行處理器，類似於`sed`。在這個命令中`jq`的基本過濾器`.`，起到了pretty printer美化JSON輸出結果的作用。

題外話，在CSS的selector的影響下誕生了HTML快速生成的snippet工具Zen Coding(現在更名爲Emmet)，`jq`給人類似的感覺。

上面的命令有類似下面的JSON輸出結果：

```
[
  {
    "updated_on": 1376700136,
    "missing_revisions_found": 1678,
    "docs_written": 1678,
    "docs_read": 1678,
    "doc_write_failures": 0,
    "doc_id": null,
    "continuous": true,
    "checkpointed_source_seq": 620684,
    "pid": "<0.1872.0>",
    "progress": 99,
    "replication_id": "d2d78ebb34eb57384335196827cdc81e+continuous",
    "revisions_checked": 12615,
    "source": "http://isaacs.iriscouch.com/registry/",
    "source_seq": 624071,
    "started_on": 1376513978,
    "target": "registry",
    "type": "replication"
  }
]
```

CouchDB文檔不全，我沒有找到各個字段的含義，下面是個人臆斷：

`progress` name表示的是進度，最大爲100，是用`floor(checkpointed_source_seq * 100 / source_seq)`計算出來的。如果是`progress`達到了100就表明完全達到了官方數據庫的某一歷史版本狀態。如果沒到100，鏡像就處於不一致狀態，可能metadata信息和實際包不一致，但這種不一致性影響比較小，通常不會產生問題。

## 禁止普通用戶PUT/POST/DELETE

客戶端npm向服務端抓取數據只會用到GET請求，我們的鏡像是官方數據庫的一個slave database，不是權威服務器，也不具有官方服務器的用戶賬戶信息，所以無法提供用戶登錄，而且鏡像也不應該允許用戶上傳，所以PUT/POST/DELETE方法都用不到的。而CouchDB默認是允許任何用戶做修改的，這是個很大的安全風險，需要屏蔽掉這幾個方法。解決方案是給`registry`這個database的`_design/security`添加如下驗證函數`validate_doc_update`：

```javascript
function(newDoc, oldDoc, userCtx, secObj) {
  if (! userCtx || ~ userCtx.roles.indexOf('_admin'))
    log('Admin change on read-only db: ' + newDoc._id);
  else
    throw {'forbidden':'This database is read-only'};
}
```

可以執行下面的命令添加上面這段驗證函數：

```bash
curl -X PUT admin:password@localhost:5984/registry/_design/security -d \
  '{"validate_doc_update": "function(newDoc, oldDoc, userCtx, secObj) { if (! userCtx || ~ userCtx.roles.indexOf('\''_admin'\'')) log('\''Admin change on read-only db: '\'' + newDoc._id); else throw {'\''forbidden'\'':'\''This database is read-only'\''}; }"}'
```

## 使用Nginx做反向代理

CouchDB支持vhost，但爲了讓它和其他鏡像更好地協作，用Nginx做反向代理。

```nginx
server {
  listen [::]:80;
  server_name npm.*;
  root /mirror/npm/www;
  index index.html;

  location /registry {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_pass http://127.0.0.1:5984/registry/_design/app/_rewrite;
    # proxy_pass http://127.0.0.1:5984/registry;
  }

  location /registry/_design {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_pass http://127.0.0.1:5984/registry/_design;
  }
}
```

其中被`#`註釋掉的`proxy_pass http://127.0.0.1:5984/registry;`不可用，不明原因。

[fqj1994](http://fqj.me)同學報告並修復了一個問題：metadata返回的url的host是根據請求的`Host:`首部來決定的，所以需要`proxy_set_header Host $host;`來讓CouchDB生成正確的url。否則客戶端npm會收到包含127.0.0.1的metadata，試圖從自己機器獲取數據，自然就得不到。

另外，`leecade`同學提出了一個想法，希望本地源返回404

    % curl http://npm.tuna.tsinghua.edu.cn/registry/xxxxxxxx
    {"error":"not_found","reason":"document not found"}

時讓Nginx作爲前向代理替客戶端向官方源請求。當有一個包剛剛傳到官方源，本地源尚無相應信息時挺有用。可以在配置中修改`locatioon /registry`和`location /registry/_design`的配置：

```
   location /registry {
     proxy_redirect off;
     proxy_set_header Host $host;
     proxy_pass http://127.0.0.1:5984/registry/_design/app/_rewrite;
     # proxy_pass http://127.0.0.1:5984/registry;
     proxy_intercept_errors on;
     error_page 404 @official;
   }

   location /registry/_design {
     proxy_redirect off;
     proxy_set_header Host $host;
     proxy_pass http://127.0.0.1:5984/registry/_design;
     proxy_intercept_errors on;
     error_page 404 @official;
   }
```

並添加：

```
   location @official {
     proxy_redirect off;
     proxy_pass http://registry.npmjs.org;
   }
```

其中`proxy_intercept_errors`的作用就是讓Nginx解析後端返回的狀態碼>=400的錯誤，這樣`error_page`就能生效並引導至`@official`塊。

## 網頁介紹

然後要設計一個網頁介紹。

用`jade`來做模板引擎無疑是最方便的，不過標籤裏要添加尾部空格比較難以實現。Ruby社區的`slim`在`jade`的基礎上做了些改進，引入了一些便捷的東西，比`jade`更加好用。

CSS可以考慮用`stylus`結合`nib`插件。

## 從上游同步時使用代理

在向上遊同步時，偶爾會碰到流量被過濾，同步無法順利進行的情況。我碰到過幾次這樣的情況了，等幾天都同步都不會有進展的，用`ls -l`查看數據庫的修改日期會發現一直沒有變化。

這個時候需要臨時去除無代理的上游配置，換上有代理的上游配置：

```bash
curl -sX POST admin:password@127.0.0.1:5984/_replicate -d '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry", "continuous":true, "cancel":true}' -H "Content-Type: application/json"
curl -sX POST admin:password@127.0.0.1:5984/_replicate -d '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry", "continuous":true, "create_target":true, "proxy":"http://127.0.0.1:xxxx"}' -H "Content-Type: application/json"
```

第一句是去除原有的無代理上游配置，第二局是添加有代理的上游配置。

過了幾秒鐘後即可再次換上無代理的上游配置：

```bash
curl -sX POST admin:password@127.0.0.1:5984/_replicate -d '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry", "continuous":true, "cancel":true, "proxy":"http://127.0.0.1:xxxx"}' -H "Content-Type: application/json"
curl -sX POST admin:password@127.0.0.1:5984/_replicate -d '{"source":"http://isaacs.iriscouch.com/registry/", "target":"registry", "continuous":true, "create_target":true}' -H "Content-Type: application/json"
```

第一句是去除有代理上游配置，第二句是添加無代理的上游配置。

## 同步腳本

下面的同步腳本會跟蹤`registry.couchdb`文件的修改時間，以此判斷CouchDB的replication過程是不是卡住了，如果是則臨時掛上代理：

2013年10月16日更新，根據[http://wiki.apache.org/couchdb/Replication#Cancel_replication]()，CouchDB 1.2開始取消同步的方式有所變化。

```ruby
#!/usr/bin/env ruby
require 'net/http'
require 'net/smtp'
require 'json'

LOG_ROOT_DIR = File.expand_path '~mirror/log'
LOG_DIR = File.join LOG_ROOT_DIR, 'npm'
STATUS_FILE = File.join LOG_DIR, 'status.txt'
DB_PATH = '/srv/local/npm/couchdb/registry.couch'
USER = 'admin'
PASSWORD, HTTP_PROXY = JSON.parse(File.read '/srv/local/npm/couchdb/passwd').values_at 'password', 'http_proxy'
WAIT = 30
DEBUG = false

replicate = ->opts=({}) {
  req = Net::HTTP::Post.new '/_replicate'
  req.basic_auth USER, PASSWORD
  if opts[:cancel]
    form_data = {'replication_id' => opts[:replication_id], 'cancel' => true}
  else
    form_data = {'source'=>'http://isaacs.iriscouch.com/registry/',
      'target'=>'registry', 'continuous'=>true, 'create_target'=>true}
    form_data.update 'proxy'=>HTTP_PROXY if opts[:proxy]
  end
  req.set_content_type 'application/json'
  req.body = form_data.to_json
  res = Net::HTTP.new('localhost', 5984).request req
  puts res.body if DEBUG
}

cancel_all_replicates = ->sources {
  sources.each {|source|
    puts "replication: #{source['replication_id']}"
    replicate[cancel: true, replication_id: source['replication_id']]
  }
}

get = ->path {
  req = Net::HTTP::Get.new path
  req.basic_auth USER, PASSWORD
  Net::HTTP.new('localhost', 5984).request req
}

get_json = ->path {
  JSON.parse get[path].body
}

stuck = ->{
  Time.new - File.stat(DB_PATH).mtime > 60 * 10
}

write_log = ->log{
  puts "write log: #{log}" if DEBUG
  File.write STATUS_FILE, log
}

send_email = ->{
  name = 'Ray'
  from = 'issues.tuna@gmail.com'
  to = 'i@maskray.me'
  server = 'localhost'
  msg = <<E
From: #{name} <#{from}>
To: #{to}
Subject: npm ERROR on #{Time.now}

failed to sync
E

  Net::SMTP.start(server) do |smtp|
    smtp.send_message msg, from, to
  end
}

status = File.readlines(File.join log_dir, 'status.txt')[0].split ',' rescue 'failed,1377760143,-,0,74.5217GB,0,0,0,0,0,0,0,0'
sources = get_json['/_active_tasks']
last_fail = Time.now

loop do
  begin
    size = get_json['/registry']['disk_size']
    failed = ->{
      status[0] = 'failed'
      status[4] = size
      write_log["#{status.join(',')}\n"]
      if Time.now - last_fail > 60 * 60 * 24
        send_email[]
        last_fail = Time.now
      end
    }

    if sources.empty?
      replicate[]
    elsif stuck[]
      cancel_all_replicates[sources]
      replicate[proxy: true]
      sleep WAIT
      if stuck[]
        cancel_all_replicates[sources]
        failed[]
      else
        cancel_all_replicates[sources]
        replicate[]
      end
    else
      updated_on, _progress = sources[0].values_at 'updated_on', 'progress'
      write_log["success,#{updated_on},-,0,#{size},0,0,0,0,0,0,0,0"]
    end
  rescue => e
    $stderr.puts e
  ensure
    sleep 60
  end
end
```
