---
layout: post
title: Skipping boring functions in debuggers
author: MaskRay
tags: [gdb,lldb]
---

In debuggers, stepping into a function with arguments that involve function calls may step into the nested function calls, even if they are simple and uninteresting, such as those found in the C++ STL.

## GDB

Consider the following example:

```cpp
#include <cstdio>
#include <memory>
#include <vector>
using namespace std;

void foo(int i, int j) {
  printf("%d %d\n", i, j);
}

int main() {
  auto i = make_unique<int>(3);
  vector v{1,2};
  foo(*i, v.back()); // step into
}
```

When GDB stops at the `foo` call, the `step` (`s`) command will step into `std::vector::back` and `std::unique_ptr::operator*`.
While you can execute `finish` (`fin`) and then execute `s` again, it's time-consuming and distracting, especially when dealing with complex argument expressions.

<!-- more -->

```
% g++ -g a.cc -o a
% gdb ./a
...
(gdb) s
std::vector<int, std::allocator<int> >::back (this=0x7fffffffddd0) at /usr/include/c++/14.2.1/bits/stl_vector.h:1235
1235          back() _GLIBCXX_NOEXCEPT
(gdb) fin
Run till exit from #0  std::vector<int, std::allocator<int> >::back (this=0x7fffffffddd0) at /usr/include/c++/14.2.1/bits/stl_vector.h:1235
0x00005555555566f8 in main () at a.cc:13
13        foo(*i, v.back());
Value returned is $1 = (__gnu_cxx::__alloc_traits<std::allocator<int>, int>::value_type &) @0x55555556c2d4: 2
(gdb) s
std::unique_ptr<int, std::default_delete<int> >::operator* (this=0x7fffffffddc0) at /usr/include/c++/14.2.1/bits/unique_ptr.h:447
447             __glibcxx_assert(get() != pointer());
(gdb) fin
Run till exit from #0  std::unique_ptr<int, std::default_delete<int> >::operator* (this=0x7fffffffddc0) at /usr/include/c++/14.2.1/bits/unique_ptr.h:447
0x0000555555556706 in main () at a.cc:13
13        foo(*i, v.back());
Value returned is $2 = (int &) @0x55555556c2b0: 3
(gdb) s
foo (i=3, j=2) at a.cc:7
7         printf("%d %d\n", i, j);
```

This problem was tracked as a feature request in 2003: <https://sourceware.org/bugzilla/show_bug.cgi?id=8287>.
Fortunately, GDB provides the [`skip` command](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Skipping-Over-Functions-and-Files.html) to skip functions that match a regex or filenames that match a glob (GDB 7.12 feature).
You can skip all demangled function names that start with `std::`.

```
skip -rfu ^std::
```

Alternatively, you can execute `skip -gfi /usr/include/c++/*/bits/*` to skip these libstdc++ files.

Important note:

The `skip` command's file matching behavior uses the `fnmatch` function with the [`FNM_FILE_NAME`](https://man7.org/linux/man-pages/man3/fnmatch.3.html) flag.
This means the wildcard character (`*`) won't match slashes. So, skip `-gfi /usr/*` won't exclude `/usr/include/c++/14.2.1/bits/stl_vector.h`.

I proposed to [drop the `FNM_FILE_NAME` flag](https://sourceware.org/pipermail/gdb-patches/2024-December/214422.html).
With GDB 17, I will be able to skip a project directory with  
```
skip -gfi */include/llvm/ADT/*
```

instead of
```
skip -gfi /home/ray/llvm/llvm/include/llvm/ADT/*
```

### User functions called by skipped functions

When a function (let's call it "A") is skipped during debugging, any user-defined functions that are called by "A" will also be skipped.

For example, consider the following code snippet:

```cpp
std::vector<int> a{1, 2};
if (std::all_of(a.begin(), a.end(), predicate)) {
}
```

If `std::all_of` is skipped due to a `skip` command, `predicate` called within `std::all_of` will also be skipped when you execute `s` at the if statement.

## LLDB

By default, LLDB avoids stepping into functions whose names start with `std::` when you use the `s` (`step`, `thread step-in`) command.
This behavior is controlled by a setting:

```
(lldb) settings show target.process.thread.step-avoid-regexp
target.process.thread.step-avoid-regexp (regex) = ^std::
(lldb) set sh target.process.thread.step-avoid-libraries
target.process.thread.step-avoid-libraries (file-list) =
```

`target.process.thread.step-avoid-libraries` can be used to skip functions defined in a library.

While the command `settings set` is long, you can shorten it to `set set`.

## Visual Studio

Visual Studio provides a debugging feature [_Just My Code_](https://learn.microsoft.com/en-us/visualstudio/debugger/just-my-code?view=vs-2022) that automatically steps over calls to system, framework, and other non-user code.

It also supports a `Step Into Specific` command, which seems interesting.

The implementation inserts a call to `__CheckForDebuggerJustMyCode` at the start of every user function.
The function (`void __CheckForDebuggerJustMyCode(const char *flag)`) takes a global variable defined in the `.msvcjmc` section and determines whether the debugger should stop.

This LLDB feature request has a nice description: <https://github.com/llvm/llvm-project/issues/61152>.

For the `all_of` example, the feature can possibly allow the debugger to stop at `test`.

```cpp
std::vector<int> a{1, 2};
if (std::all_of(a.begin(), a.end(), test)) {
}
```

## Fuchsia zxdb

The Fuchsia debugger "zxdb" provides a command ["ss"](https://fuchsia.dev/fuchsia-src/development/debugger/execution) similar to Visual Studio's "Step Into Specific".

```
[zxdb] ss
  1 std::string::string
  2 MyClass::MyClass
  3 HelperFunctionCall
  4 MyClass::~MyClass
  5 std::string::~string
  quit
>
```
