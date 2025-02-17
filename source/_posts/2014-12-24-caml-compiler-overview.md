layout: post
title: ML編譯器Caml Featherweight——概覽
author: MaskRay
tags: [compiler, ocaml, ml]
mathjax: true
---

## 索引

- [概览](http://maskray.me/blog/2014-12-24-caml-compiler-overview)
- [設計](http://maskray.me/blog/2014-12-24-caml-compiler-design)
- [抽象機器](http://maskray.me/blog/2014-12-24-caml-compiler-abstract-machine)
- [數據表示](http://maskray.me/blog/2014-12-24-caml-compiler-data-representation)
- [編譯、鏈接和運行](http://maskray.me/blog/2014-12-24-caml-compiler-compiling)
- [垃圾收集](http://maskray.me/blog/2014-12-24-caml-compiler-garbage-collection)
- [其他](http://maskray.me/blog/2014-12-24-caml-compiler-others)
- [Js_of_ocaml和Emscripten實現在線編譯運行](http://maskray.me/blog/2014-12-27-js-of-ocaml-and-emscripten)

## 代碼

編譯器的設計和實現大量參考了 @Zinc 和Caml Light 0.75，因此取名Caml Featherweight了……實現方面更接近於 @Zinc ，Caml Light有些晦澀複雜的地方我換成自己亂想的實現了。

tl;dr的話可以直接讀源碼：<https://github.com/MaskRay/CamlFeatherweight>

<!-- more -->

## 完成度

## 介紹

ML是個一門通用的函數式編程語言，有如下一些特點：

-   採用call-by-value的求值策略，允許副作用和imperative programming。
-   使用垃圾收集。
-   支持first-class function，函數可以像`int`、`string`那樣的值一樣自由傳遞、創建。
-   實現了靜態類型檢查，並能自動推導類型，無需C中的類型註解。
-   提供了immutable programming的良好支持。
-   參數多態，提供了一些通用編程支持，類似於Java中的generic。
-   支持代數數據類型和模式匹配以處理複雜的數據結構。

本項目是@Zinc 提到的Zinc實驗的一個實現，並大量參考了Caml Light 0.75的源碼，實現了ML方言Caml的一個變體，以上提到的特性均已使用，在某些細節語法方面有省略和改動。

## 項目採用的Caml語言變體

### 內置類型

基本類型：

-   `bool`，如`true`、`false`
-   `char`，如`’a’`、`’\n’`
-   `int`，如`12`、`0x0e`
-   `float`，如`3.0`、`2e-5`
-   `string`，如`hello`、`world`

語言內置了一些類型構造器。

-   `’a option`，如`None`、`Some (4,true,7,’a’)`

### 操作符

-   `=`，類型爲`’a -> ’a -> bool`
-   `<>`，類型爲`’a -> ’a -> bool`
-   `==`，類型爲`’a -> ’a -> bool`
-   `!=`，類型爲`’a -> ’a -> bool`
-   `<`，類型爲`’a -> ’a -> bool`
-   `<=`，類型爲`’a -> ’a -> bool`
-   `>`，類型爲`’a -> ’a -> bool`
-   `>=`，類型爲`’a -> ’a -> bool`

`int`類型支持的額外操作符：`+`,`-`,`*`,`/`,`mod`，分別表示加、減、乘、除、和模運算，類型均爲`int -> int -> int`。
`float`類型支持的額外操作符：`+.`,`-.`,`*.`,`/.`，即加減乘除，類型均爲`float -> float -> float`。

### 元組

元組是幾種類型的複合，可以看成幾種不同類型的有序集合。元組的各個成分用逗號分割，外面套圓括號，比如：

```ocaml
  # let a_tuple = (3,"three");;
  val a_tuple : int * string
```

某些地方可以省略圓括號。

可以用模式匹配抽取元組的各個成分：

```ocaml
  # let (x,y) = a_tuple;;
  val x : int
  val y : string
```

### 列表

元組可以把固定數目的不同元組聚合在一起，列表則可以表示相同類型的任意數目元素，語言提供的表示列表字面量的語法，如表示空列表的`[]`，帶有兩個元素的列表`[3; 4]`。

列表可以看成是一個單鏈表，單個節點可以爲空(`[]`)，或者包含一個元素和一個指向後繼節點(對應的構造器爲`::`，比如：`3::[]`)。`::`構造器是一個操作符，是右結合的，之前提到的`[3; 4]`可以看作`3::4::[]`的語法糖。

### 數組

數組類似於列表，可以表示相同類型的任意數目元素，並提供了隨機訪問的功能：

```ocaml
  let a = [|3; 4; 7|]
  let b = [|"a",('c',false); "b",('d',true)|]
```

訪問數組中的一個元素：

```ocaml
  a.(3)
```

修改數組中的一個元素：

```ocaml
  a.(3) <- 'a'
```

### `option`

`option`代表可能缺失的值，比如：

```ocaml
  let divide x y =
    if y = 0 then None
    else Some(x/y)
```

### 函數

接受一個參數，返回參數加一的函數如下：

```ocaml
  let plus_one x = x + 1
```

和類C語言不同，函數名和參數間用空格分割，並且函數體前使用`=`，因爲ML中沒有像C那樣區分語句和表達式。

如果要定義多參數的函數，參數間也用空格分割：

```ocaml
  let sum_of_three x y z = x + y + z
```

函數允許自遞歸，這時需要使用`let rec`：

```ocaml
  let rec fib n =
    if n < 2 then
      n
    else
      fib (n-1) + fib (n-2)
```

如果要定義互遞歸的函數，必須在一個`let rec`中同時定義各個函數，它們之間用`and`分割：

```ocaml
  let rec is_even x =
    if x = 0 then true else is_odd (x-1)
  and is_odd x =
    if x = 0 then false else is_even (x-1)
```

### 代數數據類型和模式匹配

```ocaml
  type t = Zero | One of int | Two of char * int
  type tt = A of t
```

上面定義了一個類型`t`，它有三個構造器：`Zero`、`One`和`Two`，接受不同的參數。

模式匹配和代數數據類型配套使用，可以抽取構造器的參數：

```ocaml
  match a with
  | A t ->
      match t with
      | Zero -> 0
      | One i -> i
      | Two(ch,i) -> i
```

模式匹配也可用於`int`、`string`等基本類型。`_`是通配符，可以匹配任意值：

```ocaml
  match s with
  | "hello" -> true
  | _ -> false
```

模式匹配也可用於複雜的類型，各分支中以以小寫字母開頭的表示符表示變量，可以匹配任意值，從而可以在`->`後的表達式中引用：

```ocaml
  match s, 'a', 3+2 with
  | "a", 'a', v -> "a"
  | g, h, 5 -> g
  | _ -> "b"
```

### 異常

異常提供了一種中斷當前的計算，報告錯誤，捕獲錯誤的機制。

除數爲零時會觸發`Division_by_zero`異常，下面的程序捕獲了該異常並輸出12：

```ocaml
try
  output_int (1/0)
with Division_by_zero ->
  output_int 12
```

如果模式匹配失敗，則會產生`Match_failure`異常：

```ocaml
output_int (try
  let Some 3 = None in
  1
with Match_failure _ ->
  0)
```

`try`實質上形成了一個動態作用域，如果最近的`try`無法捕獲的話，異常會交給外層的`try`：

```ocaml
output_int (try
  try
    let Some 3 = None in
    1
  with Division_by_zero ->
    0
with Match_failure _ ->
  2)
```

## 工具鏈使用說明

在項目目錄下執行`make`生成字節碼編譯器`camlfwc`、字節碼解釋器`camlfwrun`、字節碼查看器`camlfwod`。

編譯並鏈接源文件得到可執行的字節碼文件：

```
% ./camlfwc -v /tmp/a.ml -o /tmp/a.out
Compiling /tmp/a.ml
- : unit
Linking /tmp/a.zo -> /tmp/a.out
```

使用`camlfwrun`執行：

```
% ./camlfwrun /tmp/a.out
```

`camlfwod`可以顯示目標文件或字節碼可執行文件的指令列表。

```
% ./camlfwod /tmp/a.zo
%File /tmp/a.zo
Relocatable file

Offset 8
Length 33
0008: PUSHMARK
...

% ./camlfwod /tmp/a.out
File /tmp/a.out
Executable

Length 34
000c: PUSHMARK
%File /tmp/a.zo
Relocatable file

Offset 8
Length 33
0008: PUSHMARK
...
```

## 學位論文

[thesis.pdf](/static/2014-12-24-caml-compiler-overview/thesis.pdf)
