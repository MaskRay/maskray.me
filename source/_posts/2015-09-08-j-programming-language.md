---
layout: post
title: J語言初探
author: MaskRay
tags: [j]
mathjax: true
---

先看些例子，感受一下J的魅力：

```
   +/\ i.6                     NB. prefix sum of 0~5
0 1 3 6 10 15
   (+/ % #) 2 3 4 5            NB. mean
3.5
   */~ 1+i.9                   NB. multiplication table
1  2  3  4  5  6  7  8  9
2  4  6  8 10 12 14 16 18
3  6  9 12 15 18 21 24 27
4  8 12 16 20 24 28 32 36
5 10 15 20 25 30 35 40 45
6 12 18 24 30 36 42 48 54
7 14 21 28 35 42 49 56 63
8 16 24 32 40 48 56 64 72
9 18 27 36 45 54 63 72 81
   ({.;#)/.~ 'abbccc'
┌─┬─┐
│a│1│
├─┼─┤
│b│2│
├─┼─┤
│c│3│
└─┴─┘
```

<!-- more -->

最早瞭解J是受了DarkRaven的教唆，在Project Euler的論壇裏能看到很多晦澀但又簡潔的J的實現，個個都是code golf的風格，力圖節省每一個字節，風格全然不像常規編程語言，J與C的差異感覺尤勝Haskell與C的差異，但沒有學。幾年後又瞭解到curimit也在看，之後就認真研究了一下。

## Noun

J中的數據稱爲noun，比如`3`、`'string'`等。相鄰的一串數如`3 4 2 5`視爲一個整體，表示一個array：
```
   4 5 6
4 5 6
   'aa'
aa
```

## Copula

賦值操作符`=.`和`=:`被稱爲copula。`=.`爲local assignment，有點類似局部變量定義；`=:`爲global assignment。
```
   a =: 4 5 6
   b =: 'aa'
```

賦值的內容可以是noun、verb、adverb、conjunction等。
```
  a_verb =: verb def '- y'      NB. 定義一個monadic verb
  an_adverb =: \
  a_conjunction =: &
```

## Verb

Verb(用`u`、`v`表示)是一種以noun爲參數，返回noun的函數，分爲monadic(單元)和dyadic(二元)兩種。Monadic verb在右邊取一個參數(`y`)，Dyadic verb在左右兩邊各取一個參數(`x`和`y`)。
```
   % 4         NB. monadic % (Reciprocal)
0.25
   - 3         NB. monadic - (Negate)
_3
   2 - _3      NB. dyadic - (Minus)
5
```

J中大多數操作符都具有monadic和dyadic兩種形式。

多個noun和verb組成的表達式，從右到左計算，沒有二元操作符優先級的規定，若要改變運算順序可以使用括號。若形成noun verb noun的形式，則應用dyadic verb而非monadic：
```
   3 * 1 + 2
9
   (3 * 1) + 2
5
   % 2 + 2
0.25
   % % 2 + 2
4
```

Monadic `i.`爲Integers，用於生成整數序列：
```
   i. 3        NB. 1維array，shape爲1
0 1 2
   i. 2 3      NB. 2維array，shape爲2 3
0 1 2
3 4 5
   i. 2 3 4    NB. 3維array，shape爲2 3 4
 0  1  2  3
 4  5  6  7
 8  9 10 11

12 13 14 15
16 17 18 19
20 21 22 23
```

Monadic `#`爲Tally，計算array的第0維長度即shape的首元素：
```
   # 0 1 2
3
   # i. 2 3 4
2
```

Dyadic `,`爲Append，拼接兩個array。一串數會被解析爲一個array，但變量和數不會被解析爲一個array，若要拼接需使用`,`：
```
   x =: 3 4
   2 , x
2 3 4
```

## Adverb & conjunction

Adverb是單元修飾符，跟在noun或verb後面形成新的noun、verb、adverb、conjunction。

最簡單的用法是修飾verb得到新verb，可以看作是FP中的higher-order function。
```
   + / 1 2 3 4     NB. / (Insert)，類似FP中的fold/reduce，作用爲計算 1+(2+(3+4))
10
   + / \ 1 2 3 4   NB. \ (Prefix)，類似FP中的scan，多個adverb採用左結合，即`(+/)\`，作用爲計算 (+/ 1),(+/ 1 2),(+/ 1 2 3)
1 3 6 10
   ,~ 1 2          NB. ~ (Reflex)，作用爲計算 1 2 , 1 2
1 2 1 2
```

Conjunction爲雙元修飾符，作用在左右兩個noun或verb上形成新的noun、verb、adverb或conjunction。

```
   (5 & -) 4       NB. & (Bond)，規則爲`m&v y => m v y`、`u&n y => y u n`，即固定某個dyadic verb一邊的參數，類似Haskell中infix operator的section
1
   (- & 5) 4
_1
   %@:% 4          NB. @: (At)，規則爲`u@:v y => u (v y)`，類似FP中的函數結合。
4
   ((10&+)^:3) 1   NB. ^: (Power of Verb)。若`n`爲非負整數，表示在`y`上應用`n`次`u`；若`n`爲-1(J中用`_1`表示)，表示`u`的逆運算。很多primitive都設置了逆運算，參看<http://www.jsoftware.com/jwiki/Vocabulary/Inverses>
31
```

Conjunction `"`，根據左右參數的詞性有三種語義，最常用的形式爲`u"n y`，左邊爲verb，右邊爲整數$n$，用於在參數`y`的所有$n$-cells上應用`u`。後文會提到，是array-oriented programming的重要組成部分。

若只給conjunction提供一邊的參數，則會得到adverb：
```
   f =: - @:       NB. u f = - @: u
   g =: @: -       NB. u f = u @: -
```

Noun爲值，verb作用在noun上產生新的noun，這兩個詞性和常規編程語言中表達式的構成尚無太大區別。但adverb和conjunction的結果可能是多種詞性，從而無法靜態確定局部表達式運算結果的詞性，無法表達爲context-free grammar，也無法用abstract syntax tree表示。

Adverb & conjunction可以自定義。`u`、`v`分別爲左右參數：
```
   f =: 1 : 'u'                NB. 1代表adverb。定義了一個原樣返回的adverb
   +/ f 1 2 3
6
   where =: 2 : 'u'            NB. 2代表conjunction，返回左側參數。根據求值順序，右邊參數已經被求值了
   3 plus 4   where plus =: +  NB. `plus =: +`已經執行過了，返回左側參數。模擬Haskell的where
7
   (z+1)*(z+2)   where z =: 3
20
```

## 求值方式

J的表達式的計算方式比較奇特(<http://www.jsoftware.com/help/dictionary/dicte.htm>)。待求值的表達式被劃分爲word，每個word動態確定詞性。解釋器會設置一個求值棧，有一系列規約步驟。每一步會進行規約或移入：

- 檢查棧頂四個元素，檢查它們的詞性是否滿足預設的9條規約規則之一，是則規約
- 否則把最右邊的word壓入求值棧

9條規則描述了monadic verb、dyadic verb、adverb、conjunction、hook、fork、括號、賦值等的應用場景：

```
edge           V          N       any         0 Monad
edge AVN       V          V       N           1 Monad
edge AVN       N          V       N           2 Dyad
edge AVN       VN         A       any         3 Adverb
edge AVN       VN         C       VN          4 Conj
edge AVN       VN         V       V           5 Fork
edge           CAVN       CAVN    any         6 Bident
name N         =. =:      CAVN    any         7 Is
(              CAVN       )       any         8 Paren
```

其中edge表示`=. =: (`或表達式左端隱含的哨兵字符，name爲標識符，A、C、N、V分別爲adverb、conjunction、noun、verb。

從中可以歸納出幾個求值相關結論，和之前描述的規則吻合：

- adverb和conjunction優先於verb
- dyadic verb比monadic verb優先
- 連續多個verb後跟一個noun時，從右到左求值
- conjunction是左結合的，且結合verb的能力強於noun

上面所描述的只是J內置verb、adverb、conjunction的滄海一粟，衆多的操作符是J表現力強大的重要原因。操作符大多爲1~2個字符使得J簡短而晦澀。

## 控制結構

J也包含if、for、while等控制結構，但很多時候可以被其他操作符代替。

```
   foo =: verb define    NB. 多行定義一個verb，行首輸入 ) 標示結束
if. 1 do. 1
else. 3
end.
)
```

`if.`後面跟的noun爲空或者首元素非0時爲真。

```
   f =: +&0
   g =: +&1
   h =: +&2
   t =: ]                NB. monadic ] (Same)，返回值爲參數，類似FP中的identity函數
   foo =: verb define    NB. 多行定義一個verb，行首輸入 ) 標示結束
select. t y
case. 0 do. f y
case. 1 do. g y
case. 2 do. h y
end.
)
```

當不需要缺省case時，select往往可以用`` ` ``與`@.`代替。`` ` ``是conjunction，把verb轉化爲gerund(一種box類型，屬於noun)，`@.`則根據`t`的測試結果在gerund array中選擇指定元素，恢復成verb後應用到參數上。

```
   f`g`h
┌─┬─┬─┐
│f│g│h│
└─┴─┴─┘
   foo_alternative =: f`g`h @. t'
```

Verb可以作爲adverb、conjunction的參數，但無法作爲verb的參數，gerund使得它在另一個verb面前成爲first-class object，從中可以幻化出很多奇妙的用法。

上面提到的`^:` (Power of Verb)當右邊參數爲`_`(正無窮大)時表示重複迭代直到值不再發生變化。右邊參數爲verb時：`(u ^: v) y = u ^: (v y)`。兩個`^:`常被連用以表示當條件滿足時則反覆執行，有時可用於替代while循環：
```
   halve =: -:                  NB. monadic -: (Halve)
   even =: 0: = 2&|             NB. 偶數返回1，奇數返回0
   w =: (halve ^: even) ^: _    NB. while (是偶數) 減半
   w_alternative =: verb def 'while. even y do. y =. halve y end. y'
```

另外還有`whilst.` `for.` `try. catch.`等控制結構。

## Array-oriented programming

Wikipedia上J被分類爲array-oriented programming language，J的array操控能力確實非常強大。

J的array有一個屬性稱爲shape，是表示各維長度的列表。比如`i. 2 3 4`的shape是`2 3 4`；單個數、字符稱爲scalar，其shape爲空列表。因此所有noun都具有shape。Shape列表的長度稱爲rank。

Verb有個屬性，也稱爲rank，表示期望接受的參數的維數。當實際參數的rank高於verb的rank時，J解釋器會把參數的shape切分爲兩部分，frame和cells。下表顯示shape爲`2 3 4`的array的可能劃分方案：
```
           frame           cells
           length value    rank shape
0-cells    3      2 3 4    0    empty
1-cells    2      2 3      1    4
2-cells    1      2        2    3 4
3-cells    0      empty    3    2 3 4
```

設verb的rank爲$r$，則shape長爲$r$的後綴成爲$r$-cells，剩餘的前綴部分則作爲frame。各個$r$-cells獨立地被verb作用，產生結果。所有$r$-cells的結果再組裝成shape爲frame的array(當frame爲空時則爲scalar)，爲verb作用在整個參數上的結果。這個過程就像FP語言中的多層map，或者說像多層隱式的循環。

嘗試用conjunction `"`改變`#`的rank(並不準確，這裏爲方便描述暫這樣表達，實際上產生了一個新verb，其rank爲2，可看作`#`的代理)，作用到`i. 2 3 4`上：

```
   # i. 2 3 4
2
   #"3 i. 2 3 4
2
   #"2 i. 2 3 4
3 3
   #"1 i. 2 3 4
4 4 4
4 4 4
```

行首三空格是jconsole的提示符。

`#"3`的rank爲3，`i. 2 3 4`的shape `2 3 4`被拆爲空的frame和cells `2 3 4`兩部分，`#`作用在唯一的3-cells上得到結果，和`# i. 2 3 4`相同。
`#"2`的rank爲2，`i. 2 3 4`的shape `2 3 4`被拆爲frame `2`和cells `3 4`兩部分，`#`獨立地作用在2個2-cells上得到2個結果，2個結果拼裝爲一個array(frame即爲其shape)。
`#"1`的rank爲1，`i. 2 3 4`的shape `2 3 4`被拆爲frame `2 3`和cells `4`兩部分，`#`獨立地作用在2\*3個2-cells上得到6個結果，2\*3個結果拼裝爲一個2維array(frame即爲其shape)。

善加利用rank可以省卻很多for循環。

上文討論的是monadic verb的rank，對於dyadic verb，兩個參數可以有不同的rank。比如dyadic `+`用於計算兩數之和，其左右參數的rank均爲0。不難想象它可以用於計算兩個array的和、兩個矩陣的和、或是兩個更高維array的和，只要它們的shape一致。實際上`+`兩個參數可以具有不同shape，比如`3 4 5 + (i. 3 2)`，左邊參數shape爲3，右邊參數shape爲3 2。Dyadic `+`的左右rank均爲0，左邊參數的frame爲3，右邊參數的frame爲3 2。這個計算是合法的理由是較短的frame是較長的frame的前綴。具體發生的過程比較複雜，`3 4 5`每個數被重複多份，化作了和較長frame相同的形狀(3\*2)：
```
3 3
4 4
5 5
```
參與運算。

## Boxing

爲了支持異構array，J提供了另一種scalar數據類型：box。數、字符、字串(字符array)、array等都可用monadic `<`操作符轉成box，box也可以嵌套：
```
   < 1
┌─┐
│1│
└─┘
   < 1 2
┌───┐
│1 2│
└───┘
   <'abc'
┌───┐
│abc│
└───┘
   < < 1
┌───┐
│┌─┐│
││1││
│└─┘│
└───┘
```

Box是scalar，可以和其他box並置組成array，而各box的內容的類型則允許不同。J通過這一機制實現異構array，可以模擬C的struct：
```
   'abc' ; 1 2 3 ; (<3); (i.2 2)
┌───┬─────┬───┬───┐
│abc│1 2 3│┌─┐│0 1│
│   │     ││3││2 3│
│   │     │└─┘│   │
└───┴─────┴───┴───┘
```

Monadic `>`用於取出box中的內容：
```
   > < 1 2
1 2
   > < 'abc'
abc
   > < < 1
┌─┐
│1│
└─┘
```

Box可以用於包裝多個不同類型的參數，從而用於形成複雜數據結構、模擬多元verb：
```
   <0;1;<<1
┌─────────┐
│┌─┬─┬───┐│
││0│1│┌─┐││
││ │ ││1│││
││ │ │└─┘││
│└─┴─┴───┘│
└─────────┘
   (<0;1;<<1) { i. 2 2 3         NB. 獲取array中相應元素：第0維取0、第1維取1、第2維取非1(即0,2)
3 5
```

Conjunction `&.`名爲Dual，`u&.v y => v^:_1@u@v y`，即把參數按`v`變形後，應用`u`，之後再應用`v`的逆運算。

`<`和`>`互爲逆運算，可以用conjunction `b.`確認：
```
   >b._1
<
   <b._1
>
```

對於box類型有個技巧：`&.>`，這是conjunction `&.`固定右邊參數(Bident規則)後得到的adverb，效果是解開box後採取特定運算，再box回去，類似Perl的Schwartzian transform，例如：
```
   (1&+)&.> 4;5;6
┌─┬─┬─┐
│5│6│7│
└─┴─┴─┘
```

## Tacit programming

J中tacit programming有兩個基石，其一是adverb、conjunction，另一個是hook和fork。

Hook和fork是J中的一套語法，用於把參數傳遞給多個verb而不用引用參數的變量名，實現tacit programming，規則如下：
```
(V0 V1) Ny         =>   Ny V0 (V1 Ny)                monadic hook
(V0 V1 V2) Ny      =>   (V0 Ny) V1 (V2 Ny)           monadic fork
```

先看monadic hook，兩個verb並置會形成一個新verb，接受的參數會變成雙份，這條規則酷似SKI combinator calculus中的S combinator，`S x y z = x z (y z)`。Dyadic hook是其二元變體。

Monadic fork的設計非常精巧和簡潔，三個verb並置。若參數只被使用一次，則不難用`&` (Bond)以及一系列函數組合實現。當參數要被使用多次時，通常希望在被處理後有個匯合過程。Fork的設計精確地刻畫了這一點。效果類似Haskell Reader monad的`liftM2`：
```
% pointfree '\x y r -> x r + y r'
liftM2 (+)
```

5、7、9……個verb也能形成fork，可以看作右邊3個verb形成fork後再跟其餘verb形成fork：`V0 V1 V2 V3 V4 = V0 V1 (V2 V3 V4)`。Fork規則的設計精妙如此，可以輕易擴展。

Fork產生的verb也可以用作dyadic，參與fork的V0可以換成noun，得到一些派生規則：
```
Nx (V0 V1) Ny      =>   Nx V0 (V1 Ny)                dyadic hook
Nx (V0 V1 V2) Ny   =>   (Nx V0 Ny) V1 (Nx V2 Ny)     dyadic fork
(N0 V1 V2) Ny      =>   N0 V1 (V2 Ny)                monadic noun fork
Nx (N0 V1 V2) Ny   =>   N0 V1 (Nx V2 Ny)             dyadic noun fork
```

`13 : n`可以讓解釋器幫你找tacit形式：

```
   13 : 'x-y'
-
   13 : '(4*x)-y'
] -~ 4 * [
```

## Conjunction彌補操作符數目的不足

J把ASCII可見字符中幾乎所有標點符號(可能是所有)都用上作爲primitive了，另外把`.`、`:`跟在操作符後又得到大量primitive，即便如此畢竟無法涵蓋所有常用的verb。因此又有一系列conjunction應運而生，用不同的左參數來表示不同的函數，比如`o.`家族：

```
cop=: 0&o.        NB. sqrt (1-(y^2))
sin=: 1&o.        NB. sine of y
cos=: 2&o.        NB. cosine of y
tan=: 3&o.        NB. tangent of y
coh=: 4&o.        NB. sqrt (1+(y^2))
sinh=: 5&o.       NB. hyperbolic sine of y
cosh=: 6&o.       NB. hyperbolic cosine of y
tanh=: 7&o.       NB. hyperbolic tangent of y
...
```

相當晦澀，好在編號還是有一些規律，比如奇數表示奇函數，偶數表示偶函數。大量系統函數也是用這種方式定義的：<http://www.jsoftware.com/jwiki/Vocabulary/Foreigns>。

J中定義了不少常量來緩解難以記憶的問題：
```
   plus =: verb define
x+y
)
   verb
3
   define
 :0
```

## Object-oriented programming

J中實現了namespace，和J中的很多其他概念一樣，給了它一個術語叫locale。同一時刻可以有多個locale，每個locale存放了一些變量定義，同一時刻一個標識符可能存在於多個locale裏，但具有不同的定義。J的REPL環境裏的默認locale爲`base`。每個locale可以定義search path，當找不到某標識符定義時到search path的locale裏找。另外有一個特殊locale名爲`z`，若search path也找不到某標識符定義時會引用`z`裏的定義。`z`中定義了大量庫函數。

```
   coname ''     NB. 當前locale名稱。標識符`coname`來自locale `z`
┌──┐
│bb│
└──┘
   nl ''         NB. 當前locale定義的標識符列表。`nl`來自locale `z`

   foo =: 0
   nl ''
┌───┐
│foo│
└───┘
```

J定義了兩個語法引用指定locale的標識符：

- `name_loc_`。臨時切換到locale `z`，引用標識符`name`，之後回到原先的locale。
- `name__var`。`var`爲locale名的box，通過變量`var`間接引用，也會臨時切換locale。

```
   names_z_ ''   NB. 臨時切換到locale `z`後執行verb調用，因此輸出`z`定義的標識符列表
...
```

Locale語法在很多地方和JavaScript的prototype相似，它被用於實現prototype-based object-oriented programming。Class的方法定義在一個單獨的locale裏，每個object實例也會分配到一個新的locale，設置search path爲其class，優先查找自身定義的標識符，若不存在則會在class的locale裏找。通過設置class locale的search path可以實現繼承。

下面代碼定義了一個class：
```
   coclass 'Stack'                     NB. 切換到locale `Stack`
   create =: verb def 'a =: 0 $ 0'     NB. constructor，被下文的dyadic conew引用
   destroy =: codestroy                NB. destructor，手動調用，用於刪除實例的locale
   push =: verb def '# a =: (<y) , a'  NB. 方法
   top =: verb def '> {. a'
   pop =: verb def '# a =: }. a'
   cocurrent 'base'                    NB. cocurrent與coclass等價。回到locale `base`
```

```
   ] S =: 0 conew 'Stack'              NB. 創建Stack的實例S，相伴的locale爲`0`
┌─┐
│0│
└─┘
   copath <'0'                         NB. search path爲Stack與z
┌─────┬─┐
│Stack│z│
└─────┴─┘
   push__S 'meow'                      NB. 調用方法，在locale `0`中執行
   push__S 'hello'
   (top__S '') (1!:2) 2                NB. 打印棧頂，在locale `0`中執行
   pop__S ''
   (top__S '') (1!:2) 2
   destroy__S ''                       NB. 在locale `0`中執行，刪除locale `0`
   erase <'S'                          NB. 刪除locale `base`中的名字S
```

規則很簡單，是個優雅的設計。

## APL

[J for the APL Programmer](http://www.jsoftware.com/papers/j4apl.htm)可以瞭解J和前輩APL的差異，<http://www.jsoftware.com/jwiki/Essays/Bibliography>上能找到大量有關APL和J淵源的文章。

關於APL有段評論：

> APL is like a diamond. It has a beautiful crystal structure; all of its parts are related in a uniform and elegant way. But if you try to extend this structure in any way - even by adding another diamond - you get an ugly kludge. LISP, on the other hand, is like a ball of mud. You can add any amount of mud to it and it still looks like a ball of mud." -- Joel Moses

用於描述J也很合適。J語法簡單，一致性強，很多細節上精巧的涉及和大量簡短的原語造就了強大的表現力，但擴展它的功能很困難。不管怎麼樣，它是個不錯的桌上計算器(不需要導入庫，默認即有大量primitive)，我在xmonad配置(<https://github.com/MaskRay/Config/tree/master/home/.xmonad/xmonad.hs>)裏也爲它單獨分配了一個快捷鍵以運行J的REPL環境。改變你對編程語言的刻板印象，它已經出色地完成了任務，不是嗎？

## 安裝與運行

J system下載頁面：<http://www.jsoftware.com/stable.htm>。Arch Linux用戶可以安裝AUR裏的j8-git。

可執行文件`jconsole`爲命令行解釋器，`jqt`爲Qt圖形界面。j8-git則提供了`/usr/bin/j8`。

## 學習材料

<http://www.jsoftware.com/jwiki/Books/Beginners>列出了一些關於J的書。如果不願花費太大工夫、只想簡單瞭解的話，可以看44頁的[Easy J](http://www.jsoftware.com/books/pdf/easyj.pdf)，深入學習的話則可以看較全較新的[J for C Programmers](http://www.jsoftware.com/help/jforc/contents.htm)或[Learning J](http://www.jsoftware.com/help/learning/contents.htm)。[Exploring Math](http://www.jsoftware.com/books/pdf/expmath.pdf)和J Phrases給出了大量例子，可作爲習題集閱讀。

上面提到的書中很多能在<http://www.cs.trinity.edu/About/The_Courses/cs2322/j-books/>找到PDF版本。

<http://www.jsoftware.com/jwiki/NuVoc>是詞典，用於查閱各primitive的語義。

<http://rosettacode.org/wiki/Category:J>也有大量J代碼。可以寫一段UserScript以自動跳轉到J：
```javascript
// ==UserScript==
// @name         RosettaCode jump to J
// @match        http://rosettacode.org/wiki/*
// ==/UserScript==
location.href = '#J'
```
或者下載別人整理的代碼：<https://github.com/acmeism/RosettaCodeData/tree/master/Lang/J>。

運行jqt，打開Tools->Package Manager，安裝`labs/labs`，之後在Help->Studio->Labs裏能找到一些教程。
