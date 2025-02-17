---
layout: post
title: 基於 mutt+offlineimap+notmuch+inotifywait 的個人郵件系統
description: 個人使用的基於mutt+offlineimap的郵件系統，使用notmuch檢索郵件，inotify實現新郵件提醒
author: MaskRay
tags: [email, mutt, offlineimap, notmuch, inotify]
---

## 收郵件

早先使用[getmail](//pyropus.ca/software/getmail/) + [procmail](//www.procmail.org/)。缺點是本地對郵件的操作無法在服務器上反映出來。

使用[offlineimap](//offlineimap.org/)可以利用Gmail的filter功能，把郵件分揀到本地的各個maildir子目錄。

注意在Gmail上設置filter規則時，要選上Skip Inbox，以免一封郵件*同時出現在分類子目錄和INBOX中*。請看下面這條示例規則：

<!-- more -->

    Matches: list:"<tuna-general.googlegroups.com>"
    Do this: Skip Inbox, Apply label "Tuna"

## 新郵件提醒功能

我大致想到這項功能可以在以下三個地方處理。

- 在offlineimap配置文件中處理。草草看了下它的配置，`postsynhook`和`presynchook`不夠用，因爲無法知道郵件的標題和其他信息。
- `man notmuch-hooks`沒有滿足需要的hook。`postnewhook`也無法知道是否添加了新郵件。
- mutt中配置。Mutt配置太複雜，難以集成。
- 求助於`inotifywait`。

最終選擇了最後一個方案，編寫了這樣一個常駐的腳本。

```bash
##!/bin/bash
inotifywait -mrqe create ~/Maildir | while read; do
  if ! [[ "$reply" =~ notmuch ]] && notmuch new | grep -q ^Added; then
    msg=$(notmuch search --sort=newest-first --limit=1 --output=summary '*' | cut -d';' -f2-)
    notify-send -i ~/Icons/gmail.png -c mail -t 5 -- 'New mail' "$msg"
  fi
done
```

大意是遞歸監控`~/Maildir`目錄下文件的創建事件，如果創建的文件不是`notmuch`建立的，那麼就執行`notmuch new`把這個新郵件添加到`notmuch`數據庫中，調用`notify-send`提示最新的郵件。

## 在mutt中集成郵件搜索功能

使用[notmuch](//notmuchmail.org/)作爲郵件檢索系統。

在某個`$PATH`路徑下建立腳本，比如`~/bin/mutt-notmuch`：

```bash
##!/bin/bash
tmpdir=/tmp/.notmuch
rm -rf $tmpdir/{cur,new}
mkdir -p $tmpdir/{cur,new,tmp}
echo 'notmuch search terms:'
read
notmuch search --output=files "$REPLY" . |
  awk 'NR<=50;NR==50{system("notify-send -i ~/Icons/gmail.png Mail
  \"Too many results\"")}' | xargs -r ln -sft $tmpdir/cur
```

這個腳本的作用是讀入一行，用`notmuch search`檢索匹配的郵件，把它們的軟鏈接放到`/tmp/.notmuch`這個maildir中。

然後在`~/.mutt/muttrc`中添加

    macro index ,s \
      "<enter-command>unset wait_key<enter><shell-escape>mutt-notmuch --output \
      /tmp/.notmuch --prompt search<enter><change-folder-readonly>/tmp/.notmuch<enter>" \
      "search mail (using notmuch)"

這樣在Mutt中按`,s`會調用之前的腳本，並且檢查`/tmp/.notmuch`這個搜索結果形成的maildir。
