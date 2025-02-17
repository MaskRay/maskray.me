---
layout: post
title: 用Google Analytics API爲靜態站點實現“閱讀最多文章”功能
author: MaskRay
tags: [website, web analytics, node.js]
---

## Hexo

我一直走靜態博客路線。最早用`make`+`m4`搭建博客，後來改用Haskell的Hakyll，因爲網站相關的工具鍊(HTML、CSS、JS模板引擎等)不夠用去年11月又換成了Node.js社區的DocPad。最近升級DocPad壞掉了，於是8月上旬又折騰了下遷移到了Hexo。

<!-- more -->

[zippera](http://zipperary.com/)寫了好幾篇關於Hexo的文章，非常棒：

- [hexo系列教程：（一）hexo介紹](http://zipperary.com/2013/05/28/hexo-guide-1/)
- [hexo系列教程：（二）搭建hexo博客](http://zipperary.com/2013/05/28/hexo-guide-2/)
- [hexo系列教程：（三）hexo博客的配置、使用](http://zipperary.com/2013/05/29/hexo-guide-3/)
- [hexo系列教程：（四）hexo博客的優化技巧](http://zipperary.com/2013/05/30/hexo-guide-4/)
- [hexo系列教程：（五）hexo博客的優化技巧續](http://zipperary.com/2013/06/02/hexo-guide-5/)

博客站點的功能性特性我能想到下面幾個：

- 評論系統
- Web analytics比較容易實現
- 關聯文章
- 多語言，比如[https://www.byvoid.com/zhs/]()
- 列出閱讀最多的文章

評論系統使用[Disqus](http://disqus.com)，網站分析用Google Analytics。

下面介紹怎麼用Google Analytics實現“閱讀最多文章”功能。

## Google Analytics Tracking Code

如果你已經在自己的網站上加入了Google Analytics tracking code，請跳過這一段。

首先要給頁面加上tracking code，可以參考[Tracking Basics (Asynchronous Syntax)](https://developers.google.com/analytics/devguides/collection/gajs/)。其實用不着看文檔，Hexo的默認主題`light`已經自帶這個功能了，只需要修改`themes/light/_config.yml`中的這一行

```yaml
google_analytics: UA-35225578-1
```

把其中的`UA-35225578-1`換成你的網站的property ID。

## 用Google Analytics API獲取所有文章的pageview

先註冊一個Gmail帳號(這個方法需要登錄，爲了安全性起見不要用主要帳號)，在Google Analytics的Admin-User Management頁面授予這個帳號User權限。

另外我們需要獲取GA跟蹤的站點的profile ID，這個比較難找。在Google Analytics查看自己的網站會看到這樣的URL：

```
https://www.google.com/analytics/web/?hl=en&pli=1#croverview/cr-overview/aXXXXXXXXwYYYYYYYYpZZZZZZZZ/
```

其中`ZZZZZZZZ`就是profile ID。

下面[LiveScript](http://livescript.net/)腳本根據Gmail帳號用戶名、密碼認證，然後根據profile ID返回所有文章的`ga:pageviews`，輸出爲JSON格式。

腳本的工作原理是先用[ClientLogin for Installed Applications](https://developers.google.com/accounts/docs/AuthForInstalledApps)登錄剛纔新註冊的Gmail帳號，得到Google返回的字符串`AUTH=xxxxxxxxxxxxxx`，然後根據這個`AUTH`向`http://www.googleapis.com/analytics/v3/data/ga`發送GET請求。

```
% ls
node_modules/  out/  public/  scaffolds/  scripts/  source/  themes/  util/  db.json  package.json  twistd.log  _config.yml
% npm install -g LiveScript
% lsc util/generate-popular-json.ls
```

```livescript
# util/generate-popular-json.ls
require! 'https'
require! 'querystring'

pp = (o) ->
  console.log JSON.stringify o, null, 2

class GA extends require('events').EventEmitter
  (@user, @password) ->

  receive: (res, cb) ->
    chunks = []
    len = 0
    res.on 'data', (chunk) ->
      chunks.push chunk
      len += chunk.length
    res.on 'end', ->
      buf = new Buffer len
      offset = 0
      for c in chunks
        c.copy buf, offset, 0
        offset += c.length
      cb buf.toString!

  login: (cb) ->
    options =
      host: 'www.google.com'
      port: 443
      method: 'POST'
      path: '/accounts/ClientLogin'
      headers: 'Content-Type': 'application/x-www-form-urlencoded'

    post-data =
      Email: @user
      Passwd: @password
      accountType: 'HOSTED_OR_GOOGLE'
      source: 'curl-accountFeed-v2'
      service: 'analytics'

    req = https.request options, (res) ~>
      @receive res, (data) ~>
        if m = data.match /(Auth=[^\s*]*)\s/
          @token = m[1]
          cb null, @token
        else
          cb data
    req
      ..write querystring.stringify post-data
      ..end!

  get: (request, cb) ->
    if @debug
      console.log 'token:', @token
    options =
      method: 'GET'
      host: 'www.googleapis.com'
      port: 443
      path: "/analytics/v3/data/ga?#{querystring.stringify request}"
      headers:
        Authorization: "GoogleLogin #{@token}"
        'GData-Version': 2
    req = https.request options, (res) ~>
      @receive res, (raw-data) ->
        data = JSON.parse raw-data
        if data.error?
          cb data.error.message
        else
          cb null, data
    req.end!

ga = new GA 'email address', 'password'
profile-id = 64586883

(err, _token) <- ga.login!
console.error(err)+process.exit(1) if err?

(err, results) <- ga.get {
  dimensions: 'ga:pagePath,ga:pageTitle'
  ids: "ga:#{profile-id}"
  'start-date': '2006-01-01'
  'end-date': '2026-01-01'
  metrics: 'ga:pageviews'
  sort: '-ga:pageviews'
}
console.error(err)+process.exit(2) if err?

pp results.rows
```

因爲是靜態站點，我們需要每隔一段時間更新pageviews，把信息同步到網站服務器上。我使用`fcron`(功能比`vixie-cron`、`cronie`等多很多)：

```
% fcrontab -l
PATH=/usr/bin:/bin:/home/ray/bin:/home/ray/.local/bin
0 */6 * * *      cd ~/maskray.me; lsc util/generate-popular-json.ls > out/api/popular.json
```

即每個六小時抓取文章的`ga:pageviews`信息，生成`http://maskray.me/api/popular.json`使用的靜態文件。

## 爲Hexo添加一個widget顯示“閱讀最多文章”

新建文件`themes/light/layout/_widget/popular.ejs`：

```html
<div class="widget popular">
  <h3 class="title">Popular</h3>
  <ul class="entry" id="js-popular">
  </ul>
</div>
```

修改`themes/light/_config.yml`中的`widgets:`，添加`- popular`。

我創建了目錄`out`表示靜態站點的生成目錄，其中：

```
% readlink out/blog
../public
% ls out/js/popular.js
out/js/popular.js
```

把這段LiveScript腳本編譯成`out/js/popular.js`：

```livescript
truncate = (s) ->
  if s.length > 24
    "#{s.slice(0, 21)}..."
  else
    s

$.get '/api/popular.json', (data) ->
  data = JSON.parse data if typeof data is 'string'
  for row in data
    $('#js-popular').append $('<li>').append $('<a>').attr('href', row.path).text("#{truncate row.title} (#{row.pageView})")
```

在`themes/light/layout/_partial/after_footer.ejs`末尾添加：

```html
<script src="/js/popular.js"></script>
```

現在只有一個assets文件：`/js/popular.ls`，所以每次修改後就手動執行`lsc -c out/js/`生成。以後東西多了就該切換到[Grunt](http://gruntjs.com/)了。

## 2017年6月3日更新：OAuth 2代替ClientLogin

2015年某時起，使用Google賬戶密碼認證的ClientLogin已不再支持，現在得用Google API Client。因爲某些文章的路徑名發生過變化，我還用了一個髒辦法去掉這些失效的鏈接。2017年之前的API改變了，再次更新。下面基本上是我現在用的生成腳本，其中的ID信息都隱藏了：

```python
#!/usr/bin/env python3
# pip install --user pyopenssl google-api-python-client
import httplib2

from googleapiclient.discovery import build
from googleapiclient.http import HttpError
from oauth2client.service_account import ServiceAccountCredentials
import re, sys, json, os

VIEW_ID = 'ga:XXXXXXXX'

def get_metrics():
    credentials = ServiceAccountCredentials.from_json_keyfile_name(
            os.path.join(os.path.dirname(os.path.abspath(__file__)), 'API Project-XXXXXXXXXXXX.json'),
            scopes=['https://www.googleapis.com/auth/analytics.readonly'])
    http = credentials.authorize(httplib2.Http())
    analytics = build('analytics', 'v4', http=http, discoveryServiceUrl='https://analyticsreporting.googleapis.com/$discovery/rest')
    result = analytics.reports().batchGet(body={'reportRequests': [{
        'viewId': VIEW_ID,
        'dateRanges': [{'startDate': '2012-01-01', 'endDate': 'today'}],
        'metrics': [{'expression': 'ga:pageviews'}],
        'dimensions': [{'name':'ga:pagePath'}, {'name':'ga:pageTitle'}],
    }]}).execute()['reports'][0]['data']['rows']

    #with open('/tmp/g.json') as f:
    #    result = json.load(f)['reports'][0]['data']['rows']
    return result

def normalize_path(path):
    return re.sub('[.?].*|\\/$', '', path)

def normalize_title(title):
    return re.sub('MaskRay [|]|[|] MaskRay', '', title).strip()

r = [(normalize_path(x['dimensions'][0]), normalize_title(x['dimensions'][1]), int(x['metrics'][0]['values'][0])) for x in get_metrics()]
r = sorted([x for x in r if re.match('/blog/20..-..-..-', x[0])], key=lambda x: x[0])
i = 0
rr = []
while i < len(r):
    j = i
    pv = 0
    opt_pv = i
    while j < len(r) and r[i][0] == r[j][0]:
        pv += r[j][2]
        if r[j][2] > r[opt_pv][2] and r[j][1] != '(not set)':
            opt_pv = j
        j += 1
    if not re.search('-document-viewer|build-system-tup|2013-07-28-beauty-of-programming|2012-11-12-build-website-with-docpad|2012-11-12-migrate-to-docpad|asc14-to-isc15-of-my', r[i][0]):
    #if not re.search('-document-viewer|build-system-tup|/-document-viewer|build-system-tup|2012-11-19-ai9|2012-11-12-build-website-with-docpad|2013-07-28-beauty-of-programming|2012-11-12-migrate-to-docpad|2015-05-01-jq-internals-bytecode|2015-03-26-leetcode-best-time-to-buy-and-sell-stock-iv|2015-03-25-elf-hacks|2015-03-22-bctf-2015-camlmaze|2015-03-13-debug-hacks-2|2012-11-01-perfect-maze-generation|2014-11-23-jsxajs-workgroup|2015-06-15-morris-post-order-traversal|2014-10-13-wechat-export|2012-09-09-parallel-n-body|2013-03-13-xv-olimpiada-informatyczna-etap-1-klo|2015-06-29-bmc-firmware-reverse-enginnering|2014-12-30-summary', r[i][0]):
        rr.append({'path':r[i][0], 'title':r[opt_pv][1], 'pageView':pv})
    i = j
rr.sort(key=lambda x: - x['pageView'])

# remove entries with wrong date
rrr = []
exist = set()
for x in rr:
    m = re.match('/blog/20..-..-..-([-\\w]*)', x['path'])
    if not m:
        rrr.append(x)
    elif m.groups()[0] not in exist:
        exist.add(m.groups()[0])
        rrr.append(x)
json.dump(rrr, sys.stdout, ensure_ascii=False)
```

使用`ensure_ascii`使用中文而不是`\u1234`這樣的字串是爲了用OpenCC轉換成統一的簡體和繁體比較方便，參見[Nginx根據Accept-Language的简繁体支持](/blog/2015-10-06-nginx-accept-language-zhs-zht)。
