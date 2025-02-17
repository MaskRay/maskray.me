layout: post
title: Command line processing in LLVM
author: MaskRay
tags: [llvm]
---

Updated in 2023-05.

There are two libraries for processing command line options in LLVM.

## `llvm/Support/ComandLine.h`

See <https://llvm.org/docs/CommandLine.html> for documentation.

Global variables (mostly `llvm::cl::opt<type>`, some `llvm::cl::list<type>`) are most common to represent command-line options.
The `llvm::cl::opt` constructor registers this command line option in a global registry.
The program calls `llvm::cl::ParseCommandLineOptions(argc, argv, ...)` in `main` to parse the command line options.
`opt` supports various integer types, `bool`, `std::string`, etc. Defining some specialization can support support custom class/enum types.

<!-- more -->

```cpp
static cl::OptionCategory cat("split-file Options");

static cl::opt<std::string> input(cl::Positional, cl::desc("filename"),
                                  cl::cat(cat));

static cl::opt<std::string> output(cl::Positional, cl::desc("directory"),
                                   cl::value_desc("directory"), cl::cat(cat));

static cl::opt<bool> noLeadingLines("no-leading-lines",
                                    cl::desc("Don't preserve line numbers"),
                                    cl::cat(cat));

int main(int argc, const char **argv) {
  cl::ParseCommandLineOptions(argc, argv, ...);
}
```

Beside functionality options, there are many internal command line options in LLVM.

* Having an option to select different code paths. Usually introduced when one feature is under development and is considered experimental.
* The functionality has been stable for a period of time, and the default value has been changed to true. In some cases, users who find a regression can set the option to false as a workaround.
* Provide more input to a pass for testing.

The `llvm::cl` library is easy to use. Adding a new option only requires adding a variable to a source file.
The library provides some icing on the cake, e.g. recommending options with similar spellings in case of an incorrect spelling.
But the customizability is poor. For example:

* A `llvm::cl::opt<bool>` option accepts various input methods such as `-v 0 -v=0 -v false -v=false -v=False`
* It is inconvenient to support both `--long` and `--no-long` at the same time. Occasionally, the workaround is to set a variable for `--no-long`. If you want to deal with two options override each other, you must determine the relative position of the two options in the command line

User-oriented external tools often have such customization requirements. The style of GNU `getopt_long` is `--long`.
`-long` and `--long` can be mixed in `llvm/Support/ComandLine.h`, and mandatory `--` is not supported for a long time.

LLVM binary utilities (llvm-nm, llvm-objdump, llvm-readelf, etc.) in order to replace GNU binutils,
Need to provide grouped short options in the style of POSIX shell utilities (`-ab` means `-a -b`).
This feature is not supported for a long time, which troubles users who want to migrate to LLVM binary utilities.

In addition, `llvm::cl::opt` is a singleton, and local variables can also be defined to dynamically increase options, but this usage is rare (llvm-readobj and llvm-cov).
There is also a very peculiar usage. The legacy pass manager in the opt tool automatically obtains the pass name list and registers a large number of global options.

To prevent errors, `llvm::cl::opt` does not support defining the same option multiple times. If you link both shared object and archive LLVM libraries at the same time, a classic error will be triggered:

```
: CommandLine Error: Option'help-list' registered more than once!
LLVM ERROR: inconsistency in registered CommandLine options
```

If you want to set the value of the `llvm::cl::opt` variable in Clang, you can use `-mllvm -option=value`. Use ld.lld/LLVMgold.so Full/Thin LTO to set these option values, use `-plugin-opt=-option=value` (ld.lld can also use `-mllvm`).

## `llvm/Option/OptTable.h`

This was originally developed for Clang. It was later moved to llvm and adopted by many components such as llvm-objcopy, lld, and most binutils counterparts (llvm-symbolizer, llvm-objdump, etc).
OptTable uses a domain-specific language (TableGen) to describe command line options and generate a parser.
The parsed options are organized into an object, and each option is represented by an integer.
It is easy to check whether a pair of boolean options with variable default values (`--demangle --no-demangle`) are in effect: `Args.hasFlag(OPT_demangle, OPT_no_demangle, !IsAddr2Line)`.

Here is an example TableGen file:

```tablegen
multiclass B<string name, string help1, string help2> {
  def NAME: Flag<["--", "-"], name>, HelpText<help1>;
  def no_ # NAME: Flag<["--", "-"], "no-" # name>, HelpText<help2>;
}

multiclass Eq<string name, string help> {
  def NAME #_EQ: Joined<["--", "-"], name #"=">,
                  HelpText<help>;
  def: Separate<["--", "-"], name>, Alias<!cast<Joined>(NAME #_EQ)>;
}

defm debug_file_directory: Eq<"debug-file-directory", "Path to directory where to look for debug files">, MetaVarName<"<dir>">;
defm default_arch: Eq<"default-arch", "Default architecture (for multi-arch objects)">;
defm demangle: B<"demangle", "Demangle function names", "Don't demangle function names">;
def functions: F<"functions", "Print function name for a given address">;
```

In the C++ source file, one writes:

```cpp
  opt::InputArgList Args = parseOptions(argc, argv, IsAddr2Line, Saver, Tbl);

  LLVMSymbolizer::Options Opts;
  ...
  Opts.DebugFileDirectory = Args.getAllArgValues(OPT_debug_file_directory_EQ);
  Opts.DefaultArch = Args.getLastArgValue(OPT_default_arch_EQ).str();
  Opts.Demangle = Args.hasFlag(OPT_demangle, OPT_no_demangle, !IsAddr2Line);
  Opts.DWPName = Args.getLastArgValue(OPT_dwp_EQ).str();
```

### Grouped short options

Note that GCC's command line options do not support grouped short options, so Clang does not implement it.
Some binutils counterparts need the support, and I added grouped short options (D83639) in July 2020.

Anecdote: LLD uses this library to parse command-line options. GNU ld actually supports grouped short options, for example, `ld.bfd -vvv` means `-v -v -v`. I suggested that GNU ld actually supports many `-long` style options, and supporting grouped short options can cause confusion.

```
% touch an ommand':)'
% ld.bfd -you -can -ofcourse -use -this -Long -command -Line':)'
:)
```

binutils 2.36 is expected to deprecate grouped short options :)

### Target-specific options

In Clang, `clang/include/clang/Driver/Options.td` declares both generic and target-specific options.

On the plus side, if a useful feature is machine-specific in GCC, it is easy to implement it as a target-agnostic option in Clang.

On the down side, if it is an inherently target-specific option, it is easy to forget to report an error for other targets.
There will usually be a `-Wunused-command-line-argument` warning, but a warning may not be good enough.

For example, GCC's powerpc port doesn't support `-march=`, but Clang incorrectly parsed and ignored it, resulting in a `-Wunused-command-line-argument` warning.
<https://reviews.llvm.org/D145141> changed the warning to an error.

To prevent the aforementioned issues, when dealing with target-specific options in `clang/include/clang/Driver/Options.td`, we should add the ability to annotate compatible `clang::driver::ToolChain`.

### Comparison with `getopt_long`

Many users of `getopt_long` use a switch and a large number of cases to process command line options, and it is easy to create various position dependent behaviors.
For large build systems, sometimes it is not clear where the compiler/linker options are added, and some position dependent behaviors are quite annoying.

The `Args.getLastArgValue(..., ...)` pattern widely used in Clang has a limitation.
For options that take a value, we typically verify just the last option, ignoring previous options.
For example, invalid option values non-last options in `clang -ftls-model=xxx -ftls-model=initial-exec` and `clang -mstack-protector-guard=xxx -mstack-protector-guard=global` cannot be detected.

# 中文版

題外話：不知不覺，達成了llvm-project 1900 commits的成就。

LLVM中命令行選項的處理有兩個庫。

## `llvm/Support/ComandLine.h`

文檔參見<https://llvm.org/docs/CommandLine.html>

簡單來說，用全局變量(`llvm::cl::opt<type> var`最常見，也有`llvm::cl::list`等)表示命令行選項。`opt`的構造函數會在一個全局的registry中註冊這個命令行選項。
在`main`中調用`llvm::cl::ParseCommandLineOptions(argc, argv, ...)`解析命令行。
`opt`支持很多類型，如各種integer types、bool、std::string等，還支持自定義enum類型。

<!-- more -->

```cpp
static cl::OptionCategory cat("split-file Options");

static cl::opt<std::string> input(cl::Positional, cl::desc("filename"),
                                  cl::cat(cat));

static cl::opt<std::string> output(cl::Positional, cl::desc("directory"),
                                   cl::value_desc("directory"), cl::cat(cat));

static cl::opt<bool> noLeadingLines("no-leading-lines",
                                    cl::desc("Don't preserve line numbers"),
                                    cl::cat(cat));

int main(int argc, const char **argv) {
  cl::ParseCommandLineOptions(argc, argv, ...);
}
```

LLVM中有很多開發者使用的命令行選項，除了功能選項外，還有：

* 對某一pass有較大改動，in-tree開發時爲了防止衰退，設置一個預設爲false的enable變量
* 一段時間功能穩定，把預設值改爲true。在某些場合下發現衰退的用戶可以使用false作爲workaround
* 給一個pass提供更多輸入，用於測試

這個庫使用便捷，添加一個新選項只需要在一個局部文件中加一個變量。還提供了一些錦上添花的小功能，如推薦拼寫接近的選項。
但命令行解析的定製性很弱。比如：

* 一個`cl::opt<bool>`選項接受`-v 0 -v=0 -v false -v=false -v=False`等多種輸入方式
* 不便同時支持`--long`和`--no-long`。偶爾有需求時的workaround是給`--no-long`也設置一個變量。假如要處理兩個選項互相override，就要判斷兩個選項在命令行中的相對位置

面向用戶的外部工具往往有這類定製需求。GNU `getopt_long`的風格是`--long`。
`llvm/Support/ComandLine.h`裏`-long`和`--long`可以混用，很長一段時間不支持強制`--`。

LLVM binary utilities (llvm-nm、llvm-objdump、llvm-readelf等)爲了替代GNU binutils，
需要提供POSIX shell utilities風格的grouped short options (`-ab`表示`-a -b`)。
很長一段時間這個功能不被支持，困擾了想要遷移到LLVM binary utilities的用戶。

另外`cl::opt`是singleton，也可以定義局部變量動態增加選項，但這種用法很少見(llvm-readobj和llvm-cov)。
還有個很奇特的用法，opt工具中legacy pass manager自動獲取pass name列表，並註冊大量全局選項。

爲了防止錯誤，`cl::opt`不支持多次定義同一個選項。如果同時鏈接了shared object和archive兩種LLVM庫，就會觸發經典錯誤：

```
: CommandLine Error: Option 'help-list' registered more than once!
LLVM ERROR: inconsistency in registered CommandLine options
```

在Clang裏如果要設置`cl::opt`變量的值，可以用`-mllvm -option=value`。使用ld.lld/LLVMgold.so Full/Thin LTO也可以設置這些選項值，用`-plugin-opt=-option=value`(ld.lld也可用`-mllvm`)。

## `llvm/Option/OptTable.h`

原先給Clang開發，後來移入llvm，被llvm-objcopy、lld、llvm-symbolizer等採用。
用一個domain-specific language (TableGen)描述選項，生成一個parser。解析過的選項組織成一個object，每個選項用一個integer表示。
檢查一對預設值不定的boolean選項(`--demangle --no-demangle`)是否生效很容易：`Args.hasFlag(OPT_demangle, OPT_no_demangle, !IsAddr2Line)`。

```tablegen
multiclass B<string name, string help1, string help2> {
  def NAME: Flag<["--", "-"], name>, HelpText<help1>;
  def no_ # NAME: Flag<["--", "-"], "no-" # name>, HelpText<help2>;
}

multiclass Eq<string name, string help> {
  def NAME #_EQ : Joined<["--", "-"], name #"=">,
                  HelpText<help>;
  def : Separate<["--", "-"], name>, Alias<!cast<Joined>(NAME #_EQ)>;
}

defm debug_file_directory : Eq<"debug-file-directory", "Path to directory where to look for debug files">, MetaVarName<"<dir>">;
defm default_arch : Eq<"default-arch", "Default architecture (for multi-arch objects)">;
defm demangle : B<"demangle", "Demangle function names", "Don't demangle function names">;
def functions : F<"functions", "Print function name for a given address">;
```

```cpp
  opt::InputArgList Args = parseOptions(argc, argv, IsAddr2Line, Saver, Tbl);

  LLVMSymbolizer::Options Opts;
  ...
  Opts.DebugFileDirectory = Args.getAllArgValues(OPT_debug_file_directory_EQ);
  Opts.DefaultArch = Args.getLastArgValue(OPT_default_arch_EQ).str();
  Opts.Demangle = Args.hasFlag(OPT_demangle, OPT_no_demangle, !IsAddr2Line);
  Opts.DWPName = Args.getLastArgValue(OPT_dwp_EQ).str();
```

注意GCC的命令行選項不支持grouped short options，因此Clang也沒有需求。很長一段時間因爲缺少這個功能限制了它的使用場景。我在2020年7月加入了grouped short options (D83639)。

軼聞：LLD採用這個庫解析命令行選項。GNU ld實際上支持grouped short options，比如`ld.bfd -vvv`表示`-v -v -v`。我提出GNU ld實際上支持很多`-long`風格的選項，再支持grouped short options容易引起混亂。

```
% touch an ommand ':)'
% ld.bfd -you -can -ofcourse -use -this -Long -command -Line ':)'
:)
```

binutils 2.36有望deprecate grouped short options:)

再拓展一下，很多`getopt_long`用戶用一個switch加大量case處理命令行選項，很容易弄出各種各樣position dependent行爲。
對於大型build system，有時候搞不清楚compiler/linker options是在什麼地方添加的，有些position dependent行爲挺討厭的。
