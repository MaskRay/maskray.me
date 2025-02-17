---
layout: post
title: 用DocPad構建靜態網站
author: MaskRay
tags: [website, node.js]
---

## makefile + m4

最早用 makefile 管理站點，其中緣由可以參看[用Makefile搭建博客](/blog/2011-07-12-blogging-with-makefile)，在不懂*css*的情況下硬生生用*m4*根據一個 WordPress 主題構建起了自己的站點。

評論系統採用第三方的 [Disqus](//disqus.com)。

<!-- more -->

## Hakyll

幾個月前因爲覺得makefile + m4的方案不夠靈活，打算換用現成的靜態站點生成工具。出於對Haskell的着迷，我不由自主地選擇了[Hakyll](//jaspervdj.be/hakyll/)，這次遷移站點內容沒有發生任何變化，只是構建方式換成了 Hakyll。

## DocPad

而今我正初窺網頁開發門徑，覺得Hakyll方案有不少缺點，其一是Haskell社區沒有好用的HTML、CSS模板語言，而這一方面的佼佼者無疑是Node.js社區的一些方案如：[Jade](//jade-lang.com)、[Stylus](//learnboost.github.com/stylus/)。量外 Hakyll 文檔極度匱乏，而且它重度依賴的arrow(比monad更一般化)這一抽象概念我理解不夠深刻，使用起來捉襟見肘。

之前還猶豫過是否應該選擇Ruby的解決方案，智能體開發中我接觸了[Slim](//slim-lang.com)，一些地方比Jade要靈活(標籤屬性不需要括號)。但在調研Jekyll和Octopress後發現結合Slim、Stylus困難不少，幾經探索最終瞭解到Node.js社區的[DocPad](https://github.com/bevry/docpad)。在[MDN](developer.mozilla.org)淺嘗了幾篇介紹HTML、CSS的文章，閱讀了朋友借我的幾本Web設計書籍，打算start from scratch來實踐所學知識。

### Google Analytics

分析用。**目前還不知道如何用它來創建一個頁面widget顯示訪問最多的文章。**

### Disqus

評論系統。這次遷移把路徑從`/posts`改成`/blog`了，通過上傳一個url變化的csv文件讓Disqus遷移。

### Syntax highlighting

[highlight.js](//softwaremaniacs.org/soft/highlight/en/)

目前highlight.js還不支持C的語法高亮(儘管支持C++)。

### 相關文章

使用了[docpad-plugin-related](https://github.com/Delapouite/docpad-plugin-related)這個插件。

我在文章中一般寫`tags: haskell, algorithm`而不是`tags: [haskell, algorithm]`，這樣它會根據tags字符串做完全匹配，而不是根據兩篇文章tags有交集。所以我修改了下代碼

```coffeescript
if typeof tags is 'string'
  tags = tags.split(',').map (s) -> s.trim()
```

讓docpad-plugin-related把tags這個String split成數組。

### 上/下一篇文章

自己寫了個小插件：

```coffeescript
module.exports = (BasePlugin) ->
  class RelatedPlugin extends BasePlugin
    name: 'around'

    renderBefore: (opts,next) ->
      documents = @docpad.getCollection 'posts'

      prev = undefined
      documents.forEach (doc) ->
        if prev
          doc.set prev: prev
          prev.set next: doc
        prev = doc

      return next()
```

文章的layout裏加上類似這樣的代碼：

```jade
  if document.prev
    span.prev
      | Prev&nbsp;
      a(rel='prev', href=document.prev.get('url'))= document.prev.get('title')
```

### 分享按鈕

- [Google +1 Button](https://developers.google.com/+/plugins/share/)
- [Twitter](https://twitter.com/about/resources/buttons)
- [新浪微博](http://open.weibo.com/sharebutton)

### Atom

寫了個Ruby腳本產生[Atom](/atom.xml)feed，不知道Node.js有什麼解決方案：

```ruby
#!/usr/bin/env ruby
require 'hpricot'
require 'rss/maker'

content = RSS::Maker.make('atom') do |m|
  m.channel.title = 'MaskRay'
  m.channel.link = 'http://maskray.me/atom.xml'
  m.channel.description = 'MaskRay'
  m.channel.author = 'MaskRay'
  m.channel.updated = Time.now
  m.channel.id = m.channel.updated.to_s
  m.items.do_sort = true

  Dir['out/blog/*.html'].sort.reverse[0..5].each do |f|
    doc = Hpricot open(f).read
    i = m.items.new_item
    i.title = (doc/'title').inner_text.sub 'MaskRay | ', ''
    i.link = "http://maskray.me/#{f[4..-1]}"
    i.date = Time.parse((doc/'time')[0].inner_text)
    i.description = (doc/'#post-content').inner_text
  end
end

File.open('out/atom.xml', 'w') do |f|
  f.write(content)
end
```

### 去掉url中的`.html`後綴

Apache的`.htaccess`文件裏加上：

    RewriteEngine on
    RewriteBase /

    RewriteRule ^posts/(.*)(\.html)?$ /blog/$1 [L,R=301]

我的另一些奇怪要求，把瀏覽器的帶`.html`後綴的url重定向到不帶`.html`後綴的url。這個任務比想象中難很多，原因在於我不知道如何處理瀏覽器和本地解析地址不同的問題。這個問題得到了[fqj1994](http://fqj.me/)同學的大力幫助，也耽誤了他好多時間。

    RewriteCond %{REQUEST_FILENAME}/ -d
    RewriteCond %{REQUEST_FILENAME}.html !-f
    RewriteRule [^/] - [L]

    RewriteCond %{ENV:REDIRECT_STATUS} ^$
    RewriteRule ^(.*)\.html$ /$1 [R=301,L]

    RewriteCond %{REQUEST_FILENAME}.html -f
    RewriteRule [^/]$ %{REQUEST_URI}.html [QSA,L]
