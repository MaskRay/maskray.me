---
layout: post
title: A deep dive into SmallVector::push_back
author: MaskRay
tags: [llvm,codegen,performance]
---

tl;dr This blog post describes a recent `SmallVector::push_back` optimization for approximately trivially copyable element types.

`SmallVector` is LLVM's most-used container, and `push_back` its hot operation.
For the trivially-copyable specialization the fast path should be fast.

```cpp
#include <llvm/ADT/SmallVector.h>

void f(llvm::SmallVectorImpl<int> &v, int x) { v.push_back(x); }
```

`clang -S --target=x86_64 -O2 -DNDEBUG a.cc` generates:

```asm
push   rbp                 # callee-saved spills + a stack realignment,
push   rbx                 # all on the fast path
push   rax
mov    eax, [rdi + 8]      # size
cmp    eax, [rdi + 12]     # vs capacity
jae    .Lgrow
.Lstore:                   # reached from the fast path AND from .Lgrow
mov    rcx, [rdi]
mov    [rcx + rax*4], esi
inc    dword ptr [rdi + 8]
add    rsp, 8
pop    rbx
pop    rbp
ret
.Lgrow:
mov    rbx, rdi            # keep `this`/`x` alive across the call
mov    ebp, esi
call   SmallVectorBase<unsigned>::grow_pod
...
jmp    .Lstore
```

<!-- more -->

`push_back` reserves capacity and *then* stores, so the store at `.Lstore` is shared between the no-grow and post-grow paths.
On the grow path `this` and `x` must survive the `grow_pod` call, which means they are saved in callee-saved registers, leading to `push rbx`/`push rbp` in the prologue.
`push rbp` is needed to maintain the 16-byte alignment of the stack frame.

GCC's output is also inefficient:

```asm
push   rbp ; mov ebp, esi    # x -> rbp, in the entry block
push   rbx ; mov rbx, rdi    # this -> rbx
... ; cmp ; jnb .Lslow
.Lmerge:                     # reached by both paths, reads rbx/rbp
mov    rdx, [rbx] ; mov [rdx+rax*4], ebp ; ...
```

## Shrink wrapping can't remove it

Shrink wrapping relocates the save/restore of callee-saved registers; it never duplicates a block.
To carry `this`/`x` across the conditional `grow_pod` call into a store the fast path also reaches, a callee-saved register must be live from entry.
`clang -mllvm -debug-only=shrink-wrap` reports `No Shrink wrap candidate found`.
GCC's `-fshrink-wrap-separate` (on at `-O2`) does not optimize this as well.

The transformation that *would* help is tail duplication — give the slow path its own copy of the store so the fast path keeps `this`/`x` in their argument registers.
Neither compiler does it here, and it is not shrink-wrapping's job.

## Optimization: tail calling the slow path

<https://github.com/llvm/llvm-project/pull/206213> moves the grow-and-store out of line and tail calls it:

```cpp
LLVM_ATTRIBUTE_NOINLINE void growAndPushBack(ValueParamT Elt) {
  T Tmp = Elt;  // in case Elt aliases storage that grow() invalidates
  this->grow(this->size() + 1);
  std::memcpy(reinterpret_cast<void *>(this->end()), &Tmp, sizeof(T));
  this->set_size(this->size() + 1);
}

void push_back(ValueParamT Elt) {
  if (LLVM_UNLIKELY(this->size() >= this->capacity()))
    return growAndPushBack(Elt);
  std::memcpy(reinterpret_cast<void *>(this->end()), &Elt, sizeof(T));
  this->set_size(this->size() + 1);
}
```

The generated assembly is now optimal for the fast path:

```asm
mov    eax, [rdi + 8]
cmp    eax, [rdi + 12]
jae    growAndPushBack         # TAILCALL
mov    rcx, [rdi]
mov    [rcx + rax*4], esi
inc    dword ptr [rdi + 8]
ret
```

7 instructions instead of 14, no callee-saved registers, nothing to shrink-wrap.

The slow path, now in an out-of-line function (in a separate section using COMDAT), becomes even slower.

`noinline` is load-bearing, otherwise Clang and GCC may inline the helper back and the prologue returns.

```cpp
#include <llvm/ADT/SmallVector.h>
// noinline growAndPushBack is load-bearing for both Clang and GCC.
void DecodeMOVDDUPMask(unsigned n, llvm::SmallVectorImpl<int> &v) {
  for (unsigned l = 0; l < n; l += 2)
    for (unsigned i = 0; i < 2; ++i)
      v.push_back(i);
}
```

`T Tmp = Elt` handles `Elt` referencing the vector's own storage. It is elided for small by-value types.
Passing the element by reference to the out-of-line `growAndPushBack` makes it address-taken / memory-materialized (it must be readable at a fixed address across another non-inlined call), which defeats construct-in-place for large element types.
However, this is insignificant given that grow() has to copy `size()` elements.

## Results

`lld` `.text` shrinks 40,512 bytes; by-`const&` element types win most, e.g. `GotSection::addConstant` goes 167&nbsp;→&nbsp;45 bytes.
On the [LLVM compile-time tracker](https://llvm-compile-time-tracker.com/) the clang build is 0.41–0.51% fewer `instructions:u` across every configuration, for +0.13% binary size.

Sorted by relative size, a few outliers grow ~13.8% — the constexpr ByteCode interpreter (`Interp.cpp`, `EvalEmitter.cpp`).
A smaller `push_back` likely perturbs the bottom-up inliner's near-threshold decisions.

## `std::vector<T>::push_back` is slow in both libc++ and libstdc++

Both libraries need a stack frame for their `vector<int>::push_back` fast path. <https://godbolt.org/z/5h85M9Gr9>

```cpp
#include <llvm/ADT/SmallVector.h>
#include <vector>

void pb_int(std::vector<int> &v, int x) { v.push_back(x); }
void pb_int(llvm::SmallVectorImpl<int> &v, int x) { v.push_back(x); }

struct T {int x[32];};
void pb_Tcreate(std::vector<T> &v, int x){ v.push_back(T{{x, 1}}); }
void pb_Tcopy(std::vector<T> &v, const T &t){ v.push_back(t); }

void pb_Tcreate(llvm::SmallVectorImpl<T> &v, int x){ v.push_back(T{{x, 1}}); }
void pb_Tcopy(llvm::SmallVectorImpl<T> &v, const T &t){ v.push_back(t); }
```

libc++'s `push_back` forwards to `emplace_back`, which routes the grow decision through `std::__if_likely_else(cond, fast, slow)`.
The slow path is kept out of line, but as a by-reference lambda, so its closure `{&__end_, &__x, this}` is materialized on the stack and the trailing `this->__end_ = __end` is a merge.
The fast path therefore spills the closure and runs with a 48-byte frame:

```asm
push   rbx
sub    rsp, 48
mov    [rsp+12], esi          # spill x
mov    rax, [rdi+8]           # __end
mov    [rsp+16], rax
lea    rcx, [rsp+16] ; mov [rsp+24], rcx   # } closure {&__end,
lea    rcx, [rsp+12] ; mov [rsp+32], rcx   # }  &x, this}, built
mov    [rsp+40], rdi                       # }  on the fast path
jae    .Lslow                # else: store x; this->__end_ = __end
```

libstdc++ is heavier still: its `push_back` inlines `_M_realloc_insert`, pulling the whole reallocation — `operator new`, `memcpy`, `operator delete`, and the `length_error` throw — into the function. To keep state live across those calls the fast path holds six callee-saved registers, on both g++ and clang.

A direct out-of-line member taking `(this, Elt)` in registers — the `growAndPushBack` above — is what keeps the fast path free of both a frame and callee-saved registers.

Note: many libc++ builds enable [hardening](https://libcxx.llvm.org/Hardening.html) by default. Disable it (and exceptions) for the best performance:

```sh
-fno-exceptions -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_NONE
```

## Boost's small_vector has the same frame

`boost::container::small_vector<int, N>::push_back` tells the same story, independent of the inline capacity `N` (even `N == 0`):

```asm
sub    rsp, 24                  # frame on the fast path
mov    [rsp+12], esi            # spill x — dead on the fast path
mov    rax, [rdi+8]             # size (64-bit)
lea    rdx, [4*rax] ; add rdx, [rdi] ; cmp rax, [rdi+16]   # end; vs capacity
je     .Lgrow
mov    [rdx], esi ; inc rax ; mov [rdi+8], rax ; add rsp, 24 ; ret
.Lgrow:
lea    r8, [rsp+12]             # &x, for vector::priv_insert_…'s insert_emplace_proxy<const int&>
call   ...
```

`x` is spilled only so the cold grow path can pass its address to the emplace proxy, yet the `sub rsp, 24` and the spill land on the fast path (the store itself uses `esi`). Boost also keeps size and capacity as `size_t`, so `small_vector<int,0>` is 24 bytes — like `std::vector`, 8 more than `SmallVector<int,0>`.

## `absl::InlinedVector` has the same frame

`absl::InlinedVector<int, 4>::push_back`, with `clang -O2 -DNDEBUG -fno-exceptions`, tells the same frame story and adds two branches on top:

```asm
push   rax                  # frame
mov    [rsp+4], esi         # spill x — dead here; only the cold path needs &x
mov    rax, [rdi]           # metadata = (size << 1) | is_allocated
mov    edx, 4               # inline capacity N
test   al, 1                # is_allocated?  (#1: pick the capacity)
je     .L2
mov    rdx, [rdi+16]        #   heap capacity
.L2:
mov    rcx, rax ; shr rcx   # size = metadata >> 1
cmp    rcx, rdx ; je .Lslow # full?
test   al, 1                # is_allocated?  (#2: pick the data pointer)
je     .L4
mov    rdx, [rdi+8]         #   heap pointer
jmp    .L6
.L4: lea rdx, [rdi+8]       #   &inline buffer
.L6:
mov    [rdx+4*rcx], esi     # store
add    rax, 2 ; mov [rdi], rax  # size++ (it lives above bit 0)
pop    rax ; ret
.Lslow:
lea    rsi, [rsp+4] ; call …EmplaceBackSlow ; pop rax ; ret
```

The `push rax` and the spill are the libc++/Boost story once more: the cold `EmplaceBackSlow(const T&)` takes the element by address, so `&x` escapes onto the fast path.

The two `test al, 1` are new and come from the layout. `absl::InlinedVector` packs the size and an `is_allocated` bit into one word and unions the inline buffer with `{pointer, capacity}`.
With no stored data pointer, each access re-derives *both* the capacity and the base address from the bit, so it is tested twice.
The reward is a smaller object — `sizeof(absl::InlinedVector<int,4>)` is 24 vs `SmallVector<int,4>`'s 32.

`SmallVector` makes the opposite trade: it stores `BeginX` (always pointing at live storage) plus separate `size`/`capacity`, so push_back loads the pointer and capacity unconditionally — no `is_allocated` branch on the hot path — and the small-vs-heap test only matters inside `grow()`.
That costs 8 bytes at `<int,4>`, but `SmallVector<int,0>` is 16 bytes, the storage `absl::InlinedVector` can't express (it requires `N >= 1`).

## The dual: a `push_back` loop prefers `std::vector`

The tail-call's cost is the mirror of its win. Out-of-lining the slow path as `growAndPushBack(this)` passes the object's address to a `noinline` callee. Free for a single call; not in a loop, where the escape stops the optimizer from keeping the fields in registers across iterations.

```cpp
template <class V> int drain(int n) {
  V c;                                            // a local
  for (int i = 0; i < n; ++i) c.push_back(i);
  int s = 0; while (!c.empty()) { s += c.back(); c.pop_back(); }
  return s;
}
```

`std::vector`'s grow is inlined and never escapes `&v`, so `end`/`cap` stay in registers:

```asm
.Lloop:                       # std::vector<int>
  mov  [rax], r13d            # *end = i
  add  rax, 4                 # ++end       (register)
  cmp  r14, rax               # end == cap? (register)
  jne  .Lloop                 # 1 memory op / element
```

`SmallVector` reloads all three fields every iteration:

```asm
.Lloop:                       # llvm::SmallVector<int,0>
  mov  eax, [rsp+0x10]        # size      reload
  cmp  eax, [rsp+0x14]        # capacity  reload
  jae  .Lgrow
  mov  rcx, [rsp+0x8]         # BeginX    reload
  mov  [rcx + rax*4], ebx     # store
  inc  dword ptr [rsp+0x10]   # ++size    RMW
```

Keeping the fields in registers needs a slow path that takes them *by value* and *returns* the new `{BeginX, Capacity}` — so nothing escapes. But then `push_back` must keep `this`/`Elt` live across the call to write the result back, and the frame [#206213](https://github.com/llvm/llvm-project/pull/206213) removed comes back.

| design | single `push_back` | loop |
| --- | --- | --- |
| out-of-line member + tail-call (shipped) | no frame ✅ | metadata in memory ❌ |
| value-returning grow | frame ❌ | metadata in registers ✅ |

## Aside: "approximately trivially copyable"

```cpp
  std::is_trivially_copy_constructible<T> &&
  std::is_trivially_move_constructible<T> &&
  std::is_trivially_destructible<T>
```

is the predicate that selects the `SmallVectorTemplateBase<..., true>` specialization, where copy/move construction optimizes to `memcpy` and `destroy_range` is a no-op.

The condition is broader than `is_trivially_copyable`, which also demands trivial assignment.
The motivating case is `std::pair<int,int>`: its constructors are trivial, but its assignment is user-provided (to support `pair<T&,U&>`), so `is_trivially_copyable` is `false`.
`SmallVector` only ever copies or moves elements by construction into uninitialized storage (`memcpy`), never by assignment, so the distinction is unobservable and `memcpy` is sound — and `std::pair<POD,POD>` stays on the fast path.

The condition is also stronger than trivial relocatability, whose operational definition is just `is_trivially_move_constructible && is_trivially_destructible`.
The extra `is_trivially_copy_constructible` is there because `SmallVector` also calls `memcpy`s when *copying* a live element — `push_back(const T&)`, or copy-constructing from another vector.

```
  is_pod  ⊆  is_trivially_copyable  ⊆  SmallVector condition  ⊆  trivially relocatable
```

## Aside: five-class hierarchy

`SmallVector` is the bottom of a five-class hierarchy. The count looks heavy, but each layer varies over exactly one axis:

```cpp
SmallVectorBase<Size_T>               // independent of T
  SmallVectorTemplateCommon<T>        // depends on T, not on triviality
    SmallVectorTemplateBase<T, bool>  // specialized on approximate trivial-copyability
      SmallVectorImpl<T>              // independent of N
        SmallVector<T, N>             // adds the inline buffer
```

- `SmallVectorBase<Size_T>` holds the three members (`BeginX`, `Size`, `Capacity`) and the out-of-line `grow_pod`/`mallocForGrow`. It is templated only on the size type, so those two heavyweight functions are emitted twice for the whole program — one `uint32_t`, one `uint64_t` — not once per element type.
- `SmallVectorTemplateCommon<T>` adds what is identical for trivial and non-trivial `T`: the iterators, `front`/`back`/`data`/`operator[]`, and the internal-reference helpers.
- `SmallVectorTemplateBase<T, bool>` is the specialization point. The `true` half uses `memcpy` and `grow_pod`; the `false` half uses constructors, `destroy_range`, and `growAndEmplaceBack`.
- `SmallVectorImpl<T>` erases `N`. A `SmallVectorImpl<T> &` parameter accepts any inline capacity, and is the canonical way to pass a `SmallVector` around.
- `SmallVector<T, N>` carries the inline buffer.

## Aside: a smaller header than `std::vector`

`std::vector` stores three 8-byte members. `SmallVector` stores a begin pointer plus a 32-bit size and a 32-bit capacity when `sizeof(T) >= 4`.

```cpp
template <class Size_T> class SmallVectorBase {
  void *BeginX;
  Size_T Size, Capacity;
};
```

So for `int`, pointers, and most structs the header is **16 bytes** — 8 fewer than `std::vector`.
The cost of carrying a size instead of an end pointer is that addressing the end (`begin + size * sizeof(T)`) needs a multiply, visible when `sizeof(T)` is not a power of two.

## Takeaways

- A fast/slow merge that rejoins after a call forces callee-saved spills onto the hot path, and shrink-wrapping can't remove them.
- A tail-called out-of-line slow path removes the overhead.
- Inliner behavior makes size effects are non-monotonic.
