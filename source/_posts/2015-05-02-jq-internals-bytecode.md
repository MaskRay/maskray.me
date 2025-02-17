layout: post
title: jq实现原理——字节码
author: MaskRay
tags: [jq,compiler]
mathjax: true
---

[jq](http://stedolan.github.io/jq/)所用的DSL是一门dataflow language，程序中编写的几乎所有结构都是filter，函数(比如`def foo(f) f;`)接受filter为参数，产生filter。jq程序描述了如何把各个filter组合起来产生更复杂的filter，在执行时把输入数据(若干JSON)变换得到输出(若干JSON)。

jq独特的地方在于它的控制结构是由if和backtracking实现的。由于变量赋值后不可修改，因此迭代无法通过跳转和可变状态实现。

jq把程序编译成字节码后解释执行。字节码是栈式的(参看后缀表达式计算器、Forth)，处理的值是JSON，执行时当前结果和临时变量等都存放在data stack上，并提供了很多指令用于处理data stack。下面用$s$表示data stack，左边是栈顶：

持续更新中

<!-- more -->

- `DUP`：$v\cdot s \rightarrow v\cdot v\cdot s$
- `DUP2`：$v_0\cdot v_1\cdot s \rightarrow v_1\cdot v_0\cdot v_1\cdot s$
- `POP`：$v\cdot s \rightarrow s$
- `LOADV(lvl,idx)`：$v\cdot s \rightarrow framevar(lvl,idx)\cdot s$
- `STOREV(lvl,idx)`：$v\cdot s \rightarrow s$，另外$framevar(lvl,idx) \leftarrow v$
- `JUMP(pc)`：跳转
- `JUMP_F(pc)`：$v\cdot s \rightarrow s$，若$v$为false或null则跳转

jq支持函数调用，还支持嵌套函数，下面是两个例子：
```
def range($x): range(0;$x);

def scan(re):
  match(re; "g")
  |  if (.captures|length > 0)
      then [ .captures | .[] | .string ]
      else .string
      end ;
```

Call stack中维护活动记录(见`execute.c`的`struct frame`)，存储的信息包括：函数第一条字节码地址`bc`、下一个环境`env`(下文会说明)、返回地址`retaddr`、返回caller后恢复的栈地址`retdata`、闭包和局部变量`entries`(下文会说明)。
Data stack和call stack共用一块存储空间。解释器的核心是`struct jq_state`，两个偏移量`stk_top`和`curr_frame`分别指向data stack和call stack的栈顶。栈中的每个元素带有指向后继的指针，call stack中元素的后继是离它最近的下一个call stack元素，data stack中元素的后继是离它最近的下一个data stack元素，因此call stack和data stack是不相交的两条链。栈元素虽然没有标签区分call stack元素还是data stack元素，但可以从表示栈顶的`stk_top`或`curr_frame`出发，沿着后继指针下溯找出所有特定栈的元素。

jq提供了很多控制结构，这里列出一些重要的：if-else、`[]`、foreach、reduce、try-catch、`|`(串联)、`,`(`gen_both`)。<http://stedolan.github.io/jq/manual/>比较旧，可以查阅<https://github.com/stedolan/jq/blob/master/docs/content/3.manual/manual.yml>。

## if-else

`if-else`的翻译方案参看`gen_cond`：
```
DUP           # <cond>会对栈顶元素进行处理，得到true/false，而then/else部分需要用到栈顶原值，因此DUP
<cond>
JUMP_F .L0    # 栈顶为false或null则跳转
<then>
.L0:
<else>
```

## `[]`

`[]`翻译成一条字节码：`EACH`，例如：
```
% jq --debug-dump-disasm '.[]'
0000 TOP
0001 EACH
0002 RET
```

其中`--debug-dump-disasm`选项用于打印字节码。

你一定好奇这条`EACH`指令怎么能发挥这么大魔力的，比如：
```
% jq  '.[]' <<< '[1,2,3]'
1
2
3
```

翻译方案完全不像循环或递归！我也觉得这实在是太神奇了……继续看下去吧。

先引入一个概念：choice point stack，这个不是jq的术语，但我觉得和Prolog的Warren Abstract Machine里的choice point概念有相似处，因此捏造了这个名称。在jq里它和call/data stack共用存储空间，`struct jq_state`里用`fork_top`指向choice point stack的栈顶，它也以后继指针形成一条链，三个栈的链互不重叠。
Choice point stack的元素是choice point，类似于线程，choice point stack管理各个choice point，但同一时刻只有一个choice point在执行，从栈中弹出后(退出)也无法再恢复。Choice point的类型是`struct forkpoint`，包含如下字段：

- `stack_ptr saved_data_stack`：choice point执行时使用的`stk_top`(data stack栈顶指针)
- `stack_ptr saved_curr_frame`：choice point执行时使用的`curr_frame`(call stack栈顶指针)
- `int path_len, subexp_nest`：TODO
- `uint16_t* return_address`：choice point执行时使用的program counter值

Choice point的生命期从被创建出来开始算起，执行`BACKTRACK`或顶层的`RET`(顶层的`RET`用于返回输出)时结束。`BACKTRACE`的作用是从choice point stack上弹出栈顶替代当前的choice point。jq没有常规语言中的循环，用choice point机制模拟出循环的形式。

题外话：`struct forkpoint`里的这些字段可以看作保存的执行环境。不谈continuation，常规编程语言中用到环境保存的最熟悉的概念是异常处理，catch根据异常类型创建用于处理异常的活动记录，try则会unwind栈，上溯找到匹配的最近的异常处理活跃记录。

`EACH`执行时会区分首次执行(`case EACH:`。非backtracking模式执行时)和非首次执行(`case ON_BACKTRACK(EACH)`。Backtracking模式执行时)。
首次执行要求栈顶值为待迭代的容器，它会在data stack上压入迭代位置值-1，然后fall through到非首次执行的代码。
非首次执行时，从data stack上弹出迭代位置值和容器，获取容器中位置值+1处的元素，压入data stack。之后在choice point stack上创建一个choice point，其program counter是当前指令(`EACH`)。

字节码解释器(`execute.c`)中`EACH`的解释方案是让它第$i$次被某个choice point执行时把容器的第$i$个元素压入栈顶。下面看一个具体例子，`--debug-trace`可以产生指令trace：
```
% ./jq --debug-trace '.[]' <<< [0,1,2]
0000 TOP
0001 EACH       [0,1,2] (1)
0002 RET        0
0
0002 RET                <backtracking>
0001 EACH               <backtracking>
0002 RET        1
1
0002 RET                <backtracking>
0001 EACH               <backtracking>
0002 RET        2
2
0002 RET                <backtracking>
```

`EACH`首次执行时创建了一个choice point(其中记录了当前已经处理过的位置0)，之后执行`0002 RET`(顶层`RET`)，`jq_next`返回到`main.c` `process`函数(驱动)的`while (jv_is_valid(result = jq_next(jq))) {`，在stdout打印0后`jq_next`再次被执行！`jq_next`非首次执行时会立刻进入backtracking模式(`execute.c`中的`int backtracking = !jq->initial_execution;`)，而choice point在经过`ON_BACKTRACK(RET)`中转后回到了`EACH`创建的记录！因此`EACH`被再次执行，几乎重复之前的步骤：压入新choice point和产生新一轮迭代的值。

## `,`

参见`compile.c`中定义的`gen_both`。

看一个例子：
```
% jq --debug-dump-disasm '.,.'
0000 TOP
0001 FORK 0005
0003 JUMP 0005
0005 RET
```

其中`JUMP 0005`是冗余的，在这里`,`就对应一条`FORK`。

`FORK`的名称应该源自Unix进程拷贝函数`fork`，作用是根据参数program counter创建choice point(其他参数如data/call stack等都继承当前choice point)，而当前choice point则继续执行下一条指令。
和前面`EACH`的情况类似，当前choice point产生输出退出后被驱动(`main.c`的`process`函数)再次执行，保存的choice point恢复并产生下一个输出。

现在可以理解`empty`了，`jq -n '1,empty,3'`会输出两个值：1和3。两个逗号会用`FORK`创建两个choice point，而`empty`会翻译成`BACKTRACK`，即从choice point stack上弹出choice point替代当前choice point。

## `foreach`

参见`compile.c`中定义的`gen_foreach`。

`foreach EXP as $var (INIT; UPDATE; EXTRACT)`的翻译方案是：
```
DUP
<init>
STOREV <var>

FORK .L1

DUPN
<source>
LOADVN <var>
<update>
DUP
STOREV <var>
<extract>
JUMP .L0

.L1:
BACKTRACK

.L0:
```

`DUPN`和`LOADVN`涉及[issue #618](https://github.com/stedolan/jq/issues/618)，理解的时候可以分别看作`DUP`和`LOADV`。

## 垃圾收集

引用計數

TODO


<!--
考察对于第一个元素的字节码执行路径：`<init>...FORK .L1...<source>...<update>...<extract>...JUMP .L0`，后面就是其他指令翻译得到的字节码了，对于一个执行绪来说，`foreach`的`<init>`、`<update>`、`<extract>`只执行了一次！那么它如何实现对每个元素执行一次的效果呢？
答案是让这段`foreach`生成的字节码执行多次，具体的方法则是产生多个执行绪，每个都执行一次`foreach`生成的字节码。
-->
