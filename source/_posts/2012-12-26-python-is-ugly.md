---
layout: post
title: Python is ugly
description: Python很多地方設計的很醜陋，比如false values，浪費single quotes，one-expression lambda等
author: MaskRay
tags: python
---

## False values

這些都是false value：`None`, `False`, `0`, `[]`, `{}`, `''`。判斷empty的應用場景可能多些，如果像Ruby那樣只讓`nil`和`false`爲false就好了。

<!-- more -->

## Trailing colons

```python
if True:
    print 'hello'
```

寫成這樣如何？

```
if True
    print 'hello'
```

## Single/double quotes

Python不區分單/雙引號，失去了string interpolation的好處，另外就是需要儘可能少escape字符時也帶來了麻煩，當然可以用triple-quoted string和raw string來代替，但畢竟繁瑣了些。Raw string的設計還能接受，但爲什麼不引入這個：

```ruby
%w(zero one two three)
=> ["one", "two", "three"]
```

## dunder

- `__len__`: 爲什麼使用`len(obj)`而不是`def len():`和`obj.len()`呢？
- `__mul__`: `def *(self, rhs):`會更清晰些吧

## 類名

隨處可見的lowercase的標準庫類名和函數名。

- CamelCase：`collections.OrderedDict`。
- nocase：`collections.defaultdict`, `collections.namedtuple`

## map/filter/zip

- 產生序列的：`map|filter|zip`
- 產生iterator的：`itertools.i(map|filter|zip)`

爲什麼對於`list`和`iterator`要有兩套功能相近的函數呢？如果把`map`設計成這樣：參數爲`list`就返回`list`，參數爲`iterator`就返回`iterator`。這樣就不需要另成一套的`itertools.i(map|filter|zip)`了。

## Context manager和with statement

`with`可能借鑑自Lisp的`with-*` macro，實現的功能是wrap一段代碼，在其執行前後執行自定義的操作。功能和這個Haskell函數類似：

```haskell
bracket :: IO a -> (a -> IO b) -> (a -> IO c) -> IO c
  -- Defined in `Control.Exception.Base
```

我一直覺得這個feature是不值得成爲關鍵字的。如果改造一下變成Smalltalk/Ruby的block那種形式會好很多。比如如下Ruby風格的代碼：

```python
def pre():
  print('hello')

def post():
  print('world')

with pre, post do
  print(', ')
```

再注意`__exit__`的類型簽名：

```python
def __exit__(self, type, value, traceback):
```

後三個參數都是用來表示異常的，異常是不是special enough以至於可以break the rules了？

## One-expression lambda

因爲Python強制的off-side rule，超過一個表達式的語法確實很難設計。我想到的是Ruby風格的block。

## List comprehensions

體現語言表達能力的結構，不過順序和習慣有些差異：先寫出操作，後寫出執行操作的對象。Smalltalk/Ruby風格的cascading加上block可能更自然些：

```ruby
(1..100).select {|x| x % 3 == 0 }.reverse
```

```python
list(reversed([x for x in range(1,101) if x % 3 == 0]))
```

如果Python能採用Haskell的簡約寫法也不錯：

```haskell
reverse [x | x <- [1..100], x `mod` 3 == 0]
```

## yield

Python的`yield`用於實現`iterator`，給callee提供值。如果像Ruby那樣，`yield`的作用是調用一個callback會更加靈活一些。

幸好有PEP 342把`yield`改成雙向的了，實現了coroutine，可以[實現一些比較強大(花哨)的結構如Monad](http://www.valuedlessons.com/2008/01/monads-in-python-with-nice-syntax.html)。

## 後記

Python設計很醜陋，但是還得用。scapy、scrapy、scipy、matplotlib、pil都是以後打算瞭解的東西，而它們都沒有成熟的同等功能的Ruby替代品。
