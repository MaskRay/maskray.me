---
layout: post
title: clang-format and single-line statements
author: MaskRay
tags: [clang-format,c++]
---

The Google C++ Style is widely adopted by projects. It contains a brace omission guideline in [Looping and branching statements](https://google.github.io/styleguide/cppguide.html#Formatting_Looping_Branching):

> For historical reasons, we allow one exception to the above rules: the curly braces for the controlled statement or the line breaks inside the curly braces may be omitted if as a result the entire statement appears on either a single line (in which case there is a space between the closing parenthesis and the controlled statement) or on two lines (in which case there is a line break after the closing parenthesis and there are no braces).

<!-- more -->

```cpp
// OK - fits on one line.
if (x == kFoo) { return new Foo(); }

// OK - braces are optional in this case.
if (x == kFoo) return new Foo();

// OK - condition fits on one line, body fits on another.
if (x == kBar)
  Bar(arg1, arg2, arg3);
```

In clang-format's predefined Google style for C++, there are two related style options:

```
% clang-format --dump-config --style=Google | grep -E 'AllowShort(If|Loop)'
AllowShortIfStatementsOnASingleLine: WithoutElse
AllowShortLoopsOnASingleLine: true
```

The two options cause clang-format to aggressively join lines for the following code:

```cpp
for (int x : a)
  foo(x);

while (cond())
  foo(x);

if (x)
  foo(x);
```

As a heavy debugger user, I find this behavior cumbersome.

```cpp
// clang-format --style=Google
#include <vector>
void foo(int v) {}
int main() {
  std::vector<int> a{1, 2, 3};
  for (int x : a) foo(x); // breakpoint
}
```

When GDB stops at the `for` loop, how can I step into the loop body? Unfortunately, it's not simple.

If I run `step`, GDB will dive into the implementation detail of the range-based for loop. It will stop at the `std::vector::begin` function.
Stepping out and executing `step` again will stop at the `std::vector::end` function.
Stepping out and executing `step` another time will stop at the `operator!=` function of the iterator type.
Here is an interaction example with GDB:

```
(gdb) n
5         for (int x : a) foo(v);
(gdb) s
std::vector<int, std::allocator<int> >::begin (this=0x7fffffffdcc0) at /usr/include/c++/14.2.1/bits/stl_vector.h:873
873           begin() _GLIBCXX_NOEXCEPT
(gdb) fin
Run till exit from #0  std::vector<int, std::allocator<int> >::begin (this=0x7fffffffdcc0) at /usr/include/c++/14.2.1/bits/stl_vector.h:873
0x00005555555561d5 in main () at a.cc:5
5         for (int x : a) foo(v);
Value returned is $1 = 1
(gdb) s
std::vector<int, std::allocator<int> >::end (this=0x7fffffffdcc0) at /usr/include/c++/14.2.1/bits/stl_vector.h:893
893           end() _GLIBCXX_NOEXCEPT
(gdb) fin
Run till exit from #0  std::vector<int, std::allocator<int> >::end (this=0x7fffffffdcc0) at /usr/include/c++/14.2.1/bits/stl_vector.h:893
0x00005555555561e5 in main () at a.cc:5
5         for (int x : a) foo(v);
Value returned is $2 = 0
(gdb) s
__gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int> > > (__lhs=1, __rhs=0) at /usr/include/c++/14.2.1/bits/stl_iterator.h:1235
1235        { return __lhs.base() != __rhs.base(); }
(gdb) fin
Run till exit from #0  __gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int> > > (__lhs=1, __rhs=0) at /usr/include/c++/14.2.1/bits/stl_iterator.h:1235
0x0000555555556225 in main () at a.cc:5
5         for (int x : a) foo(v);
Value returned is $3 = true
(gdb) s
__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator* (this=0x7fffffffdca0) at /usr/include/c++/14.2.1/bits/stl_iterator.h:1091
1091          { return *_M_current; }
(gdb) fin
Run till exit from #0  __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator* (this=0x7fffffffdca0) at /usr/include/c++/14.2.1/bits/stl_iterator.h:1091
0x00005555555561f7 in main () at a.cc:5
5         for (int x : a) foo(v);
Value returned is $4 = (int &) @0x55555556b2b0: 1
```

You can see that this can significantly hinder the debugging process, as it forces the user to delve into uninteresting function calls of the range-based for loop.

In contrast, when the loop body is on the next line, we can just run `next` to skip the three uninteresting function calls:

```cpp
for (int x : a) // next
  foo(x); // step
```

The `AllowShortIfStatementsOnASingleLine` style option is similar.
While convenient for simple scenarios, it can sometimes hinder debuggability.

For the following code, it's not easy to skip the `c()` and `d()` function calls if you just want to step into `foo(v)`.

```cpp
if (c() && d()) foo(v);
```

Many developers, mindful of potential `goto fail`-like issues, often opt to include braces in their code.
clang-format's default style can further reinforce this practice.

```cpp
// clang-format does not join lines.
if (v) {
  foo(v);
}
for (int x : a) {
  foo(x);
}
```

## Other predefined styles

clang-format's Chromium style is a variant of the Google style and does not have the aforementioned problem.
The LLVM style, and many styles derived from it, do not have the problem either.

```
% clang-format --dump-config --style=Chromium | grep -E 'AllowShort(If|Loop)'
AllowShortIfStatementsOnASingleLine: Never
AllowShortLoopsOnASingleLine: false
% clang-format --dump-config --style=LLVM | grep -E 'AllowShort(If|Loop)'
AllowShortIfStatementsOnASingleLine: Never
AllowShortLoopsOnASingleLine: false
```

## A comparative look at other Languages

Go, Odin, and Rust require `{}` for if statements but omit `()`, striking a balance between clarity and conciseness.
C/C++'s required ()` makes opt-in braces feel a bit verbose.

C3 and Jai, similar to C++, make `{}` optional.
