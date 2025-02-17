---
layout: post
title: Nginx根據Accept-Language的簡繁體支持
author: MaskRay
tags: [website,nginx]
---

這個功能開啓很久了，但直到昨天才發現遺漏了`atom.xml`……

我想根據HTTP首部的`Accept-Language`決定提供簡體或繁體的文件。在Chrome中，`chrome://settings/languages`可以設定偏好語言，瀏覽器會據此設置`Accept-Language`首部。較好的處理方式是解析該字段，獲取qvalue，根據優先級選取最恰當的語言。但僅用於支持簡繁體，我想用取巧的辦法：忽略優先級，只要`Accept-Language`裏出現了`zh-Hant`、`zh-TW`、`zh-HK`等字樣，就返回繁體，否則返回簡體。

<!-- more -->

在Nginx配置文件中與`server`塊同級的地方加上：

```nginx
map $http_accept_language $lang {
  default zhs;
  ~zh-Hant zht;
  ~zh-TW zht;
  ~zh-HK zht;
}
```

我用Hexo生成網站，源文件用繁體寫成。對於`hexo generate`生成得到的`2015-10-06-nginx-accept-language-zhs-zht.html`，用[OpenCC](https://github.com/BYVoid/OpenCC)轉換得到簡體版本：`2015-10-06-nginx-accept-language-zhs-zht.html.zhs.html`。視情況還需要轉換其他一些文件，比如`atom.xml`、[提供“閱讀最多文章”功能](/blog/2013-08-19-popular-posts-with-google-analytics-api)的`popular.json`。

```zsh
# zsh
cd ~/maskray.me/public
opencc -c t2s.json -i atom.xml -o atom.xml.zhs.xml
for i in **/*.html 20*; do # 選擇需要簡繁體支持的文件
  c=${#${(s/.html/%)i}//[^%]/}   # 計算子串`.html`出現次數
  if (( $c <= 1 )); then         # 出現一次的爲原始文件，需要轉換成簡體
    opencc -c t2s.json -i $i -o $i.zhs.html
  fi
done
```

在Nginx配置文件中指定需要簡繁體支持的路由，

```
server {
  # ......

  location ~ ^/blog/20?? {
    try_files $uri.$lang.html $uri =404;
    add_header Vary Accept-Language;
  }

  location ~ /atom.xml {
    try_files $uri.$lang.xml $uri =404;
    add_header Vary Accept-Language;
  }

  location ~ \.json$ {
    try_files $uri.$lang.json $uri =404;
    add_header Vary Accept-Language;
  }

  # ......
}
```
