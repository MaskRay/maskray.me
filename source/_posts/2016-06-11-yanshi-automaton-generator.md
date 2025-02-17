---
layout: post
title: 偃師——finite automaton生成工具
author: MaskRay
tags: [automaton,parser generator,regular expression,yanshi]
---

我一隻上海的喵怎麼到北京來了呢？……“苟利長亭生死以，豈因禍福避趨之”那麼所以我就到了北京。到了北京我幹了這二十幾天我也沒有什麼別的，大概三件事：

一個，開發了一個安全產品(祕)；
第二個，做了一個自動機生成工具；
第三個，就是見了一些同學，從學長那裏聽到了一點人生的經驗。

<!-- more -->

如果說還有一點什麼成績就是收了Code Jam、百度之星、2016 Topcoder Open Regional Beijing三件T恤，但這些都是次要的。

# 偃師

這個自動機生成工具叫做偃師，名字取自《列子·湯問》中記載的一位工匠，善於製造能歌善舞的人偶。

## 示例

編輯`a.ys`：
```
import 'c.ys' as C

export main = goodnight C::world

action good {
  puts("good");
}

goodnight = good night
good = [Gg] 'o'{2} 'd' @ good
night = ("ni" [Gg] "ht" - 'niGht') @ C::night
```

`c.ys`：
```
action night {
  puts("night");
}

world = ('\u0076' .. '\u0077' & '\u0077' .. '\u0078') 'orld' > {puts("world");}
```

根據該文法生成表示`main`的DFA，`{}`裏的是action。

```
make
build/yanshi -S /tmp/a.ys -o /tmp/a.cc && make -C /tmp/a
/tmp/a 'Goodnightworld'
```

輸出：
```
good
night
world
len: 14
state: 14
final: true
```

## 流程

- `lexer.l,parser.y`。解析命令行指定的`.ys`文件，構建abstract syntax tree
- `loader.cc`。每個文件視爲一個模塊，
  + 記錄nonterminal定義
  + 若有`import "filename.ys"`則遞歸載入
  + 把每個nonterminal定義中引用的標識符(可以用模塊名限定)與其定義關聯。每個nonterminal代表一個頂點，依賴關係爲邊，建圖。
  + 對依賴圖用post-order DFS進行拓撲排序。若有循環依賴則報錯。
  + 按拓撲序編譯各nonterminal，每個nonterminal編譯成一個automaton，若A依賴B，則B先編譯。
  + 對於每個標記爲`export`的nonterminal，爲其生成C++代碼。

## 詞法分析

使用flex進行詞法分析。下面描述用到的技巧。

### 詞法元素添加位置信息

使用`%option bison-bridge`生成與bison協作的代碼，`lexer.l`中可以訪問`yylval`並設置其各個域，以向bison返回詞法元素信息。

用`%option extra-type="long"`記錄一個額外信息，表示文件偏移。使用`%option bison-locations`以訪問`yylloc`，定義`#define YY_USER_ACTION`即可給bison提供詞法元素的位置，並更新文件偏移。

```c++
#define YY_USER_ACTION                      \
  do {                                      \
    yylloc->start = yyget_extra(yyscanner); \
    yylloc->end = yylloc->start + yyleng;   \
    yyset_extra(yylloc->end, yyscanner);    \
  } while (0);
```

### Off-side rule

`.ys`文件中可以用`semicolon`或`nosemicolon`指定用分號或換行符標記Stmt結束。

```
semicolon;
c++ {
};
foo = !meow;

nosemicolon
meow = !foo
```

```lex
{%
static bool semicolon;
%}

"semicolon" semicolon = true;
"nosemicolon" semicolon = false;
";" if (semicolon) return '\n';
"\n" if (YY_START == INITIAL && ! semicolon) return '\n';
```

偃師當前沒有用到縮進。若要支持縮進文法，注意到各行縮進數形成了樹，可以用棧維護當前行所有祖先頂點的縮進數，用flex規則記錄換行符前空格數。

### Start conditions切換詞法分析模式

就像lex、yacc，偃師`.ys`文件中`c++ {}`大括號內可以引入任意C++代碼。詞法分析時要能判斷是否在代碼塊內部，若是，那麼大部分頂層規則失效，遇到配對的`}`則退出代碼模式。

```
c++ {
  // arbitrary C++ code
  "\a{";
  {if(0){foo();}}
  'a';
}
```

解決方案是使用flex的start conditions，`{`、`"`、`'`、`(`等均會改變start condition，改變對字符的處理方式。爲了正確處理這些特殊字符的嵌套，需要把之前的狀態保存在棧中。flex提供了`%option stack`以使用`yy_push_state`、`yy_pop_state`，不過需要注意的是實際生效的狀態是當前start condition `YY_START`而不是`yy_top_state()`。

```lex
%x EXPECT_CODE
%x IN_CODE
%s IN_PAREN
%x IN_Q_STRING
%x IN_QQ_STRING

<EXPECT_CODE>{
  "{" BEGIN IN_CODE; tmp_bracket.clear();
  /* ...... */
}

<IN_CODE>{
  "'" { tmp_bracket += '\''; yy_push_state(IN_Q_STRING, yyscanner); }
  "\"" { tmp_bracket += '"'; yy_push_state(IN_QQ_STRING, yyscanner); }
  "{" { tmp_bracket += '{'; yy_push_state(IN_CODE, yyscanner); }
  "}" {
    yy_pop_state(yyscanner);
    if (YY_START == INITIAL || YY_START == IN_PAREN) {
      yylval->str = new string(tmp_bracket);
      return BRACED_CODE;
    } else
      tmp_bracket += '}';
  }
  .|"\n" tmp_bracket += yytext[0];
  <<EOF>> yy_pop_state(yyscanner); unexpected_eof(yylval, "}");
}

' tmp_str.clear(); yy_push_state(IN_Q_STRING, yyscanner);
"\"" tmp_str.clear(); yy_push_state(IN_QQ_STRING, yyscanner);
<IN_Q_STRING>{
  ' {
    yy_pop_state(yyscanner);
    if (YY_START == INITIAL || YY_START == IN_PAREN) {
      yylval->str = new string(tmp_str);
      return STRING_LITERAL;
    }
    tmp_bracket += yytext;
  }
  <<EOF>> yy_pop_state(yyscanner); unexpected_eof(yylval, "'");
}
<IN_QQ_STRING>{
  "\"" {
    yy_pop_state(yyscanner);
    if (YY_START == INITIAL || YY_START == IN_PAREN) {
      yylval->str = new string(tmp_str);
      return STRING_LITERAL;
    }
    tmp_bracket += yytext;
  }
  <<EOF>> yy_pop_state(yyscanner); unexpected_eof(yylval, "\"");
}

<IN_Q_STRING,IN_QQ_STRING>{
  \\[0-7]{1,3} /* ...... */
  \\x[0-9a-fA-F]+ /* ...... */
  .|\n tmp_str += yytext[0]; tmp_bracket += yytext[0];
  /* ...... */
}
```

## 語法分析

### 位置信息

定義`#define YYLTYPE Location`指定用`struct Location { long start, end; };`表示位置信息。定義`#define YYLLOC_DEFAULT(Loc, Rhs, N)`，在規則匹配成功後設定`yyloc`，把該信息複製到abstract syntax tree的節點中。

```yacc
%code requires {
/* struct Location { long start, end; ...... }; */
#define YYLTYPE Location
#define YYLLOC_DEFAULT(Loc, Rhs, N)             \
  do {                                          \
    if (N) {                                    \
      (Loc).start = YYRHSLOC(Rhs, 1).start;     \
      (Loc).end = YYRHSLOC(Rhs, N).end;         \
    } else {                                    \
      (Loc).start = YYRHSLOC(Rhs, 0).end;       \
      (Loc).end = YYRHSLOC(Rhs, 0).end;         \
    }                                           \
  } while (0)
}

%locations

%%

stmt:
    define_stmt { $$ = $1; }
  | IMPORT STRING_LITERAL AS IDENT { $$ = new ImportStmt(*$2, *$4); delete $2; delete $4; $$->loc = yyloc; }
  | IMPORT STRING_LITERAL { string t; $$ = new ImportStmt(*$2, t); delete $2; $$->loc = yyloc; }
  | ACTION IDENT BRACED_CODE { $$ = new ActionStmt(*$2, *$3); delete $2; delete $3; $$->loc = yyloc; }
  | CPP BRACED_CODE { $$ = new CppStmt(*$2); delete $2; $$->loc = yyloc; }
```

## Abstract syntax tree表示

分成三類：Stmt、Expr、Action。

爲構成Expr的每種語法都創建一個Expr子類，如`BracketExpr`表示`[0-9a-z]`、`DifferenceExpr`表示差`A-B`、`UnionExpr`表示並`A|B`等。

分析模塊定義的nonterminal、關聯標識符引用與定義、根據nonterminal定義生成automaton均需遍歷AST，每一種遍歷方式對不同`Expr`子類的處理方式不同，可以應用double dispatch的visitor pattern。

以Action爲例。

`Action`繼承自`VisitableBase`，聲明`virtual void accept(Visitor<T>& visitor) = 0;`，各子類實現`accept`：調用visitor中以子類具體類型爲參數的重載方法。

實現了`Expr`遍歷的visitor繼承自`Visitor<Expr>`，遞歸訪問子樹時應調用`visit(Expr&)`，以便在pre-order和post-order時執行各個hooks。

```c++
template<class T>
struct Visitor;

template<class T>
struct VisitableBase {
  virtual void accept(Visitor<T>& visitor) = 0;
};

template<class Base, class Derived>
struct Visitable : Base {
  void accept(Visitor<Base>& visitor) override {
    visitor.visit(static_cast<Derived&>(*this));
  }
};
```

```c++
struct Action;
struct InlineAction;
struct RefAction;
template<>
struct Visitor<Action> {
  virtual void visit(Action& action) = 0;
  virtual void visit(InlineAction&) = 0;
  virtual void visit(RefAction&) = 0;
};

struct Action : VisitableBase<Action> {
  Location loc;
  virtual ~Action() = default;
};

struct InlineAction : Visitable<Action, InlineAction> {
  string code;
  InlineAction(string& code) : code(move(code)) {}
};

struct Module;
struct RefAction : Visitable<Action, RefAction> {
  string qualified, ident;
  Module* define_module; // set by ModuleUse
  RefAction(string& qualified, string& ident) : qualified(move(qualified)), ident(move(ident)) {}
};

struct PrePostStmtVisitor : Visitor<Stmt> {
  void visit(Stmt& stmt) override {
    // pre-order
    stmt.accept(*this);
    // post-order
  }
  void visit(ActionStmt& stmt) override {}
  void visit(CppStmt& stmt) override {}
  void visit(DefineStmt& stmt) override {}
  void visit(EmptyStmt& stmt) override {}
  void visit(ImportStmt& stmt) override {}
};
```

## `import "filename.ys"`的處理與多文件支持

`ImportStmt` `import "filename.ys"`可以導入其他文件定義的nonterminal，或者用`import "filename.ys" as B`帶限定地導入。

解析完一個文件後，訪問各個Stmt，若有nonterminal定義(`DefineStmt`)則記錄下來；若有`ImportStmt`則遞歸載入其他文件。爲了支持循環導入，一個文件只會被導入一次，打開文件後，判斷fstat(2)獲取的`st_dev`和`st_ino`是否之前出現過。

記錄完所有模塊的定義後再把產生式右邊的標識符引用和定義關聯。

## 循環依賴的檢測

對依賴圖用post-order DFS進行拓撲排序。每個頂點可能處於四種狀態之一：

0. 未訪問
1. 遍歷中，當前頂點是它的後裔，所有狀態1頂點在棧中
2. 所有後裔均已訪問過，不在環中
3. 所有後裔均已訪問過，在環中

枚舉各個頂點，若狀態爲0則DFS，若遇到狀態爲1的頂點則找到了一個環，從而輸出循環依賴錯誤。

比如如下`.ys`文件
```
earth = venus
venus = mars arch
mars = earth

arch = felix
felix = cat
cat = arch
```
會生成如下錯誤：
```
b.ys 2:1-17 'venus': circular embedding
  venus = mars arch
  b.ys 1:1-13 required by earth
  earth = venus
  b.ys 3:1-12 required by mars
  mars = earth
  b.ys 2:1-17 required by venus
  venus = mars arch

b.ys 7:1-10 'cat': circular embedding
  cat = arch
  b.ys 6:1-11 required by felix
  felix = cat
  b.ys 5:1-12 required by arch
  arch = felix
  b.ys 2:1-17 required by venus
  venus = mars arch
  b.ys 7:1-10 required by cat
  cat = arch
```

## Finite state automaton

`src/fsa.{cc,hh}`定義了finite state automaton的表示方式以及一些基本操作。用自然數表示狀態，鄰接表存儲邊，一條邊表示爲`pair<pair<long, long>, long>`即(標號區間,目標頂點)，$\epsilon$轉移表示爲標號區間$[-1,0)$。有序表存儲final狀態。
```c++
typedef pair<long, long> Label;
typedef pair<Label, long> Edge;

struct Fsa {
  long start;
  vector<long> finals; // sorted
  vector<vector<Edge>> adj; // sorted
  /* ...... */
};
```

`src/fsa_anno.{cc,hh}`定義了`FsaAnno`：
```c++
struct FsaAnno {
  bool deterministic;
  Fsa fsa;
  vector<vector<pair<Expr*, ExprTag>>> assoc;
  /* ...... */
};
```

`deterministic`標註是否爲DFA。編譯時，AST的每個節點都會對應一個automaton，AST對應了一棵automaton tree。Automaton tree中內部節點automaton的頂點有兩種來源：來自某個孩子automaton、新引入的。`assoc[i]`記錄了根automaton的頂點$i$之前從屬的所有automata以及所處的位置(start、final、inner)。

`assoc`信息有三個作用：

- 判斷哪些邊會觸發Action
- 在substring grammar中判斷一個頂點是否爲內部頂點
- 在recursive automaton簡化中識別`CallExpr`或`CollapseExpr`

### `DotExpr`: `.`

兩個頂點，start爲0，final爲1，邊$(0, (0, AB), 1)$代表任意codepoint都會從0轉移至1。

`assoc[0]`包含一個元素`{DotExpr*, ExprTag::start}`，表示`DotExpr`對應的automaton包含頂點0且0是start。
`assoc[1]`包含一個元素`{DotExpr*, ExprTag::final}`，表示`DotExpr`對應的automaton包含頂點1且1是final。

### `LiteralExpr`: `'literal'`

單雙引號的字串在AST中表示爲`LiteralExpr`。對於長爲$l$的字串，創建$l+1$個頂點。
`assoc[0]`包含一個元素`{LiteralExpr*, ExprTag::start}`，表示`LiteralExpr`對應的automaton包含頂點0且0是start。
`assoc[l]`包含一個元素`{LiteralExpr*, ExprTag::final}`，表示`LiteralExpr`對應的automaton包含頂點$l$且$l$是start。
其他`assoc[i]`包含一個元素`{LiteralExpr*, ExprTag::inner}`，表示`LiteralExpr`對應的automaton包含頂點$i$且$l$爲內部頂點，`ExprTag::inner`在substring grammar中用到，可以避免連邊時指向它。

### $\epsilon$-closure

輸入NFA頂點集，輸出其$\epsilon$-closure，以有序NFA頂點集表示。

使用breadth-first search，結束遍歷時隊列中所有插入過的頂點即爲結果。一個小技巧是結束後擦除頂點的訪問標記，可以避免完整初始化訪問標記。

```c++
  for (long i: src)
    vis[i] = true;
  REP(i, src.size()) {
    long u = src[i];
    for (auto& e: adj[u]) {
      if (-1 < e.first) break;
      if (! vis[e.second]) {
        vis[e.second] = true;
        src.push_back(e.second);
      }
    }
  }
  for (long i: src)
    vis[i] = false;
```

### 運算

Thompson's construction algorithm描述了幾種運算的構造方法，它產生的是normalized automaton，start和final均只有一個，且不存在指向start或從final出發的邊。並、Kleene star等運算均引入兩個頂點。這樣做的好處是每個頂點入度、出度不超過2，各運算時間複雜度爲常數。

爲了減少頂點數，我儘量讓結果爲standardized automaton：有多個final，不存在指向start的邊。

#### 交

構造兩個automata的product automaton，求NFA的product automaton會有大量$\epsilon$-closure計算操作，因此先把NFA轉成DFA再進行交運算。

修改`assoc`以標識每個頂點都在`IntersectExpr`標識的automaton中。

#### 並

引入新頂點`src`，添加新邊：
```
(src, -1, A.start)
(src, -1, B.start)
```

`src`成爲新的`start`，`A`、`B`頂點的final標記保留。這樣得到的是standardized automaton。

修改`assoc`以標識每個頂點都在`UnionExpr`標識的automaton中。

#### 差

與交類似，先轉成DFA再運算。

如果左側含有`CallExpr`或`CollapseExpr`，結果可能不正確。

修改`assoc`以標識每個頂點都在`DifferenceExpr`標識的automaton中。

#### Kleene star

引入兩個新頂點`src`和`sink`，添加新邊：
```
(src, -1, sink)
(src, -1, start)
for each final
  (final, -1, sink)
  (final, -1, start)
```

`src`成爲新的start，`sink`成爲唯一的final。這樣得到的是normalized automaton。若不存在從final出發的邊可以省掉`sink`，若不存在指向start的邊可以省掉`src`。

修改`assoc`以標識每個頂點都在`StarExpr`標識的automaton中。

#### Kleene plus

各final向start連$\epsilon$邊。

修改`assoc`以標識每個頂點都在`PlusExpr`標識的automaton中。

#### Question

引入兩個新頂點`src`和`sink`，建立新邊：
```
(src, -1, sink)
(src, -1, start)
```

`src`成爲新的start，給`sink`設置final標記。

修改`assoc`以標識每個頂點都在`QuestionExpr`標識的automaton中。

## 編譯

類似於reverse Polish calculator，每個Expr會編譯成一個automaton。

每個automaton都要轉成一個最小化的DFA，有三個步驟：

- Subset construction轉成DFA
- Hopcroft minimization合併Nerode equivalent states
- 去除co-inaccessible states即無法達到final的狀態

實踐中發現最小化每個部件比只最小化最終automaton要快很多。

## 含遞歸的產生式和簡化的recursive automaton

產生式中可能含有遞歸，比如直接遞歸`A = A b`或間接的`A = B b; B = A a`，如何用automaton描述呢？

之前fqj1994提出一個方法，對於
```
A = a B a | c
B = b A b | d
```

把產生式中遞歸引用的nonterminal用一個字符集外的terminal表示：
```
A = a "B" a | c
B = b "A" b | d
```

這裏假設`"B"`爲字符集外的terminal，表示爲含兩個頂點start和final的automaton，start向final連標號爲`"B"`的邊。`"A"`同理。

`A`、`B`兩個產生式對應的automata構造好之後，在`A`的automaton上添加新邊：
```
("B".start, -1, B.start)  # 對B產生式的調用
(B.final, -1, "B".final)  # B匹配完後返回到B被調用的地方
```

當`"B"`在多個產生式中出現或者一個產生式中出現多次時，就可能發生調用處和返回處不匹配，比如下面的C程序：
```c
void A()
{
  B();
  B();
}

void C()
{
  B();
}
```

執行A函數中第一個`B()`，返回時應該到第一個`B()`調用的下一條指令處。但是按照前文描述的連邊規則，返回地址可能是三個`B()`中任意一個的下一條指令。

偃師支持兩種調用，`EmbedExpr` `A = B`和`CollapseExpr `A = !B`。`EmbedExpr`把`B`的automaton嵌入到`A`的automaton`中，會產生依賴關係以防止`A`被嵌入到`B`中。`CollapseExpr`允許遞歸，按照前文的連邊規則構造automaton。

對於原始文法，找出循環依賴形成的強聯通分量，用`CollapseExpr`把環破開即得到了一種regular approximation。

TODO
考慮
```
S = !A !A
```

這種方式引入的頂點數很少。

我另外查閱了其他方案，比如Regular Approximation of CFLs: A Grammatical View，$k$個頂點、$l$條邊的強聯通分量會變成$4kl$條邊，不可接受。

## Substring grammar

我們的另一個需求是匹配SQL子串，即SQL的substring grammar。取suffix grammar的prefix grammar即得到substring grammar，根據對稱性，知道如何求suffix grammar後即可求prefix grammar。Suffix grammar構造方法大致是這樣：根據每個nonterminal $A$生成相應的$A'$，$A'$的產生式爲A的產生式去除proper prefix。
```
S = A B C
S' = A' B C | B' C | C'
```

不過很多suffix grammar都有歧義，無法用LR(1)，得訴諸Cocke-Younger-Kasami、Backtracking LR、Generalized LR、Earley parser等通用的parser。CYK、GLR Tomita、Binary Right Nulled GLR、Earley等都是$O(n^3)$的，$n$爲輸入長度，在我們的應用場景下不可接受。

另一方面，根據regular grammar求substring grammar很容易，構造原始文法的finite automaton，對每個頂點添加start和final標記即可。

## Action

給狀態轉移添加action。具體地，每個頂點有四類hook：

每個頂點所屬於若干互相嵌套的Expr，由`assoc`記錄。每個Expr可以有四類hook：entering、finishing、leaving、transiting。從頂點$u$轉移到頂點$v$時，若對於某個Expr $e$：

- $u\notin e, v\in e$，觸發$e$的entering hook
- $u\in e, v\notin e$，觸發$e$的leaving hook
- $u\in e, v\in finals(e)$，即$v$在$e$表示的automaton中是final頂點，觸發$e$的finishing hook
- $u\in e, v\in e$，觸發$e$的transiting hook

這部分代碼在`src/fsa_anno.{cc,hh}`實現。

## .ys文件語法

```
toplevel:
  stmt_list

stmt_list:
    %empty
  | '\n' stmt_list
  | stmt '\n' stmt_list

stmt:
    define_stmt
  | 'import' STRING_LITERAL 'as' IDENT
  | 'import' STRING_LITERAL
  | 'action' IDENT BRACED_CODE
  | 'c++' BRACED_CODE

define_stmt:
    IDENT '=' union_expr
  | IDENT ':' union_expr
  | IDENT ':' '|' union_expr
  | 'export' define_stmt
  | 'intact' define_stmt

union_expr:
    intersect_expr
  | union_expr '|' intersect_expr

intersect_expr:
    difference_expr
  | intersect_expr '&' difference_expr

difference_expr:
    concat_expr
  | difference_expr '-' concat_expr

concat_expr:
    unop_expr
  | concat_expr unop_expr

unop_expr:
    factor
  | '~' unop_expr

factor:
    'epsilon'
  | IDENT
  | IDENT '::' IDENT
  | '!' IDENT
  | '!' IDENT '::' IDENT
  | STRING_LITERAL
  | '.'
  | bracket
  | STRING_LITERAL '..' STRING_LITERAL
  | '(' union_expr ')'
  | repeat
  | factor '>' action
  | factor '@' action
  | factor '%' action
  | factor '$' action
  | factor '+'
  | factor '?'
  | factor '*'

repeat:
    factor '{' INTEGER ',' INTEGER '}'
  | factor '{' INTEGER ',' '}'
  | factor '{' INTEGER '}'
  | factor '{' ',' INTEGER '}'

action:
    IDENT
  | IDENT '::' IDENT
  | BRACED_CODE

bracket:
    '[' bracket_items ']'
  | '[' '^' bracket_items ']'

bracket_items:
    bracket_items CHAR '-' CHAR
  | bracket_items CHAR
  | %empty
```

## 頂層

源文件每行是一個定義、import、action定義、或C++代碼塊。

### import

```
import 'a.ys'
```

### action定義

定义一个带名字的action：
```
action foo {
  puts("foo");
}
```

之后可以用`> foo`等引用：
```
bar = 'a' > foo @ foo $ foo % foo > {puts("unnamed");}
```

### C++定義

## Contrib

`contrib/`下有Vim插件和Zsh配置文件。

代碼還比較亂……

此处似乎得有廣告，長亭科技<https://chaitin.cn>。

<del>聽說這個東西有望開源？</del>開源了，<https://github.com/MaskRay/yanshi>。
