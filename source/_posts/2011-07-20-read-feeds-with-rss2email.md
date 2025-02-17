---
layout: post
title: 用rss2email閱讀feeds
author: MaskRay
tags: [email, feeds]
---

很久沒用rss的閱讀器了，以前曾用過 emacs 的 newsticker ，不支持HTML。也用過Google Reader，打開速度太慢，而且對Pentadactyl不友好。

## 把feeds轉成郵件來閱讀

我的想法是找一款工具，把feeds轉換成郵件，由本地的*procmail*處理（歸類），然後再用*mutt*閱讀。

<!-- more -->

### rss2email

rss2email就是這樣一個能把feeds轉成郵件的工具，Python寫的，可以通過SMTP把郵件投遞給服務器或者把郵件轉交給MTA。可惜的是它沒法像getmail或是fetchmail那樣，指定用於處理郵件的程序。

它的使用還是比較簡單的，配置只有兩歩：

- *r2e new user@example.org* ，轉換得到的 feeds 默認投遞到`user@example.org`
- *r2e add http://feed.url/somewhere.rss* ，訂閱一個源

以後只要運行*r2e run*就能把新的feeds轉成郵件發送給你配置時設置的郵箱。我把這條命令寫到了=crontab=裏。

### 原來的設想

我最初設想是安裝個MTA，比如postfix，*r2e*把郵件交給MTA，MTA把收到的郵件交給procmail。但這樣略顯麻煩，MTA的配置挺麻煩的，還得讓它開機自動啓動。

### 能否繞過 MTA 呢

我決定修改rss2email的源碼來避免 MTA 。rss2email的源碼是比較簡單的，在`/usr/lib/python2.7/site-packages/rss2email/main.py`中搜索`sendmail`就找到了這樣一行：

```python
p = subprocess.Popen(["/usr/sbin/sendmail", recipient], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
```

不難看出 recipient 就是指定的郵箱。比如 recipient 是 ray@localhost ，它會調用 */usr/sbin/sendmail ray@localhost* 來把郵件轉交給 *sendmail* (/MTA/)。

只要把這行改成：

```python
p = subprocess.Popen(["/usr/bin/procmail", "-d", recipient], stdin=subprocess.PIPE, stdout=subprocess.PIPE)
```

r2e就會直接把郵件交給*procmail*了。

### 具體配置

- 修改`/usr/lib/python2.7/site-packages/rss2email/main.py`
- *r2e new ray* ， ray 是我的用戶名
- *r2e add* http://feed.url1/somewhere.rss
- *r2e add* http://feed.url2/somewhere.rss
- *r2e add* http://feed.url3/somewhere.rss
- 配置 *procmail* ，根據郵件頭的 User-Agent 和 X-RSS-Feed 信息來決定投遞到哪個 maildir/mbox 。
- 把 *r2e run* 加入 crontab

下面是我添加到 ~/.procmailrc 中的內容：

    :0
    * ^User-Agent: rss2email
    {
      :0
      * ^X-RSS-Feed: http://news.ycombinator.com/rss
      rss/hacker-news/

      :0
      * ^X-RSS-Feed: http://solidot.org.feedsportal.com/c/33236/f/556826/index.rss
      rss/solidot/

      :0
      * ^X-RSS-Feed: http://feeds.feedburner.com/linuxtoy
      rss/linuxtoy/

      :0
      * ^X-RSS-Feed: http://jandan.net/feed
      rss/jandan/
    }

## 2014年11月26日更新

這個方案已廢棄，改用newsbeuter閱讀rss了。
