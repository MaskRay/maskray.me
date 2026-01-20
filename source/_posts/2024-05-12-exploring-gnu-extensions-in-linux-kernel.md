---
layout: post
title: Exploring GNU extensions in the Linux kernel
author: MaskRay
tags: [gcc,linux,kernel]
---

The Linux kernel is written in C, but it also leverages extensions provided by GCC.
In 2022, it moved from GCC/Clang `-std=gnu89` to `-std=gnu11`.
This article explores my notes on how these GNU extensions are utilized within the kernel.

<!-- more -->

## Statement expressions

[Statement expressions](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html) are commonly used in macros.

## [Local labels](https://gcc.gnu.org/onlinedocs/gcc/Local-Labels.html)

Some macros use this extension to restart a for loop in a macro's replacement list.

```c
// include/linux/wait.h
#define ___wait_event(wq_head, condition, state, exclusive, ret, cmd)           \
({                                                                              \
        __label__ __out;                                                        \
        struct wait_queue_entry __wq_entry;                                     \
        long __ret = ret;       /* explicit shadow */                           \
                                                                                \
        init_wait_entry(&__wq_entry, exclusive ? WQ_FLAG_EXCLUSIVE : 0);        \
        for (;;) {                                                              \
                long __int = prepare_to_wait_event(&wq_head, &__wq_entry, state);\
                                                                                \
                if (condition)                                                  \
                        break;                                                  \
                                                                                \
                if (___wait_is_interruptible(state) && __int) {                 \
                        __ret = __int;                                          \
                        goto __out;                                             \
                }                                                               \
                                                                                \
                cmd;                                                            \
        }                                                                       \
        finish_wait(&wq_head, &__wq_entry);                                     \
__out:  __ret;                                                                  \
})
```

`include/linux/instruction_pointer.h` utilizes the feature to obtain the current instruction pointer:

```c
#define _THIS_IP_  ({ __label__ __here; __here: (unsigned long)&&__here; })
```

## [Labels as values and computed goto statements](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)

Bytecode interpreters often leverage this extension.
BPF, for instance, utilizes this extension.
We can also see this extension in `drm_exec_retry_on_contention` for the Direct Rendering Manager.

When it comes to Position-Dependent Code (PDC), a switch statement with a default label marked as `__builtin_unreachable` can be just as efficient as using labels as values.
However, for Position-Independent Code (PIC), absolute addresses offer a slight performance edge, although at the cost of requiring more dynamic relocations.

```c
static void *tbl[] = {&&do_ret, &&do_inc, &&do_dec};
#define NEXT goto *tbl[*pc++]
NEXT;
for(;;) {
do_ret: return val;
do_inc: val++; NEXT;
do_dec: val--; NEXT;
}
```

## `typeof` and `__auto_type`

`typeof` is used in numerous code. `__auto_type` has a few occurrences.

C23 standardizes `typeof` and `auto`.

## [Conditionals with omitted operands](https://gcc.gnu.org/onlinedocs/gcc/Conditionals.html)

`x ?: y` is known as the "Elvis operator". This is used in numerous code.

## [Empty structures](https://gcc.gnu.org/onlinedocs/gcc/Empty-Structures.html)

The C standard (at least C11 and C23) specifies that:

> If the member declaration list does not contain any named members, either directly or via an anonymous structure or anonymous union, the behavior is undefined.

The empty structure extension enables a dummy structure when a configuration option disables the functionality.

```c
#ifdef CONFIG_GENERIC_ENTRY

struct syscall_user_dispatch {
        char __user     *selector;
        unsigned long   offset;
        unsigned long   len;
        bool            on_dispatch;
};

#else

struct syscall_user_dispatch {};

#endif
```

## [Case ranges](https://gcc.gnu.org/onlinedocs/gcc/Case-Ranges.html)

This extension (`case low ... high:`), frequently used in the kernel, allows us to specify a range of consecutive values in a single case label within a switch statement.

## [Object size checking](https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html)

glibc 2.3.4 introduced `_FORTIFY_SOURCE` in 2004 to catch security errors due to misuse of some C library functions (primarily buffer overflow).
The implementation leverages inline functions and `__builtin_object_size`.

Linux kernel introduced `CONFIG_FORTIFY_SOURCE` in [2017-07](https://git.kernel.org/linus/6974f0c4555e285ab217cee58b6e874f776ff409).
Like the userspace, the implementation relies on optimizations like inlining and constant folding.

For example, `include/linux/string.h` combines this feature with `BUILD_BUG_ON` (`__attribute__((error(...)))`) to provide compile-time errors:

```c
#define strtomem(dest, src)     do {                                    \
        const size_t _dest_len = __builtin_object_size(dest, 1);        \
        const size_t _src_len = __builtin_object_size(src, 1);          \
                                                                        \
        BUILD_BUG_ON(!__builtin_constant_p(_dest_len) ||                \
                     _dest_len == (size_t)-1);                          \
        memcpy(dest, src, strnlen(src, min(_src_len, _dest_len)));      \
} while (0)
```

The Clang implementation utilizes `__attribute__((pass_object_size(type)))` and `__attribute__((overloadable(...)))`.

Clang [introduced `__builtin_dynamic_object_size`](https://reviews.llvm.org/D56760) in 2019 to possibly evaluate the size dynamically.
The feature was ported to GCC in 2021 and is used by the kernel.

## Pragmas

[`#pragma GCC diagnostic`](https://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Pragmas.html) is occasionally used to disable a local diagnostic.

`include/linux/hidden.h` uses [`#pragma GCC visibility("hidden")`](https://gcc.gnu.org/onlinedocs/gcc/Visibility-Pragmas.html) to force direct access (instead of GOT indirection) for external symbols for `-fpie/-fpic`.

bpf uses [`#pragma GCC poison`](https://gcc.gnu.org/onlinedocs/cpp/Pragmas.html) to forbid some undesired identifiers.

[Structure-layout pragmas](https://gcc.gnu.org/onlinedocs/gcc/Structure-Layout-Pragmas.html) are commonly used.

## Inline assembly

The Linux kernel uses inline assembly for a few reasons

* Hardware interaction: Certain operations require direct interaction with the underlying hardware capabilities, which might not be expressible in standard C.
* Performance optimization: In critical code paths, inline assembly can be used to squeeze out the last bit of performance by utilizing processor-specific instructions or bypassing certain C language constructs that might introduce overhead.
* Special facility: Some features, such as static keys, [alternatives runtime patching](https://blogs.oracle.com/linux/post/exploring-arm64-runtime-patching-alternatives), cannot be implemented with standard C.

## Built-in functions

### Special machine code instructions

GCC provides built-in functions to generate machine code instructions designed for certain tasks like popcount/clz/ctz.
The compiler has more information about the function's purpose and leverages internal optimizations tailored for these tasks.

### `__builtin_choose_expr`

This is analogous to `? :` but the condition is a constant expression and the return type is not altered by promotion rules.
In GCC, `__builtin_choose_expr` is not available in C++.

```c
// include/linux/math.h
#define abs(x)  __abs_choose_expr(x, long long,                         \
                __abs_choose_expr(x, long,                              \
                __abs_choose_expr(x, int,                               \
                __abs_choose_expr(x, short,                             \
                __abs_choose_expr(x, char,                              \
                __builtin_choose_expr(                                  \
                        __builtin_types_compatible_p(typeof(x), char),  \
                        (char)({ signed char __x = (x); __x<0?-__x:__x; }), \
                        ((void)0)))))))

#define __abs_choose_expr(x, type, other) __builtin_choose_expr(        \
        __builtin_types_compatible_p(typeof(x),   signed type) ||       \
        __builtin_types_compatible_p(typeof(x), unsigned type),         \
        ({ signed type __x = (x); __x < 0 ? -__x : __x; }), other)
```

### `__builtin_constant_p`

`__builtin_constant_p` identifies whether an expression can be evaluated to a constant.
This capability unlocks several code patterns.

*Conditional static assertions*:

```c
BUILD_BUG_ON(__builtin_constant_p(nr) && nr != 1);
```

`__builtin_constant_p` decides whether `MAYBE_BUILD_BUG_ON` expands to a compile-time static assert mechanism (`__attribute__((error(...)))`) or a runtime mechanism.
```c
#define MAYBE_BUILD_BUG_ON(cond)                        \
        do {                                            \
                if (__builtin_constant_p((cond)))       \
                        BUILD_BUG_ON(cond);             \
                else                                    \
                        BUG_ON(cond);                   \
        } while (0)
```

*Alternative code paths:*

Sometimes, the constant expression case might enable a more efficient code path.

```c
#define __trace_if_var(cond) (__builtin_constant_p(cond) ? (cond) : __trace_if_value(cond))
```

*Constant Folding Optimizations:*

By knowing expressions are constant, the compiler can perform constant folding optimizations.
This means pre-calculating the expression's value during compilation and replacing it with the actual constant in the code.
This leads to potentially smaller and faster code as the calculation doesn't need to be done at runtime.
However, I notice that `__builtin_constant_p` is often abused. This is one of the source why the kernel cannot be built with `-O0`.

### `__builtin_expect`

Used by `likely` and `unlikely` macros to give the compiler hints for optimization.

```c
// include/linux/compiler.h
# define likely(x)   __builtin_expect(!!(x), 1)
# define unlikely(x) __builtin_expect(!!(x), 0)
```

### `__builtin_frame_address`

Used by stack trace utilities.

### `__builtin_offsetof`

Used to implement `offsetof`.

### `__builtin_*_overflow`

Used by `include/linux/overflow.h` for overflow checking.

### `__builtin_prefetch`

Used as the default implementation of `prefetch` and `prefetchw`.

### `__builtin_unreachable`

Used by `include/linux/compiler.h:unreachable` to inform unreachability for optimization or diagnostics suppressing purposes.

### `__builtin_types_compatible_p`

This is often used to emulate C11 `static_assert`

```c
#define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))

// include/linux/highmem-internal.h
#define kunmap_atomic(__addr)                                   \
do {                                                            \
        BUILD_BUG_ON(__same_type((__addr), struct page *));     \
        __kunmap_atomic(__addr);                                \
} while (0)
```

or implement C++ function overloading, e.g.

```c
// fs/bcachefs/util.h
#define strtoi_h(cp, res)                                               \
        ( type_is(*res, int)            ? bch2_strtoint_h(cp, (void *) res)\
        : type_is(*res, long)           ? bch2_strtol_h(cp, (void *) res)\
        : type_is(*res, long long)      ? bch2_strtoll_h(cp, (void *) res)\
        : type_is(*res, unsigned)       ? bch2_strtouint_h(cp, (void *) res)\
        : type_is(*res, unsigned long)  ? bch2_strtoul_h(cp, (void *) res)\
        : type_is(*res, unsigned long long) ? bch2_strtoull_h(cp, (void *) res)\
        : -EINVAL)
```

`include/linux/jump_label.h` combines `__builtin_types_compatible_p` and dead code elimination to emulate C++17 `if constexpr`.
`____wrong_branch_error` is not defined. If the unexpected code path is taken, there will be a linker error.

```c
#define static_branch_likely(x)                                                 \
({                                                                              \
        bool branch;                                                            \
        if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))   \
                branch = !arch_static_branch(&(x)->key, true);                  \
        else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
                branch = !arch_static_branch_jump(&(x)->key, true);             \
        else                                                                    \
                branch = ____wrong_branch_error();                              \
        likely_notrace(branch);                                                         \                                                                                 })
```

### BTF built-in functions

### [Clang thread safety analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html)

See [perf mutex: Add thread safety annotations](https://git.kernel.org/linus/bfa339ceda3c9e49ffb58c7de50fd86912ab9e6d).

## Function attributes

`tools/include/linux/compiler.h` defines macros for many commonly used attributes.
Many of them are commonly used by other projects and probably not worth describing.

`tools/testing/selftests` has a few `__attribute__((constructor))` for initialization.

`__attribute__((error(...)))` is used with inlining and dead code elimination to provide static assertion.
```c
// include/linux/compiler_types.h
#ifdef __OPTIMIZE__
# define __compiletime_assert(condition, msg, prefix, suffix)           \
        do {                                                            \
                __noreturn extern void prefix ## suffix(void)           \
                        __compiletime_error(msg);                       \
                if (!(condition))                                       \
                        prefix ## suffix();                             \
        } while (0)
#else
# define __compiletime_assert(condition, msg, prefix, suffix) do { } while (0)
#endif
```

`__attribute__((naked))` (`__naked`) is used by arm and bpf code.

`__attribute__((section(...)))` is used to place functions in the specified section, e.g. `.init.text`.

`__attribute__((weak))` is used to allow a generic function to be replaced by an arch-specific implementation.
The weak reference feature is used by some `__start_<sectionname>`/`__stop_<sectionname>` encapsulation symbols.

`#define __noendbr    __attribute__((nocf_check))` is used to disable Intel CET for some special functions.

## Variable attributes

`__attribute__((cleanup(...)))` runs a function when the variable goes out of scope.
`include/linux/cleanup.h` defines `DEFINE_FREE` and `__free` based on this feature.

```c
// include/linux/slab.h
DEFINE_FREE(kfree, void *, if (!IS_ERR_OR_NULL(_T)) kfree(_T))
// kernel/irq/irq_sim.c
        struct irq_sim_work_ctx *work_ctx __free(kfree) =
                                kmalloc(sizeof(*work_ctx), GFP_KERNEL);
```

## Type attributes

`__attribute__((aligned(...)))` and `__attribute__((packed))` are often to control structure layouts.

## Language dialects

### `-fno-delete-null-pointer-checks`

Assume that programs can safely dereference null pointers, and code or data element may reside at address zero.

### `-fno-strict-aliasing`

The C11 standard (section 6.5) defines strict aliasing rules.

> An object shall have its stored value accessed only by an lvalue expression that has one of the following types:
>
> — a type compatible with the effective type of the object,
> — a qualified version of a type compatible with the effective type of the object,
> — a type that is the signed or unsigned type corresponding to the effective type of the object,
> — a type that is the signed or unsigned type corresponding to a qualified version of the effective
> type of the object,
> — an aggregate or union type that includes one of the aforementioned types among its members
> (including, recursively, a member of a subaggregate or contained union), or
> — a character type.

Compilers leverage these aliasing rules for optimizations, which can be disabled with `-fno-strict-aliasing`.

Linus Torvalds has expressed reservations about strict aliasing.
Here is a [discussion](https://lore.kernel.org/all/CA+55aFyoxvrA-7X_8ZfQXd1=GfLT96ZaHAM+o5gKwOOfiy_HDA@mail.gmail.com/) from 2018.

### `-fno-strict-overflow`

In GCC, this option is identical to `-fwrapv -fwrapv-pointer`: make signed integer overflow defined using wraparound semantics.
While avoiding undefined behaviors, unexpected arithmetic overflow bugs might be lurking.

[[RFC] Mitigating unexpected arithmetic overflow](https://lore.kernel.org/all/202404291502.612E0A10@keescook/) is a 2024-05 thread about detecting and mitigating unsigned integer overflow.
