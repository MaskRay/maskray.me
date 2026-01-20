---
layout: post
title: Ouroborus Program - quine-relay 分析
author: MaskRay
tags: [ruby]
---

## Quine Relay

今天上網時看到有人分享了這個項目：[quine-relay](https://github.com/mame/quine-relay)，點進去看是這樣一個程序：一開始是一段Ruby代碼，運行後生成Scala代碼，後者運行後生成Scheme代碼，如此執行，依次生成了Bash、Smalltalk、Tcl等代碼，最後又回到最初的Ruby代碼。這類程序有個名字，叫**ouroborus programs**。Ouroborus即銜尾蛇，是一條正在吞食自己尾巴的蛇，在這裏剛好表示循環的概念。

![](http://upload.wikimedia.org/wikipedia/commons/thumb/7/71/Serpiente_alquimica.jpg/220px-Serpiente_alquimica.jpg)

<!-- more -->

## 分析

項目的有趣之處在於兩點：塑型，把代碼組成了特定圖案；一生二，二生三，三生一，形成循環往復的銜尾蛇效果。

### 塑型

先談塑型。

初始的Ruby程序位於項目根目錄下的`QR.rb`，代碼自身組織成銜尾蛇和六芒星的造型，這個看上去相當不容易。在一個所有詞法單元只佔據一個字符且忽略空白字符的語言(比如Brainfuck)中，把代碼組織成任意的圖案是很容易的。但是對於Ruby等詞法單元並非只佔據一個字符的語言，直接將代碼排版成特定圖案是很困難的。`quine-relay`採用了一個技巧，在字符串中編碼了一段程序用於生成下一種語言的代碼，調用`eval`解釋這個字符串。這個字符串表示是忽略空白字符的。文件`src/QR.rb.gen.rb`末段實現了這一功能：

```ruby
Tmpl = File.read("uroboros.txt")

code[-1, 0] = "#" * (Tmpl.count("#") - code.size)
code = Tmpl.gsub(/#+/) { code.slice!(0, $&.size) }
File.write("../QR.rb", code)
```

`Tmpl`是銜尾蛇圖案的模板文件，可以看作一個蒙版，指定了哪些地方填字符，哪些地方爲空格或換行，把最終代碼塑型成該圖案。

## 銜尾蛇效果

`quine-relay`想要實現的邏輯可以表達如下：

```
Ruby {\orig_ruby -> print (Scala {print (Scheme {print orig_ruby})})}
```

其中省去了其他語言的print。`Ruby {...}`代表一段Ruby代碼，`{}`中的部分是程序的內容。根據這段邏輯得到的Ruby代碼就是`orig_ruby`，並且`orig_ruby`被自身引用了。

下面分成兩部分來解釋實現方式：

- 實現不同語言的print
- 在代碼中引用自身代碼(`orig_ruby`)

### 實現不同語言的print

執行一下Ruby程序，再執行依次產生的各語言程序，可以發現它們做的事都是`print`某個特定字符串，而`print`的字符串自然就是下一個程序的源代碼了。各語言如何`print`一字符串，不少網頁都有收集hello-world的例子，比如：

- [http://www.roesler-ac.de/wolfram/hello.htm]()
- [http://en.wikipedia.org/wiki/List_of_Hello_world_program_examples]()
- [rosettacode.org/wiki/Hello_world/Text]()
- [c2.com/cgi/wiki?HelloWorldInManyProgrammingLanguages]()

`src/code-gen.rb`裏有大量編程語言輸出字符串的模板代碼，比如這段OCaml的：

```ocaml
class OCaml < CodeGen
  Ext = ".ml"
  Cmd = "ocaml QR.ml > OUTFILE"
  Apt = "ocaml"
  Code = %q("print_string"+E[PREV])
end
```

`PREV`之後會被替換爲OCaml生成的下一個編程語言的代碼，`E`則是用於字符串的引號轉義，並非看懂需要關注非常重要的點，暫時不用理會。

觀察`src/QR.rb.gen.rb`的頭幾行，可以發現作者在試圖構建單鏈表效果：

```ruby
s = '%(eval$s=%q(#$s))'
CodeGen::List[0..-2].each {|c| s = c.gen_code(s).tr(" \\","X`") }
```

`CodeGen::List[0..-2]`就是除去開頭結尾的Ruby以外其他各編程語言了，每一個都以需要產生的下一個語言的代碼爲參數，返回用本語言寫成的，返回那段字符串的代碼。

### 在代碼中引用自身代碼

把Ruby以外的其他編程語言的`CodeGen`子類都註釋掉，在`src`目錄下執行`rake ../QR.rb`，對`QR.rb`抽絲剝繭後可以觀察到這段核心代碼：

```ruby
eval$s=%q(eval(%w(
puts(eval(%q(%(eval$s=%q
(#$s))))))*""))
```

這段代碼是原始的Ruby寫成的quine，執行後的標準輸出依然爲這段代碼自身。理解它需要瞭解Ruby從Perl學習來的語法：general delimited input，即`%w`、`%q`這些符號，以及`Array`重載的`*`操作符的含義：

先看`$s`，它是`%q()`構建的字符串，`eval`它的效果是再次`eval`下面的字符串：

```ruby
%w(puts(eval(%q(%(eval$s=%q (#$s)))))) * ""
```

其中使用了`Array#*`的方法，即`Array#join`，因此效果是`eval`下面的字符串：

```ruby
puts(eval(%q(%(eval$s=%q(\#$s)))))
```

即輸出`eval$s=%q(`後，輸出`$s`的值，再跟`)`。這段Ruby quine的邏輯是：

```
eval$s=%q(
輸出`eval$s=%q(`
輸出$s
輸出`)`
```

字符串`$s`包含了一段需要`eval的代碼，代碼中包含了`$s`實現了自指，這可以看作是一種fixed-point combinator：Y combinator，的實現。

定義函數`f`，它應用參數`t`的效果是打印`t`的代碼。根據fixed-point theorem，存在代碼`n`使得`f(n)=n`，也就是說`n`會打印自身代碼。下面說明這段Ruby代碼本質上就是`n`的一種構造。

`$s`可以看作一個函數：`y = \x -> f (x x)`。`$s`中的自指以及`eval`的解釋操作，可以看作把自身作爲參數應用給自己：`y y`，`y`得到參數`x`後的效果是把`x x`的代碼打印出來。而打印`x x`需要打印以code形式存在的`x`和與之關聯的以data形式存在的`x`。考慮到參數傳遞給`y`的參數是一個固定值`x`即`y`，我們不需要寫一個通用的能打印任意代碼的程序，只需要針對這個特例，實現一個特例。在這裏data是`$s`，code是被`eval`解釋的`$s`，而所要打印的代碼則是`eval$s=%q(`、一段任意字符串和`)`。

`y y`即`(\x -> f (x x)) (\x -> f (x x))`，應用一次會打印自身代碼(副作用)並返回自身。爲了表述方便沒有嚴格考慮evaluation strategy的影響，這裏應該視爲使用了call-by-need，但是最外層的`y y`應用有一次強制的求值。

另外這段代碼的`puts`前可以執行任意多不影響標準輸出的代碼，`puts`的內容（括號裏面）也可以接一些代碼，比如最終`QR.rb`裏`puts`的內容就跟着``.gsub(/[HJK^`X]/)``等一長串代碼。`src/code-gen.rb`裏在`puts`前添加了`PROLOGUE`，用來規整化第一個Ruby程序看到的下一段代碼字符串。比如用`N`代替換行符，`B`代替反斜槓等，而`e`、`E`、`d`、`D`、`Q`等則是各語言裏字符串內特殊字符的轉義方式，在`CodeGen`各子類的hello-world模板代碼裏用到了這些轉義函數。

## 其他

另外值得稱道的是項目中的一款Whitespace解釋器：`src/whitespace.rb`，詞法符號的提取用了`\G`、named capture、named backreference等高級玩意兒：

```ruby
RE = /\G(?<n>[01]+){0}
  ( 00  (?<push> )\g<n>2
  | 010 (?<copy> )\g<n>2
  | 012 (?<slide>)\g<n>2
  | 020 (?<dup>)
  | 021 (?<swap>)
  | 022 (?<pop>)
  | 1000(?<add>)
  | 1001(?<sub>)
  | 1002(?<mul>)
  | 1010(?<div>)
  | 1011(?<mod>)
  | 110 (?<set>)
  | 111 (?<get>)
  | 1200(?<outc>)
  | 1201(?<outn>)
  | 1210(?<readc>)
  | 1211(?<readn>)
  | 200 (?<mark>)\g<n>2
  | 201 (?<call>)\g<n>2
  | 202 (?<jump>)\g<n>2
  | 210 (?<jz>)\g<n>2
  | 211 (?<jn>)\g<n>2
  | 212 (?<ret>)
  | 222 (?<end>)
  |     (?<eof>)\z
  |     (?<error>)
  )/x
```

`x`開啓free-spacing mode，正則表達式裏的空白符號會被引擎忽略。`|`分割的子表達式每個都表示一個詞法結構，用named capture來具體是哪個子表達式匹配成功了。題外話，關於正則表達式，《Mastering Regular Expressions》、《Regular Expressions Cookbook》都值得一讀。
