---
layout: post
title: ML编译器Caml Featherweight——编译
author: MaskRay
tags: [compiler, ocaml, ml]
---

## 编译、链接和运行

### 流程概览

整个编译、链接和运行过程拆分为若干阶段：

-   解析。包括词法分析和语法分析，使用工具ocamllex和menhir。

-   类型检查。检查程序是否合法，类型推导也在这一步进行。

-   把语法树翻译成一种中间表示：扩充的$\lambda-calculus$

-   把扩充的$\lambda-calculus$编译为Zinc抽象机器字节码。每个源文件会生成一个目标文件。

-   链接各目标文件得到可被运行时系统执行的字节码文件。

-   运行时系统解释执行。

下面分别介绍各个阶段。

### 解析

使用ocamllex进行词法分析，menhir进行语法分析得到抽象语法树。

多数yacc实现都采用了操作符优先级来解决移入/规约冲突，在实现中我们碰到一些典型问题并得以解决。

#### 词法分析错误信息

#### 解析逗号分割的列表

考虑表示表达式的非终结符`expr`，它的其中两个产生式是逗号分割的表达式和分号分割的表达式，约定逗号(`COMMA`)优先级高于分号(`SEMI`)，可以如下实现：

    expr:
      | ...
      | expr_comma_list %prec below_COMMA { Pexpr_tuple($1) }
      | expr SEMI expr { Pexpr_seq($1,$3) }
      | ...

    expr_comma_list:
      | expr COMMA expr_comma_list { $1 :: $3 }
      | expr COMMA expr { [$1; $3] }

其中`%prec below_COMMA`是指定第一条产生式的优先级，从而在栈为`expr_comma_list`且向前看符号为`COMMA`时，menhir选择移入而不是规约`expr_comma_list -> expr`。

但这种语法描述还有一个问题。当栈内当前为`expr COMMA expr_comma_list`且向前看符号为`SEMI`时，规约`expr_comma_list -> expr`和规约`expr COMMA expr_comma_list -> expr`的优先级均大于`SEMI`，因此无法确定采取哪一个规约，产生规约/规约冲突。

解决方案是用右递归重新描述`expr_comma_list`：

    expr_comma_list:
      | expr_comma_list COMMA expr { $3 :: $1 }
      | expr COMMA expr { [$3; $1] }

在使用`expr_comma_list`要注意把列表反转。考虑到函数式语言中在列表头部添加元素比较容易，描述文法时通常用左递归而非右递归，但在上述情形下就不得已采用右递归。在更为复杂的OCaml的`parsing/parser.mly`中也能看到一些这样的例子。

#### 解析标识符，区分构造器和变量

Caml Light的语法分析文件中把实现文件划分为用双分号分割的多个phrase，解析完一个phrase后立即进行类型检查、代码生成等工作，并导入全局的值、类型、构造器信息。在解析模式匹配的地方遇到一个标识符时，通过查询之前是否定义过该标识符的构造器来区分该标识符是构造器还是变量。Caml Light把部分变量是否定义的工作糅合到了解析阶段。

我们希望能完整解析源文件后再进行变量定义判断的工作，因此借鉴了OCaml的解析器实现，区分大小写两种标识符，大写视为构造器，小写视为变量。

在这里也能一窥语言实现的考量，了解为什么很多支持模式匹配的语言规定构造器使用大写、变量使用小写。

这里也能看到为了语言的一致性，`false`和`true`应该看作构造器，使用大写，实现中如果要摈弃Caml Light中解析阶段判断是否定义的做法，比较好的方式是在词法分析中把`false`和`true`也作为词法单元。

### 类型检查

解析得到抽象语法树后，进行类型检查判断程序是否合法。项目中使用Damas-Hindley-Milner类型系统的algorithm W进行类型推导。

Algorithm W中使用unification来判断两个类型是否可能相等，为此需要适配两个类型的形状。比如如果要unify函数类型`int -> a`和另一个类型`b -> char`，需要把类型变量`b`变为具体类型`int`、类型变量`b`变为具体类型`int`。

实现unification时有两种方法：

-   使用一个类型映射表，记录各个类型变量被映射为什么类型(具体类型、其他类型变量、或generic类型)

-   每个类型变量维护一个可修改的域，表示不相交的集合，类型变量可以是尚未确定与其他类型的关系(自身为一个集合)，或者被确定与某些类型相同(与它们在一个集合中)。这里的可变域表示为并查集算法的父节点。

使用可变域性能较高，因此我们的实现中也采用了这一方法。

#### 类型的generalization

Damas-Hindley-Milner类型系统中的let-polymorphism阶段，需要把尚未和外部类型联系起来的类型变量generalize成generic类型。

原始的实现方式是遍历类型(表示为一棵树)，访问其中所有类型变量，判断是否和外部类型unify过，如果没有则generalize。<http://okmij.org/ftp/ML/generalization.html>中提到的Rémy’s level-based generalization采取了一种剪枝策略，也是Caml Light实现所采用的。我们的项目也实现了这一方法。另外文章中还提到了一种惰性实现，可以优化occur check，并加速整个类型推导过程的时间复杂度到近似线性。我们实现了该方法，但考虑到多数情况下类型树不会很大，occur check和遍历类型树的代价不会很大，并且惰性方法要引入额外的空间需求，以及实现的复杂性，未在项目中引入。

#### 实例化generic类型

Caml Light在实例化generic类型时采用了一种技巧(`cl75/src/compiler/types.ml`)，把实例化分成两个阶段：`copy_type`和`cleanup_type`。

`copy_type`把原来的类型复制了一份，修改了类型树中generic类型变量的可变域，指向一个该变量的实例(不再为generic)，返回的副本与原本的类型共享那些非generic的部分。当原来的类型被复制第二遍的时候，新的副本会与第一份副本共用新生成的非generic类型变量。因此对于相同的generic类型变量，它的各个副本都会是相同的类型实例。

之后经过`cleanup_type`，原来的类型被还原成`copy_type`前的形态。

`copy_type`的特性在产生相关联的类型实例时很方便，比如两棵类型树共享一个generic的类型变量。那么实例化时希望得到的两个类型树实例的对应类型实例也是相同的。可以这样做：先分别对两棵树进行`copy_type`，再分别`cleanup_type`，得到的两个副本保证共用了相同的类型实例。

### 扩充的$\lambda$-calculus

ML的语法比较复杂，在编译到字节码需要进行一些简化，为此Caml Light采用了一个扩充的$\lambda$-calculus作为中间表示，我们也实现了类似的、但更加简化的扩充$\lambda$-calculus。

原始的无类型$\lambda$-calculus中唯一的值类型是$\lambda$抽象，另外也只有变量和应用两种项形式。经过de-Bruijn index的变换后，可以引入环境这一元素(若干嵌套的$\lambda$抽象中的值的列表)，变量指向环境中特定位置项，因此可以用一个索引值替代，由此我们就把`Lvar of string`换成了`Lvar of int`，这里之所以能做这样的变换还得感谢静态作用域。

比如在`fun x -> fun y -> x`中，`x`和`y`都在环境中，`x`可以用de Bruijn index 1代替。

在ML中，值类型还可以是像`int`这样的基本类型，因此我们需要一个额外的构造器`Lconst of constant`。原始的无类型$\lambda$-calculus也无法表示加法这样的运算，只能采用Church encoding。在ML这样的实际语言中需要有办法表示这些运算，以及其他很多操作如数组取元素、比较两数、根据构造器和参数创建代数数据类型等，因此我们添加了构造器`Lprim of prim * lambda list`，可以看作`prim`表示的原语函数应用在一些参数上。

抛开let-polymorphism，let可以等价变形为$\lambda$抽象和应用的一个简单组合(`let x=... in y`变换为`(fun x -> y) (...)`)。而在类型检查通过后，如果把所有值看作是boxed类型，并且使用无类型的$\lambda$-calculus，那么我们可以不忽略let-polymorphism的影响。但实现为函数应用会有一定的性能开销，因此尽管不是必须有的，我们还是提供了构造器`Llet of lambda list * lambda`。

利用call-by-value的求值策略顺序，可以用$\lambda$抽象和应用实现表示顺序执行的`;`(`x; y`变换为`(fun _ -> y) x`)，但为了性能考虑设立了`Lsequence of lambda * lambda`构造器，它会先执行第一项，再执行第二项。通过使用这种方式，实现了对C中语句和表达式的统一。

模式匹配引入了更多的复杂性。当没有模式匹配时，环境中的项(函数的形式参数)只有一种引用方式，即视为一个整体引用。但涉及模式匹配后，考虑`fun ((x,y) as a) -> `，作为整体的参数可以用`a`引用，但模式匹配又引入了构成二元组`a`的两个成分`x`、`y`，它们也可以被函数体引用，所以该如何表示它们呢？另外代数数据类型的构造器可以有多个参数，也可以在模式匹配中出现，需要提供一种办法引用它们的参数。

就像代数数据类型的描述那样(元组可以看成是提供了语法支持的特殊代数数据类型)，参数可以看成是它们的子域，我们使用原语`Pfield of int`来引用某个子域。用于de Bruijn index的环境不再是简单变量的列表，列表中的每一项不仅要提供引用参数自身的方法，也要提供方法引用出现在模式匹配中的所有变量。因此我们把环境表示为一个列表的列表，每个内层列表代替的原来的简单参数，其中包含所有变量的引用方式。

项目源文件`front.ml`实现了抽象语法树到扩充的$\lambda$-calculus的翻译。很多节点的翻译很直观：

``` {.ocaml}
    | Pexpr_apply(e,es) ->
        Lapply(go e, List.map go es)
    | Pexpr_array es ->
        Lprim(Pmakeblock(0,0), List.map go es)
    | Pexpr_constant c ->
        Lconst c
    | Pexpr_sequence(e1,e2) ->
        Lsequence(go e1, go e2)
    | Pexpr_tuple es ->
        Lprim(Pmakeblock(1,0), List.map go es)
```

比如代表函数应用的`Pexpr_apply`，代表字面量的`Pexpr_constant`等。元组稍复杂一些，可以看作是使用原语`Pmakeblock`创建了一个新的block，其各个域是元组的各个成分。

构造器较为复杂，需要判断在代数数据类型定义中该构造器是否带参数(`Constr_constant`或`Constr_regular`，以及在表达式中是否带参数，选择翻译成一个创建新block的原语，或者是个$\lambda$抽象：接受参数以创建block。

``` {.ocaml}
    | Pexpr_constr(id,arg) ->
        let cd = find_constr_desc id in
        begin match arg with
        | None ->
            begin match cd.info.cs_kind with
            | Constr_constant ->
                Lprim(Pmakeblock cd.info.cs_tag, [])
            | _ ->
                Labstract(Lprim(Pmakeblock cd.info.cs_tag, [Lvar 0]))
            end
        | Some arg ->
            Lprim(Pmakeblock cd.info.cs_tag, [go arg])
        end
```

#### 全局值

处于顶层的`let`会产生一个全局值(如基本类型值、函数等)，可以被之后的代码引用。用通常的静态作用域观点看，这样产生的值不在之后代码的作用域内。当然我们可以把在顶层并列放置的`let`绑定视为是嵌套的，但这种模型无法解决多文件全局值的引用问题，因此不可行。

解决这个问题需要引入全局值的概念，设置原语`Pgetglobal of long_ident`和`Psetglobal of long_ident`用于表示读取和写入全局值。

#### 翻译模式匹配

模式匹配是最复杂的部分，允许匹配多个代数数据类型复合的结构，混合字面量、变量和构造器，各个匹配项还有顺序要求，因为如果多个模式都能匹配，那么根据语义应该选取最靠前的匹配项。@slpj的Chapter 5，Philip Wadler提供了详细的描述。@Zinc的5.2.4 Compiling pattern matching提供了Caml Light中的实现方法。在项目中我们又进行了进一步的简化。

对于复杂的模式匹配，比如：

``` {.ocaml}
  match false, false with
  | false, true -> ()
  | false, false -> ()
```

设想一个按顺序匹配元组的各个项的扩充$\lambda$-calculus表示，第一个匹配项的`false`匹配成功后，`true`会匹配失败，此时应该跳转到第二个匹配项。如何实现这样的逻辑呢？

当涉及多个匹配项时，翻译器会把它们组织成`Lstaticcatch(p1,Lstaticcatch(p2,Lstaticcatch(...)))`，`p1`、`p2`是各模式匹配项翻译得到的代码，执行`p1`时，若发现后续的模式匹配失败了，就会执行`Lstaticraise`。其效果相当于抛出了一个异常，该异常会被嵌套它的最内层的`Lstaticcatch`捕获，尝试执行作为替代的第二条指令。这一机制可以看成是一个静态作用域的异常处理，因此使用了这样的名字：`Lstaticcatch`和`Lstaticraise`。

`if`可以看作是模式匹配的特例，即匹配`true`和`false`，但为了性能考虑，设立了`Lif`这一构造器。

#### 翻译异常处理

`Lcatch of lambda * lambda`的第一个参数是`try`的代码体，如果抛出异常则把控制权交给模式匹配部分，即第二个参数。

#### 扩充$\lambda$-calculus定义

`lambda.ml`中的定义：

``` {.ocaml}
type lambda =
  | Labstract of lambda
  | Lapply of lambda * lambda list
  | Lcatch of lambda * lambda
  | Lcond of lambda * (constant * lambda) list
  | Lconst of constant
  | Lif of lambda * lambda * lambda
  | Llet of lambda list * lambda
  | Lletrec of lambda list * lambda
  | Lprim of prim * lambda list
  | Lsequence of lambda * lambda
  | Lstaticcatch of lambda * lambda
  | Lstaticraise
  | Lswitch of int * lambda * (constr_tag * lambda) list
  | Lvar of int
```

### 扩充$\lambda$-calculus翻译成Zinc抽象机器的linear code

接下来把中间表示翻译成Zinc抽象机器的linear code。字节码相当于机器码，而linear code就相当于汇编代码。Linear code的定义在文件`instruction.ml`中：

``` {.ocaml}
type zinc_instruction =
  | Kaccess of int
  | Kapply
  | Kbranch of int
  | Kbranchif of int
  | Kbranchifnot of int
  | Kcur of int
  | Kdummy of int
  | Kendlet of int
  | Kgetglobal of long_ident
  | Kgrab
  | Klabel of int
  | Klet
  | Kmakeblock of constr_tag * int
  | Kpoptrap
  | Kprim of prim
  | Kpush
  | Kpushmark
  | Kpushtrap of int
  | Kquote of struct_constant
  | Kreturn
  | Ksetglobal of long_ident
  | Kswitch of int array
  | Ktermapply
  | Ktest of bool_test * int
  | Kupdate of int
```

`type zinc_instruction`的多数构造器都和字节码有直观的对应关系，比如构造器`Kaccess of int`翻译成字节码时，会额外输出一个`uint8_t`类型的操作数。表示的标号构造器`Klabel of int`不会生成字节码，但能作为一个标签供其他指令引用。

中间表示翻译成linear code的实现在文件`back.ml`中，采用了compile with continuation的编译方法。其中的主要函数是`compile_lambda`中的`compile_expr : int -> lambda -> instruction list -> instruction list`，第一个参数表示模式匹配失败后需要跳转到的代码，第二个参数是待编译的扩充$\lambda$-calculus，第三个参数代表该指令执行完后需要执行的continuation，返回Zinc指令的列表(需要包含continuation)。

很多结构的翻译是直观的：

``` {.ocaml}
      | Lvar i ->
          Kaccess i :: cont
      | Lconst c ->
          Kquote c :: cont
      | Lprim(Pdummy n, []) ->
          Kdummy n::cont
      | Lprim(Pgetglobal id, []) ->
          Kgetglobal id::cont
```

模式匹配失败后会执行`Lstaticraise`，翻译的方式是跳转到嵌套该代码的最靠近的`Lstaticcatch`。

``` {.ocaml}
      | Lstaticraise ->
          Kbranch staticraise::cont
```

有部分结构比如`Lif`，代码执行路径可能有多条，需要引入一些分支指令。

`Lswitch`根据参数(必须是block)的tag值及跳转表进行跳转。

`Lcatch`的翻译方案是压入一个异常处理栈帧，执行`try`代码体时若发生异常，控制权会交给最近的异常处理栈帧，如果它匹配失败就把控制权交给外层的异常处理栈帧。若未发生异常则弹出异常处理栈帧。

#### 全局值

扩充$\lambda$-calculus中对全局值的处理可以直接翻译成相应的抽象机器指令，同样需要用于处理读取和写入全局值的指令。读取和写入相同名称的指令应该访问同一个内存地址。为了性能的考虑，我们不希望在运行时维护一个名称到具体内存地址的映射表。再考虑到一个文件访问另一个文件定义的全局值，我们在生成代码后还需要一个过程把这些名字解析到合适的槽位，因此需要下一个阶段——链接。

``` {.ocaml}
      | Lprim(Pgetglobal id, []) ->
          Kgetglobal id::cont
      | Lprim(Psetglobal id, [e]) ->
          c_expr e (Ksetglobal id::cont)
```

### Linear code翻译为字节码，生成目标文件

这一阶段的实现在文件`emit.ml`中。我们使用了一个简单的翻译方案。每个指令拆分为操作码和参数两部分，操作码占据一个字节，参数则有多种形式：`uint8_t`、`int16_t`、标签等。

因为标签定义和引用都是在当前源文件里局部的，生成代码时可以把这些标签引用都解析为字节码中的相对地址。

### 链接器

正如之前所提到的，这一过程并非必须，但基于性能的考虑需要在编译流程中引入这一阶段。读取各个源文件编译得到的目标文件(需要重定位)，梳理其中全局值定义和引用的关系，把各条`Ksetglobal`、`Kgetglobal`指令解析到正确的槽位是这一阶段的主要目标。

在我们的实现中这一部分的代码在`ld.ml`中，`ld`是传统的链接器的名字，我们借用了这一名字来贴切地表示这段代码需要完成的任务。

`ld.ml`使用了一个两遍扫描过程，先扫描各目标文件得到所有的全局值定义，再扫描各目标文件，解析其中的全局值定义和引用，把指令写入可执行文件中。该文件实际上还处理了原语操作。

### 运行时系统和字节码解释器

运行时系统和字节码解释器用C实现，在目录`runtime/`中。

#### 初始化共享的size域为0的构造器

因为构造器的域不可变，且size域为0的block经常被使用，我们可以把tag值为0到255的256个block缓存起来，供所有size域为0的block共享。Caml Light的`cl75/src/runtime/mlvalues.h`中声明了`extern header_t first_atoms[];`，`()`、`true`等就用这些`first_atoms`元素的指针表示。我们的运行时也采用了这一技巧。

#### 全局值初始化

字节码文件中有一些`float`、`string`常量，执行时需要一个加载过程，把这些常量分配为block供代码使用。另外还有一些`Psetglobal`需要引用的槽位，初始化为整数0(bit pattern是1)。

#### 指令解码和派发

解释器的主要部分是`runtime/main.c`中的一个取指令并解释的循环。常规解释器实现方法是在循环里放置一个switch实现指令的解码和派发：

``` {.c}
  for(;;) {
    switch (*pc++) {
      case ACCESS:
        ...
        break;
      case ADDFLOAT:
        ...
        break;
      ...
    }
  }
```

但这样做，指令一条抽象机器指令会引入至少两条CPU的跳转指令，即break引入的跳转到switch处的指令和switch跳到某一个标签处的指令。

我们可以让break实现解码下一条指令并跳转到相应标签处的功能，这样就可以节省一条跳转指令。GCC提供了computed goto特性，可以获取可以获取标签所指代码的地址(`void*`类型)，从而可以手工实现一个跳转表：

``` {.c}
  void *jumptable[] = {&&ACCESS, &&ADDFLOAT, ...};
  goto *jumptable[*pc++];

  ACCESS:
    ...
    goto *jumptable[*pc++];
  ADDFLOAT:
    ...
    goto *jumptable[*pc++];
  ...
```

#### 垃圾收集

Caml Light使用的垃圾收集算法可以参考@Sestoft94，结合了stop-and-copy和分代mark-sweep。我们的实现则使用了一个非常简单的方法，基于Schorr-Waite graph marking算法。该算法不使用栈，只需要在节点上分配极少量信息($\lceil\log_2{n+1}\rceil$ bits，其中$n$是该节点的指针域数目)。在一些资料中被称为pointer reversal。基本思想是在节点信息里维护一个域$s$，表示遍历过程中的访问次数，并把遍历过程中祖先节点下溯的指针翻转，再设置一个表示当前节点父节点的指针$p$以把祖先节点组织为一个链。
