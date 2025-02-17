---
layout: post
title: 使用Double-array trie優化Proxy auto-config
author: MaskRay
tags: [data structure, coffee script]
---

## Proxy auto-config

瀏覽器的proxy auto-config讓我們可以根據URL和host決定使用什麼代理。

<!-- more -->

PAC文件是一個JavaScript腳本，其中最重要的就是函數`FindProxyForURL`，下面給出一個簡單的例子：

```javascript
function FindProxyForURL(url, host)
{
  return 'PROXY 0.1.2.3:4; SOCKS5 6.7.8.9:10; DIRECT';
}
```

這個函數接受兩個參數，分別爲URL和host，需要返回一個字符串，表示使用什麼代理。
`DIRECT`即爲不使用代理，直接鏈接；`PROXY`指定HTTP代理，IP地址和端口用冒號隔開；
`SOCKS5`指定SOCKS5代理。

### Chrome

Chrome可以帶上命令行選項`--proxy-pac-url`指定PAC文件，比如：

```bash
google-chrome --proxy-pac-url=file:///home/ray/.config/google-chrome/proxy.pac "$@"
```

在地址欄輸入`chrome://net-internals/#proxy`，如果PAC成功啓用了，`Effective settings`會這樣如下字樣：

```
PAC script: file:///home/ray/.config/google-chrome/proxy.pac
```

PAC文件修改之後，需要點擊`Re-apply settings`加載新的配置。

### Firefox

找到

```
Preferences - Advanced - Network - Settings - Automatic proxy configuration URL
```

填入`file:///home/ray/.config/google-chrome/proxy.pac`，點擊`Reload`。

## 提高host匹配性能

實際應用的`FindProxyForURL`應該維護了一份需要代理的host列表，如果其中某個host是URL中host的後綴，那麼就選擇使用代理。

如果host列表很長，一一匹配性能會很低，需要考慮使用一些算法優化。

對於字符串檢索問題，可以考慮使用trie這一數據結構。

## Double-array trie

樸素實現的trie對於每個節點，都需要維護大小爲`|SIGMA|`的children指針，其中`|SIGMA|`爲字符集大小（這裏就是合法的host字符集），空間複雜度較高。

Trie的一種改良double-array trie能大量節省空間，具體可以參考這篇文章：[An Implementation of Double-Array Trie](http://linux.thai.net/~thep/datrie/datrie.html)

### 實現

下面給出用於生成PAC文件的CoffeeScript程序：

```coffeescript
class DATrie
  constructor: (@n) ->
    @alphabet = [0].concat '-.'.split('').map((c) -> c.charCodeAt(0)), [48..57], [97..122]
    @mapping = {}
    @mapping[x] = i for x, i in @alphabet

    @base = Array(n)
    @check = Array(n)
    last = 0
    for i in [@alphabet.length+2...n]
      @check[last] = -i
      @base[i] = -last
      last = i
    @check[last] = -0
    @base[0] = -last
    @base[1] = 2

  chars: (s) ->
    c for c in [0...@alphabet.length] when @check[@base[s]+c] == s

  freeCell: (s) ->
    @check[s] = @check[0]
    @base[-@check[0]] = -s
    @base[s] = -0
    @check[0] = -s

  useCell: (s) ->
    if s >= @alphabet.length+2
      @base[-@check[s]] = @base[s]
      @check[-@base[s]] = @check[s]

  allocate: (chars) ->
    test = (b) =>
      for c in chars
        return false if @check[b+c] > 0 or @base[b+c] > 0
      true

    loop
      b = -@check[0]
      while b != 0
        if b+@alphabet.length < @n and test b
          return b
        b = -@check[b]
      nn = Math.ceil @n*1.3
      base2 = Array(nn)
      check2 = Array(nn)
      for i in [0...@n]
        base2[i] = @base[i]
        check2[i] = @check[i]
      @base = base2
      @check = check2
      for i in [nn-1..@n] by -1
        @freeCell i
      @n = nn

  relocate: (s, b) ->
    for c in @chars s
      @useCell b+c
      @check[b+c] = s
      @base[b+c] = @base[@base[s]+c]
      for cc in @chars(@base[s]+c)
        @check[@base[@base[s]+c]+cc] = b+c
      @freeCell @base[s]+c
    @base[s] = b

  ord: (ch) ->
    @mapping[ch.charCodeAt(0)]

  insertBranch: (s, c) ->
    if @base[s] > 0
      if @check[@base[s]+c] == s
        return
      if @check[@base[s]+c] > 0
        @relocate s, @allocate([c].concat @chars s)
    else
      t = @allocate [c]
      # @base may vary
      @base[s] = t
    @useCell @base[s]+c
    @check[@base[s]+c] = s

  insert: (str) ->
    s = 1
    for _, i in str
      o = @ord str[i]
      @insertBranch s, o
      s = @base[s]+o
    @insertBranch s, 0

  retrieve: (str) ->
    s = 1
    for c in str
      ss = @base[s]+@ord(c)
      return false if @check[ss] != s
      s = ss
    true

String::reverse = ->
  this.split('').reverse().join ''

hosts = [
  'github.com'
  'jquery.com'
]

trie = new DATrie(100)
for host in hosts
  trie.insert host.reverse()

FindProxyForURL = (url, host) ->
  proxy = 'PROXY 0.1.2.3:4; SOCKS5 6.7.8.9:10; DIRECT'
  if trie.retrieve host.reverse()
    proxy
  else
    'DIRECT'
```

使用`coffee -bc datrie-pac.coffee`編譯得到不含top-level function wrapper的PAC文件。

另外，Chrome的PAC不支持`Int32Array`，上面代碼就用`Array`代替了。
