---
layout: post
title: ML編譯器Caml Featherweight——設計
author: MaskRay
tags: [compiler, ocaml, ml]
mathjax: true
---

## 編譯器的設計

編譯器的設計和實現大量參考了 @Zinc。

<!-- more -->

### 執行方法

編譯器的執行方法大致分成四類：

1.  解釋執行語法樹。容易實現，解析得到語法樹後再做較少的工作即可實現。運行時解釋執行，但性能很低。
2.  生成本地代碼，處理器可以直接執行。優點是性能較高。
3.  生成在抽象機器上執行的字節碼。類似於本地代碼，但生成的代碼基於一個抽象機器。優點是移植性強，只需把運行時系統移植到目標架構上即可。
4.  翻譯到其他高級語言。和生成抽象機器的字節碼類似，但前者過於低級，實現很多結構需要大量指令。編譯到一門高級語言在很多地方上能得到簡化。

### 語法樹設計和類型信息表示

#### 表達式

我們需要一個表示表達式的類型，爲了提供更具體的解析錯誤信息，需要把位置信息放在附在節點信息裏。表示位置信息有兩種方法，一種是在所有構造器裏設置一個域表示：

``` {.ocaml}
type expression =
  | Pexpr_apply of location * expression * expression list
  | Pexpr_array of location * expression list
  | ...
```

這樣在所有模式匹配的地方都要匹配位置域，很不方便。如果把構造器都改成record類型的話，模式匹配也會不方便。

比較好的方式是拆分爲兩個類型：

``` {.ocaml}
type expression = { e_desc: expression_desc; e_loc: location }
and expression_desc =
  | Pexpr_apply of expression * expression list
  | Pexpr_array of expression list
```

如果前端工作複雜的話，那麼`expression`還可以考慮設計成參數化的。但在我們的實現中只需要使用到一種語法樹表示，沒必要使用參數化。

``` {.ocaml}
type expression = { e_desc: expression_desc; e_loc: location }
and expression_desc =
  | Pexpr_apply of expression * expression list
  | Pexpr_array of expression list
  | Pexpr_constant of constant
  | Pexpr_constraint of expression * type_expression
  | Pexpr_constr of long_ident * expression option
  | Pexpr_for of string * expression * expression * bool * expression
  | Pexpr_function of (pattern * expression) list
  | Pexpr_ident of long_ident
  | Pexpr_if of expression * expression * expression option
  | Pexpr_let of bool * (pattern * expression) list * expression
  | Pexpr_sequence of expression * expression
  | Pexpr_try of expression * (pattern * expression) list
  | Pexpr_tuple of expression list
```

#### 模式匹配

模式匹配和表達式共享了一些語法單元，比如字面量、元組、構造器、變量等，但其中不允許出現`for`、`;`等，和表達式還是有一些差別，因此需要爲它定義一個類型`pattern`。

`pattern`是個和`expression`平級的概念，同樣出於錯誤信息的考量，添加位置信息並拆分爲兩個類型：

``` {.ocaml}
and pattern = { p_desc: pattern_desc; p_loc: location }
and pattern_desc =
  | Ppat_alias of pattern * string
  | Ppat_any
  | Ppat_array of pattern list
  | Ppat_constant of constant
  | Ppat_constraint of pattern * type_expression
  | Ppat_constr of long_ident * pattern option
  | Ppat_or of pattern * pattern
  | Ppat_tuple of pattern list
  | Ppat_var of string
```

#### 類型表達式

`expression`和`pattern`中允許出現類型限定(如`int`、`’a list`)，爲此需要一個`type_expression`類型表示，這也是一個和`expression`平級的概念。

``` {.ocaml}
and type_expression = { te_desc: type_expression_desc; te_loc: location }
and type_expression_desc =
  | Ptype_var of string
  | Ptype_arrow of type_expression * type_expression
  | Ptype_tuple of type_expression list
  | Ptype_constr of long_ident * type_expression list
```

#### Type constructor、具體類型

解析階段以外用到的、類型相關 由於Caml支持參數化類型，因此如何表示類型、構造器等的描述變得很複雜。

考慮如何統一表示無參數的類型(如`int`、`string`)和參數化類型(如`’a option`)，它們都是由type constructor(`int`、`string`、`option`)接受零或多個參數得到的，type constructor可以看作是一個獨一無二的標識符，我們用`type_constr`表示。

具體的類型用`typ`表示，可以由type constructor應用到參數上得到(`Tconstr`)，也可以通過product類型或arrow類型等方式得到，也可以是一個generic類型(`Tvar`，`typ_level`爲-1)或者是一個尚未確定的類型(`Tvar`，`level`非-1)。

另外Caml支持type abbreviation，type constructor可能指向另一個具體類型。

綜合上述考量得到下面的定義：

``` {.ocaml}
type typ = { typ_desc: typ_desc; mutable typ_level: int }
and typ_desc =
  | Tarrow of typ * typ
  | Tconstr of type_constr global * typ list
  | Tproduct of typ list
  | Tvar of typ_link ref
and typ_link =
  | Tnolink
  | Tlink of typ
and type_constr = { ty_stamp: int; mutable ty_abbr: type_abbrev }
and type_abbrev =
  | Tnotabbrev
  | Tabbrev of typ list * typ
```

#### 類型描述和構造器描述

對於類似`type t = A`這樣的類型定義，以及一些內置類型(如`float`)等，用`type_desc`表示，並存儲在一個全局的表裏，供編譯的後續階段引用。參數化類型允許參數爲任意類型，因此`type_desc`中只需要記錄參數個數(`ty_arity`)。等號右邊的描述則用`type_components`類型表示，可以是abstract type、variant type或是type abbreviation。

與`type_desc`關聯的是構造器描述`constr_desc`，表達式或模式匹配中出現構造器時會用到這個類型，它需要的域是所表示的具體類型(`typ`)，是否接受參數(`cs_kind: constr_kind`)及接受的參數的類型(`of`後面的)等。另外需要一個域`cs_tag: constr_tag`用於區分同一個具體類型的不同構造器。

``` {.ocaml}
type type_desc =
  { ty_constr: type_constr global; ty_arity: int; mutable ty_desc: type_components }
and type_components =
  | Abstract_type
  | Variant_type of constr_desc global list
  | Abbrev_type of typ list * typ

and constr_desc =
  { cs_arg: typ; cs_res: typ; cs_tag: constr_tag; cs_kind: constr_kind }

and constr_kind =
  | Constr_constant
  | Constr_regular
  | Constr_superfluous of int
```

其中`constr_tag`描述構造器的類型。考慮模式匹配的代碼生成場景，我們需要一種辦法能區分同一個類型的不同構造器。如果構造器是closed type的，那麼區分不同的構造器只需要一個信息，即構造器的標號，同一個類型的不同構造器標號互不相同。爲了代碼生成方便，構造器的數目也編碼到標號中，可以用一個元組`int * int`來表示標號。

爲了一致性，`exception`類型也應看作構造器，而`exception`類型是開放的，即構造器可以定義在不同地方，數目不受限制，可以實現爲一種extensible type，這些類型用標識符表示。

``` {.ocaml}
type constr_tag =
  | Constr_tag_regular of int * int
  | Constr_tag_extensible of long_ident * int
```

### 詞法分析

#### 詞法分析

使用工具ocamllex。這一部分實現在`lexer.mll`中。

Caml支持自定義操作符，它們的優先級由第一個字符決定，優先級會對語法分析產生影響，因此把優先級不同的操作符表示爲不同的詞素。

Caml Light沒有對大寫和小寫開頭的標識符進行區分，但參閱OCaml源碼可以看到區分後語法分析會更加容易。因此我們的項目中也按大小寫進行區分。

還有一些需要注意的地方是字符串、字符、註釋等的解析，需要考慮轉義字符等問題，但都不是難點。

#### 語法分析

這一部分實現在`parser.mly`中。使用工具ocamlyacc或menhir。我還不知道如何使用這兩個工具產生友好的錯誤信息。

多數yacc實現都採用了操作符優先級來解決移入/規約衝突，<http://caml.inria.fr/pub/docs/manual-ocaml-4.00/manual026.html>描述了這一解決過程。在實現中我們碰到一些典型問題並得以解決。

##### 解析逗號分割的列表

考慮表示表達式的非終結符`expr`，它的其中兩個產生式是逗號分割的表達式和分號分割的表達式，約定逗號(`COMMA`)優先級高於分號(`SEMI`)，可以如下實現：

    expr:
      | ...
      | expr_comma_list %prec below_COMMA { Pexpr_tuple($1) }
      | expr SEMI expr { Pexpr_seq($1,$3) }
      | ...

    expr_comma_list:
      | expr COMMA expr_comma_list { $1 :: $3 }
      | expr COMMA expr { [$1; $3] }

其中`%prec below_COMMA`是指定第一條產生式的優先級，從而在棧爲`expr_comma_list`且向前看符號爲`COMMA`時，menhir選擇移入而不是規約`expr_comma_list -> expr`。

但這種語法描述還有一個問題。當棧內當前爲`expr COMMA expr_comma_list`且向前看符號爲`SEMI`時，規約`expr_comma_list -> expr`和規約`expr COMMA expr_comma_list -> expr`的優先級均大於`SEMI`，均爲有效的規約，無法確定選取哪一個，產生規約/規約衝突。

解決方案是把左遞歸改寫爲右遞歸：

    expr_comma_list:
      | expr_comma_list COMMA expr { $3 :: $1 }
      | expr COMMA expr { [$3; $1] }

在使用`expr_comma_list`時要注意反轉列表。考慮到函數式語言中在列表頭部添加元素比較容易，描述文法時通常用左遞歸而非右遞歸，但在上述情形下就不得已採用右遞歸。OCaml的語法解析更爲複雜，在`parsing/parser.mly`中也能看到一些這樣的例子。

#### 解析標識符，區分構造器和變量

Caml Light的語法分析文件中把實現文件劃分爲用雙分號分割的多個phrase，解析完一個phrase後立即進行類型檢查、代碼生成等工作，並導入全局的值、類型、構造器信息。在解析模式匹配的地方遇到一個標識符時，通過查詢之前是否定義過該標識符的構造器來區分該標識符是構造器還是變量。Caml Light把部分變量是否定義的工作糅合到了解析階段。

我們希望能完整解析源文件後再進行變量定義判斷的工作，因此借鑑了OCaml的解析器實現，區分大小寫兩種標識符，大寫視爲構造器，小寫視爲變量。

在這裏也能一窺語言實現的考量，瞭解爲什麼很多支持模式匹配的語言規定構造器使用大寫、變量使用小寫。

這裏也能看到爲了語言的一致性，`false`和`true`應該看作構造器，使用大寫，實現中如果要擯棄Caml Light中解析階段判斷是否定義的做法，比較好的方式是在詞法分析中把`false`和`true`也作爲詞法單元。

##### 可省略的雙分號

Caml Light中要求每一個phrase(表達式、`let`定義、`type`定義或`exception`定義)後面必須跟雙分號，而OCaml中，非表達式phrase前的雙分號是可省略的。

方法是區分表達式開始的實現和非表達式開始的實現。我們用用一個非終結符`structure_item`表示單個非表達式phrase。

實現文件有三種可能：

-   只包含一個表達式phrase。
-   以表達式phrase開頭，之後接雙分號。
-   以非表達式phrase開頭(用`structure_tail`表示)

`structure_tail`可以看作是實現和表達式開頭的實現的差集，有幾種可能：

-   空或僅包含雙分號
-   `SEMISEMI seq_expr structure_tail`
-   `SEMISEMI structure_item structure_tail`
-   `structure_item structure_tail`，這種情況對應了省略雙分號的情形

<!-- -->

    implementation:
      | structure EOF { $1 }

    structure:
      | structure_tail { $1 }
      | seq_expr { [make_impl(Pimpl_expr $1)] }
      | seq_expr SEMISEMI structure_tail { make_impl(Pimpl_expr $1)::$3 }

    structure_tail:
      | /* emtpy */ { [] }
      | SEMISEMI { [] }
      | SEMISEMI seq_expr structure_tail { make_impl(Pimpl_expr $2)::$3 }
      | SEMISEMI structure_item structure_tail { $2::$3 }
      | structure_item structure_tail { $1::$2 }

    structure_item:
      | TYPE type_decl_list { make_impl(Pimpl_typedef $2) }
      | LET rec_flag let_binding_list { make_impl(Pimpl_letdef($2, $3)) }
      | EXCEPTION constr_decl { make_impl(Pimpl_excdef $2) }

##### Caml Light語法分析的一些缺陷

Caml Light的語法分析文件還存在其他一些缺陷，包括符號優先級定義比較混亂，很多地方應該用`%nonassoc`而不是`%left`或`%right`等。

### 數據表示

### First-class function的實現

支持first-class function需要支持把函數作爲值傳遞，可以存儲在變量中、用作返回值，支持內嵌的函數。內嵌函數作爲值傳遞時，它仍可以訪問上層函數的局部變量，因此如果作爲傳遞到外界，需要提供一種方式訪問那些變量。

一種方式是閉包，把需要訪問的變量和函數代碼一起存儲。函數代碼可以用指向待執行指令的指針表示，而內嵌函數需要訪問的上層函數的局部變量就需要保存在一個數據結構裏，稱爲環境。@Compiling中提到了一些閉包實現策略的考量。函數應用時沒有訪問環境，只有構建閉包的代碼和訪問相應變量的代碼需要。在選擇環境的表示上我們有相當的自由度。

另一種方式是lambda lifting，可以實現first-class function的部分特性。

### 編譯流程概覽

-   解析源代碼得到抽象語法樹
-   類型推導得到帶類型的語法樹
-   翻譯成使用de Bruijn index的擴充$\lambda$-calculus。原始的無類型$\lambda$-calculus中唯一的值類型是$\lambda$抽象，該擴充$\lambda$-calculus提供了let、原語操作(如加法、比較、創建數組、創建boxed data等)。這一部分還包括一個編譯模式匹配的子系統。
-   把擴充$\lambda$-calculus翻譯到Zinc抽象機器
-   運行時系統解釋執行Zinc抽象機器字節碼
