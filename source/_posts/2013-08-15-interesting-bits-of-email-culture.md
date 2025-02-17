---
layout: post
title: E-mail文化拾趣
author: MaskRay
tags: [hacker culture,email]
---

## 首部

### Carbon Copy (Cc:)

Carbon copying指的是用複寫紙複寫，電子郵件世界裏借用了這個術語表示把副本抄送給非主要收件人。

### Blind Carbon Copy (Bcc:)

[RFC5322](http://tools.ietf.org/html/rfc5322)提及郵件客戶端有三種處理Bcc:的方式：

- To、Cc、Bcc收件人均收到同樣的郵件，該郵件中Bcc:首部被移除
- To、Cc收件人收到Bcc:首部被移除的郵件。Bcc收件人收到帶有Bcc:首部的郵件，
  實現可以決定Bcc收件人是否能見到其他Bcc收件人。
- To、Cc收件人會看到空的Bcc:首部。

供參考，下面是RFC5322的3.6.3節對Bcc:的描述原文：

```
The "Bcc:" field (where the "Bcc" means "Blind Carbon Copy") contains
addresses of recipients of the message whose addresses are not to be
revealed to other recipients of the message.  There are three ways in
which the "Bcc:" field is used.  In the first case, when a message
containing a "Bcc:" field is prepared to be sent, the "Bcc:" line is
removed even though all of the recipients (including those specified
in the "Bcc:" field) are sent a copy of the message.  In the second
case, recipients specified in the "To:" and "Cc:" lines each are sent
a copy of the message with the "Bcc:" line removed as above, but the
recipients on the "Bcc:" line get a separate copy of the message
containing a "Bcc:" line.  (When there are multiple recipient
addresses in the "Bcc:" field, some implementations actually send a
separate copy of the message to each recipient with a "Bcc:"
containing only the address of that particular recipient.)  Finally,
since a "Bcc:" field may contain no addresses, a "Bcc:" field can be
sent without any addresses indicating to the recipients that blind
copies were sent to someone.  Which method to use with "Bcc:" fields
is implementation dependent, but refer to the "Security
Considerations" section of this document for a discussion of each.
```

在Gmail裏發封沒有To:、Cc:，只有Bcc:的郵件，收件人就會看到`To: undisclosed-recipients: ;`

### Signature Block

附在郵件正文後面作爲簽名信息，通常會包含發件人的職位、聯繫方式等。
爲了和正文分隔開，一般使用兩個連字符跟着一個空格和一個換行符(sig dashes)，用C語言的字面字符串表示方式就是`"  -\n"`。
郵件客戶端可以認出雙連字符記號用與正文不同的樣式標記出來或者隱藏。

### Top Posting vs Bottom Posting

Posting這個詞是用來描述新聞組的，

在回覆郵件時，表示回覆文本時在原文下面寫回覆(bottom posting)還是在原文上面先回覆(top posting)。
傳統的方式是採用bottom posting，
先發生的事(引文)出現在前面、後發生的(回覆)出現在後面在時間順序上較爲自然，
而top posting則會對理清事件的先後順序造成阻礙：

```
> Where are you?
At home.
```

他們抨擊top posting的理由就是後者顛倒了引文和回覆的時間順序：

```
At home.
> Where are you?
```

對閱讀郵件特別是很長的線索時造成障礙。在很多郵件列表，網絡禮儀就是使用bottom posting，

另外還有interleaved posting，把原文分爲多段，在每一段後寫下自己的回覆。
這種風格可以被用作point-by-point rebuttal，逐條駁斥對方的觀點。

現在top posting佔到了上風，我覺得很大一部分原因是網頁版郵件客戶端的興起和非黑客郵件使用者數目的激增。
郵件客戶端缺乏標註引文和切換引文顯示功能時，採取top posting的方式能減少鼠標滾動，更容易看到回覆的內容。

比如說Mutt默認就設置了快捷鍵跳到下一個不包含引文的行：

```
<skip-quoted> (default: S)

    This function will go to the next line of non-quoted text which comes after
    a line of quoted text in the internal pager.
```

以及用於切換引文是否顯示的：

```
<toggle-quoted> (default: T)

    The pager uses the $quote_regexp variable to detect quoted text when
    displaying the body of the message. This function toggles the display of
    the quoted material in the message. It is particularly useful when being
    interested in just the response and there is a large amount of quoted text
    in the way.
```

### Reply-To

這個郵件首部爲用戶回覆郵件時提供建議，設定新郵件的To:首部。

### Thread

#### References

#### In-Reply-To

### format=flowed

### Mail-Followup-To

回覆郵件時如果選擇了follow up(或者說reply to all)的方式，那麼收件人會被填爲Mail-Followup-To設定的地址。
避免發件人如果訂閱了該郵件列表的話收到兩封相同的郵件。

Mutt在你設置subscribe該郵件列表時會自動設置Mail-Followup-To首部。

## X-Mailer

一些不友好的客戶端：

### QQMail

不使用`References:`首部。

## Mutt

### query_command

```
set query_command="echo; grep %s ~/.mutt/aliases | cut -d' ' -f3-"
bind editor <tab> complete-query
```

## 其他

- 三種“From”：[http://stackoverflow.com/questions/1235534/what-is-the-behavior-difference-between-return-path-reply-to-and-from]()

### 参考配置

[http://www.spinnaker.de/mutt/muttrc]()

## Sender Policy Framework
