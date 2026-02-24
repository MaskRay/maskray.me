---
layout: post
title: Bit-field layout
author: MaskRay
tags: [c,clang,gcc]
---

The C and C++ standards leave nearly every detail to the implementation.
C23 §6.7.3.2:

> An implementation may allocate any addressable storage unit large enough to hold a bit-field. If enough space remains,
> a bit-field that immediately follows another bit-field in a structure shall be packed into adjacent bits of the same
> unit. If insufficient space remains, whether a bit-field that does not fit is put into the next unit or overlaps
> adjacent units is implementation-defined. The order of allocation of bit-fields within a unit (high-order to low-order
> or low-order to high-order) is implementation-defined. The alignment of the addressable storage unit is unspecified

C++ is also terse — `[class.bit]p1`:

> Allocation of bit-fields within a class object is implementation-defined. Alignment of bit-fields is implementation-defined. Bit-fields are packed into some addressable allocation unit.

The actual rules come from the platform ABI:

- **Itanium ABI** — used on Linux, macOS, BSD, and most non-Windows platforms.
  The Itanium C++ ABI ([section 2.4](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#class-types))
  defers bit-field placement to "the base C ABI" but adds its own constraints
  (notably: bit-fields are never placed in the tail padding of a base class).
- System V ABI Processor Supplement. The x86-64 psABI says little about bit-fields,
  while the [AArch64 AAPCS](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst#id115) has a more detailed description.
- **Microsoft ABI** — used on Windows (MSVC). In GCC and Clang, structs with the `ms_struct` attribute also mimics this ABI.

Clang implements both ABIs in `clang/lib/AST/RecordLayoutBuilder.cpp`.
It processes bit-fields in **two distinct phases**:

1. **Layout** (storage units) — assign a bit offset to every bit-field.
   This is ABI-specified and determines `sizeof` and `alignof`.
2. **Codegen** (access units) — choose what LLVM IR loads and stores to emit.
   This is a compiler optimization that affects generated code but not the ABI.

Understanding these separately is the key to understanding bit-fields.
This article focuses on Itanium (the default on most platforms), with a section on how the Microsoft ABI differs.

## Phase 1: Storage Units

In `clang/lib/AST/RecordLayoutBuilder.cpp`, `ItaniumRecordLayoutBuilder::LayoutFields` lays out fields of a `RecordDecl`.
For each bit field, it calls `LayoutBitField` to determine the storage unit and bit offset.

A **storage unit** is a region of `sizeof(T)` bytes, by default aligned to `alignof(T)`.
For an `int` bit-field, that's a 4-byte region at a 4-byte-aligned offset.
The alignment can be reduced by the `packed` attribute and `#pragma pack`.

- `StorageUnitSize = sizeof(T) * 8` — the unit's size in bits
- `FieldAlign = alignof(T)` in bits — the unit's alignment (before modifiers)
- `FieldOffset` — the first bit after the last bit-field

### Itanium's Core Rule

```cpp
if (FieldSize == 0 ||
    (AllowPadding &&
     (FieldOffset & (FieldAlign-1)) + FieldSize > StorageUnitSize))
  FieldOffset = alignTo(FieldOffset, FieldAlign);
```

Compute where `FieldOffset` falls within its aligned storage unit.
If the remaining space is less than `FieldSize`, round up to the next aligned boundary.
Otherwise, pack the bit-field at the current position.

### Declared Type Matters

Consider two structs that store the same total number of bits (7 + 7 + 2 = 16) but use different declared types:

```c
struct U8  { uint8_t  a:7, b:7, c:2; };   // sizeof = 3
struct U16 { uint16_t a:7, b:7, c:2; };   // sizeof = 2

struct S1 { int a:14; int b:10; int c:30; };   // sizeof = 8
```

**Walk-through for `U8`** (all fields have StorageUnitSize = 8, FieldAlign = 8):

- `a` at bit 0. Position = 0, 0 + 7 = 7 <= 8. Fits. **Offset = 0.**
- `b` at bit 7. Position = 7, 7 + 7 = 14 > 8. Doesn't fit. New unit at bit 8. **Offset = 8.**
- `c` at bit 15. Position = 15 - 8 = 7, 7 + 2 = 9 > 8. Doesn't fit. New unit at bit 16. **Offset = 16.**

Three 1-byte storage units. `sizeof(U8) = 3`. Eight padding bits wasted.

**Walk-through for `U16`** (all fields have StorageUnitSize = 16, FieldAlign = 16):

- `a` at bit 0. Position = 0, 0 + 7 = 7 <= 16. Fits. **Offset = 0.**
- `b` at bit 7. Position = 7, 7 + 7 = 14 <= 16. Fits. **Offset = 7.**
- `c` at bit 14. Position = 14, 14 + 2 = 16 <= 16. Fits. **Offset = 14.**

One 2-byte storage unit. `sizeof(U16) = 2`. No waste.

**Walk-through for `S1`** (all fields have StorageUnitSize = 32, FieldAlign = 32):

- `a` at bit 0. Position = 0, 14 fits in 32. **Offset = 0.**
- `b` at bit 14. Position = 14, 14 + 10 = 24 <= 32. Fits. **Offset = 14.**
  Bits 24–31 are padding (unfilled tail of the first storage unit).
- `c` at bit 24. Position = 24, 24 + 30 = 54 > 32. Doesn't fit. New unit at bit 32. **Offset = 32.**
  Bits 62–63 are padding (unfilled tail of the second storage unit).

`sizeof(S1) = 8`, `alignof(S1) = 4`.

Note: Phase 1 uses two `int` storage units,
but Phase 2 is free to merge `a`, `b`, and `c` into a single `i64` access unit
(since there are no non-bit-field barriers and 8 bytes fits in a register).
On x86_64, the LLVM type ends up as `{ i64 }`.

### Mixed Types

When bit-fields have different declared types, the storage unit size changes:

```c
struct S2 { int a:24; short b:8; };   // sizeof = 4
```

- `a` is `int` (StorageUnitSize = 32). Placed at bit 0.
- `b` is `short` (StorageUnitSize = 16, FieldAlign = 16). Current offset = 24.
  Position within a 16-bit aligned unit: 24 % 16 = 8. 8 + 8 = 16 <= 16. Fits. **Offset = 24.**

`sizeof(S2) = 4`.
The `short` bit-field overlaps into the `int`'s storage unit.
Under Itanium, storage units of different types *can* share bytes.

The `short` can also reuse space left by a smaller bit-field:

```c
struct S2b { int a:16; short b:8; };   // sizeof = 4
```

- `a` is `int` (StorageUnitSize = 32). Placed at bit 0.
- `b` is `short` (StorageUnitSize = 16, FieldAlign = 16). Current offset = 16.
  Position within a 16-bit aligned unit: 16 % 16 = 0. 0 + 8 = 8 <= 16. Fits. **Offset = 16.**

Here `b`'s 16-bit storage unit (bits 16–31) falls entirely within `a`'s 32-bit storage unit.

> Under Microsoft ABI, `sizeof` is 8: the type size change from `int` to `short` forces a new storage unit.

This overlapping extends to non-bit-field members too.
A non-bit-field can be allocated within the unfilled bytes of a preceding bit-field's storage unit:

```c
struct S2c { uint16_t first:8; uint8_t second; };   // sizeof = 2
```

- `first` is `uint16_t:8`. Placed at bit 0. Uses 8 bits of a 16-bit storage unit (bytes 0–1).
- `second` is a non-bit-field `uint8_t`. The bit-field state resets, but DataSize is only 1 byte.
  `second` (alignment 1) goes at **byte 1** (bit 8) — inside `first`'s storage unit.

Note that this overlapping means a write to `first` via its access unit could touch byte 1 where `second` lives.
Phase 2 must ensure the access units don't clobber each other (see [Hard constraints](#itanium-merging-algorithm)).

> Under Microsoft ABI, `sizeof` is 4: `first` gets a full `uint16_t` unit (2 bytes), and `second` starts at byte 2 instead of byte 1.

### Non-bit-field After Bit-field

When a non-bit-field field cannot fit within the remaining bytes,
it resets the bit-field state and unfilled bits become padding:

```c
struct S3 { int a:10; int b:6; char c; int d:6; };   // sizeof = 4
```

- `a` at bit 0, `b` at bit 10 — both fit in the first `int` storage unit.
  `a + b` occupy 16 bits = 2 bytes, leaving 16 bits unused in the 32-bit storage unit.
- `c` is not a bit-field. It resets `UnfilledBitsInLastUnit` to 0.
  `c` (a `char`, alignment 1) goes at **byte 2** (bit 16).
  A subsequent bit-field could have used bits 16–31, but the non-bit-field `c` claims byte 2.
- `d` is a new `int` bit-field. Current bit offset = 24 (byte 3).
  Position = 24 % 32 = 24. 24 + 6 = 30 <= 32. Fits. **Offset = 24.**

`sizeof(S3) = 4`.

> Under Microsoft ABI, `sizeof` is 12: `a`+`b` get a full `int` unit (4 bytes), `c` starts at byte 4, and `d` gets a new `int` unit at byte 8.

### Bit-field After Non-bit-field

The overlap works in the other direction too.
When a bit-field follows a non-bit-field, its storage unit can encompass the preceding bytes:

```c
struct NB { char a; int b:4; };   // sizeof = 4
```

- `a` is a `char` at byte 0. DataSize = 1 byte.
- `b` is `int:4`. FieldOffset = 8, FieldAlign = 32, StorageUnitSize = 32.
  Position: `8 & 31 = 8`. `8 + 4 = 12 ≤ 32`. Fits. **Offset = 8.**

`b`'s 4-byte `int` storage unit (bytes 0–3) encompasses `a` at byte 0.
No padding is inserted — the core rule only cares whether the field fits within an aligned unit,
not whether that unit overlaps earlier non-bit-field storage.

> Under Microsoft ABI, `sizeof` is 8: `b`'s `int` unit starts at byte 4, after `a` is padded to `int` alignment.

### Attributes and Pragmas

Several attributes and pragmas alter the placement rules.
They all work by changing `FieldAlign`.

**`packed`** — sets `FieldAlign = 1` (bit-granular packing).
Bitfields pack at the next available *bit* with no alignment constraint.

```c
struct [[gnu::packed]] P { int x:4, y:30, z:30; };
// 4 + 30 + 30 = 64 bits = 8 bytes. sizeof = 8.
```

> Under Microsoft ABI, `sizeof` is 12: each bit-field must fit within a single `int` unit, so `x`, `y`, and `z` each get their own 4-byte unit.

`packed` can also be applied to individual fields:

```c
struct P2 { short a:8; [[gnu::packed]] int b:30; };   // sizeof = 6, b at bit 8
// Without packed on b: b at bit 32, sizeof = 8
```

Without packed, `b`'s FieldAlign is 32, so it doesn't fit in `a`'s `short` storage unit and starts a new `int` unit at bit 32.
With packed, `b`'s FieldAlign drops to 1, so it packs immediately after `a` at bit 8.

**`#pragma pack(N)`** — caps `FieldAlign` at `N * 8` bits and suppresses the padding-insertion test
(`AllowPadding = false`, so the overflow check is skipped — the field is placed at the current offset without rounding up).

```c
#pragma pack(1)
struct PP { char a; int b:4; int c:28; char s; };   // sizeof = 6
#pragma pack()
```

`b` packs at bit 8 by the normal core rule — `(8 & 31) + 4 = 12 ≤ 32`, so it fits.
Without `#pragma pack`, `c:28` at bit 12 would fail the same check — `12 + 28 = 40 > 32` —
and round up to bit 32. With `#pragma pack(1)`, `AllowPadding` is false,
so the overflow check is skipped and `c` stays at bit 12.
Total: `a`(8) + `b`+`c`(32) + `s`(8) = 48 bits = 6 bytes.

**`aligned(N)`** — forces minimum alignment.
Overrides `packed`, but is itself overridden by `#pragma pack`.

```c
struct A { char a; [[gnu::aligned(16)]] int b:1; char c; };
// b aligned to 16 bytes = bit 128. c at byte 17. sizeof = 32, alignof = 16.
```

**Precedence** (for non-zero-width bit-fields): `#pragma pack` > `aligned` attr > `packed` attr > natural alignment.

### Zero-width Bitfields

`T : 0` rounds up to `alignof(T)`, acting as a separator.
Subsequent fields start in a new storage unit.

```c
struct Z { char x; int : 0; char y; };
// x86:         y at offset 4, sizeof = 5, alignof = 1
// ARM/AArch64: y at offset 4, sizeof = 8, alignof = 4
```

On most targets, anonymous bit-fields don't contribute to struct alignment.
But on AArch32/AArch64 (with `useZeroLengthBitfieldAlignment()`),
zero-width bit-fields *do* raise the struct's alignment.

Zero-width bit-fields are exempt from both `packed` and `#pragma pack` — they always round up to `alignof(T)`.

### Microsoft ABI Differences

Clang uses the Microsoft layout rules in two situations:
targeting a Windows triple (e.g. `x86_64-windows-msvc`), which uses `MicrosoftRecordLayoutBuilder`;
or applying `__attribute__((ms_struct))` to individual structs on any target,
which activates the `IsMsStruct` path inside `ItaniumRecordLayoutBuilder`.
GCC documents the rules under [`TARGET_MS_BITFIELD_LAYOUT_P`](https://gcc.gnu.org/onlinedocs/gccint/Storage-Layout.html#:~:text=TARGET_MS_BITFIELD_LAYOUT_P).

The Microsoft ABI uses a fundamentally different layout strategy.
While Itanium packs bit-fields into overlapping storage units of potentially different types,
Microsoft allocates a **complete** storage unit of the declared type, then parcels bits among successive bit-fields **of the same type size**.

The key differences:

**Type size changes force a new storage unit.**
In the GCC documentation's wording:
"a bit-field won't share the same storage unit with the previous bit-field if their underlying types have different sizes,
and the bit-field will be aligned to the highest alignment of the underlying types of itself and of the previous bit-field."
Itanium would let them overlap.

```c
struct Itn { int a:24; short b:8; };                             // sizeof = 4
struct __attribute__((ms_struct)) MS { int a:24; short b:8; };   // sizeof = 8
```

Under Itanium, `b`'s `short` storage unit overlaps into `a`'s `int` unit — everything fits in 4 bytes.
Under Microsoft, the type size changes from 4 to 2, so `b` gets its own storage unit.
The `int` unit (4 bytes) plus the `short` unit (2 bytes, padded to 4 for alignment) gives 8 bytes.
Note that the rule is about type *size*, not type identity — `int a:24; unsigned b:8` share a unit because both types are 4 bytes.

Each unit is discrete — this is a direct consequence of the type size rule.

**Zero-width bit-fields are ignored unless they follow a non-zero-width bit-field.**
(`MicrosoftRecordLayoutBuilder::layoutZeroWidthBitField`.)
GCC's documentation: "zero-sized bit-fields are disregarded unless they follow another nonzero-size bit-field."
When honored, they terminate the current run and affect the struct's alignment.

```c
// MS mode:
struct MS_ZW1 { long : 0; char bar; };                       // sizeof = 1 (no preceding bit-field)
struct MS_ZW2 { char foo; int : 0; char bar; };              // sizeof = 2 (preceding non-bit-field doesn't count)
struct MS_ZW3 { int : 0; long : 0; char bar; };              // sizeof = 1 (zero-width doesn't count either)
struct MS_ZW4 { char foo : 4; int : 0; char bar; };          // sizeof = 8 (non-zero-width bit-field — honored)
struct MS_ZW5 { long : 0; char foo : 4; int : 0; char bar; };  // sizeof = 8 (first ignored, second honored)
```

**Alignment = type size.**
The alignment of a fundamental type always equals its size —
`alignof(long long) == 8` even on targets where the natural alignment is 4 (like Darwin PPC32).

**Unions.**
ms_struct ignores all alignment attributes in unions.
All bit-fields use alignment 1 and start at offset 0.

## Phase 2: Access Units

LLVM IR has no bit-field concept. To access a bit-field, the Clang-generated IR must:

1. Load an integer from memory (the **access unit**)
2. Mask and shift to extract or insert the bit-field's bits
3. Store the integer back

The access unit is the LLVM type that gets loaded and stored.
Choosing it well matters:

- Too narrow means multiple memory operations for adjacent bit-field writes;
- Too wide means touching memory unnecessarily or clobbering adjacent data.

Implementation: `CGRecordLowering::accumulateBitFields` (`clang/lib/CodeGen/CGRecordLayoutBuilder.cpp`).

### Itanium: Merging Algorithm

**Hard constraints** — an access unit must never:

1. **Overlap non-bit-field storage.**
   The C memory model allows non-bit-field members to be accessed from other threads.
   A load/store of the access unit must not touch bytes belonging to other members.
2. **Cross a zero-width bit-field** at a byte boundary.
   Zero-width bit-fields define memory location boundaries — they are barriers.
3. **Extend into reusable tail padding.**
   In C++, a derived class may place fields in a non-POD base class's tail padding.
   The access unit must not overwrite those bytes.

**Soft goals** — subject to the hard constraints, access units should be:

- **Power-of-2 sized** (1, 2, 4, 8 bytes).
  Non-power-of-2 sizes (e.g., 3 bytes) get lowered as multiple smaller loads plus bit manipulation.
- **No wider than a register.** Avoids multi-register loads.
- **Naturally aligned** (on strict-alignment targets).
  Avoids the compiler synthesizing unaligned access sequences.
- **As wide as possible** within the above.
  Fewer, wider accesses let LLVM combine adjacent bit-field writes into one read-modify-write.

**The algorithm: spans then merging.**

*Step 1 — Spans.*
Bitfields that share a byte are inseparable.
They form a minimal "span" that must be in the same access unit.
A span is a maximal run of bit-fields where each successive one starts mid-byte.

Spans break at byte-aligned boundaries and at zero-width bit-field barriers.
A field mid-byte is unconditionally part of the current span — step 2 never sees it as a merge point.

*Step 2 — Merge.*
Starting from each span, try to widen the access unit by incorporating the next span.
Accept the merge if the combined unit:

- Fits in one register (`<= RegSize`)
- Is power-of-2 and naturally aligned (on strict-alignment targets)
- Doesn't cross a barrier (zero-width bit-field or non-bit-field storage)
- The natural `iN` type fits before the limit offset

Track the best candidate and install it when merging can't improve further.

**Access unit representation.**

Clang represents each access unit as either an integer type `iN` or an array type `[N x i8]` (see `CGRecordLowering::accumulateBitFields`).
`iN` is preferred — it generates a single load/store instruction.
But LLVM's `iN` types have allocation sizes rounded up to powers of 2 (`DataLayout.getTypeAllocSize`).
For example, `i24` has allocation size 4 bytes.

If that rounded-up size would extend past the next field or past reusable tail padding,
the access unit is **clipped** to `[N x i8]`, which has an exact byte count.
Clang assumes clipped for each new span (`BestClipped = true`)
and sets it to false only when the natural `iN` fits within the available space (`BeginOffset + TypeSize <= LimitOffset`).

```c
// Tail padding reuse (C++)
struct A { int x:24; ~A(); };      // non-POD: DataSize=3, Size=4
struct B : A { char c; };          // c at offset 3, in A's tail padding

// i24 allocates 4 bytes, but byte 3 belongs to B::c.
// Access unit for x is clipped to [3 x i8].
```

**Strict vs cheap unaligned.**
On targets with cheap unaligned access (x86, AArch64 without `+strict-align`), alignment checks are skipped — spans merge freely up to register width.
On strict-alignment targets (e.g. `-mstrict-align`),
a merge is rejected if the combined access unit would not be naturally aligned at its offset within the struct.

```c
struct Align { char x; short a:12; short b:4; char c:8; }; // sizeof = 6

// AArch64 -mno-strict-align:  %struct.S = type <{ i8, i8, i32 }>
//   → a+b+c merged into one i32 at offset 2 (unaligned, but cheap)
// AArch64 -mstrict-align:     %struct.S = type { i8, i16, i8 }
//   → a+b merged
//   → +c rejected; a+b stay as i16, c gets its own i8
```

**`-ffine-grained-bit-field-accesses`.**
This Clang flag disables merging entirely.
Each span becomes its own access unit — no adjacent spans are combined.
For example:

```c
struct S4 { unsigned long f1:28, f2:4, f3:12; };
// Default:        %struct.S4 = type { i64 }       — spans merged into one access unit
// Fine-grained:   %struct.S4 = type { i32, i16 }  — each span kept separate
```

[The flag is incompatible with sanitizers](https://reviews.llvm.org/D36562) and is automatically disabled (with a warning) when any sanitizer is active.

Returning to S3:

```c
struct S3 { int a:10; int b:6; char c; int d:6; };
```

Phase 1 assigned: `a`@0, `b`@10, `c`@16 (byte 2), `d`@24 (byte 3).

Phase 2 sees two bit-field runs (separated by non-bit-field `c`):

*Run 1: `a` and `b`* (bits 0–15, bytes 0–1).
They share byte 1 (bits 8–15), so they form one span.
The span covers 2 bytes.
The natural type `i16` fits exactly — no clipping needed.
Access unit: `i16`.

*Run 2: `d`* (bits 24–29, byte 3).
Single span, 6 bits in 1 byte.
Access unit: `i8`.

The resulting LLVM struct type:

```
%struct.S3 = type { i16, i8, i8 }
                    a,b   c   d
```

To read `a`, codegen loads the `i16`, extracts bits 0–9.
To read `b`, it loads the same `i16`, extracts bits 10–15.
Neither load touches `c`.

**When clipping is needed.**
Widen the bit-fields so `a + b` no longer fits in 2 bytes:

```c
struct S3w { int a:14; int b:10; char c; int d:6; };
```

Phase 1 assigned: `a`@0, `b`@14, `c`@24 (byte 3), `d`@32 (byte 4). `sizeof(S3w) = 8`.

*Run 1: `a` and `b`* (bits 0–23, bytes 0–2).
The span covers 3 bytes.
The natural type `i24` has allocation size 4 bytes — but byte 3 belongs to `c`.
The access unit is **clipped to `[3 x i8]`**.

*Run 2: `d`* (bits 32–37, byte 4). Access unit: `i8`.

```
%struct.S3w = type { [3 x i8], i8, i8, [3 x i8] }
                      a,b       c    d    padding
```

**Endianness.**

Access unit selection is endianness-agnostic — spans, merging, and clipping all work in byte offsets from the start of the struct.
Endianness matters only when codegen emits the shift/mask sequence to extract or insert a bitfield within its access unit.

LLVM loads an access unit as a single integer.
On little-endian, bit 0 of the integer corresponds to the lowest-addressed byte's LSB — bitfield offsets from Phase 1 can be used directly as shift amounts.
On big-endian, bit 0 of the integer corresponds to the highest-addressed byte's MSB, so the bit numbering within the loaded integer is reversed.

Clang handles this in `setBitFieldInfo` (`CGRecordLayoutBuilder.cpp`):

```cpp
Info.Offset = (unsigned)(getFieldBitOffset(FD) - Context.toBits(StartOffset));
// ...
if (DataLayout.isBigEndian())
  Info.Offset = Info.StorageSize - (Info.Offset + Info.Size);
```

The little-endian offset counts up from the LSB; the big-endian offset is mirrored to count down from the MSB.
`EmitLoadOfBitfieldLValue` (`CGExpr.cpp`) then uses `Info.Offset` uniformly — it right-shifts by `Offset` and masks to `Size` bits, which works for both endiannesses because the flip was already baked into `Offset`.

### Microsoft: Discrete Access Units

Microsoft ABI's codegen is simple: each bit-field gets an access unit of its declared type.
Adjacent bit-fields of the same type size share one access unit.
Zero-width bit-fields and type-size changes break runs.
There is no complex merging — the Phase 1 storage units *are* the access units.

Contrast S3 under both ABIs:

```c
struct S3 { int a:10; int b:6; char c; int d:6; };
```

```
Itanium:   %struct.S3  = type { i16, i8, i8 }        // a,b merged into i16, d is i8
Microsoft: %struct.MS3 = type { i32, i8, i32 }       // a,b share i32 unit, d gets own i32
```

Itanium's Phase 2 merges `a` and `b` into the tightest access unit that covers both (`i16`),
and clips or shrinks to avoid touching `c`.
Microsoft uses the full declared type (`int` = `i32`) for each storage unit — no merging, no clipping.

Similarly for mixed types:

```c
struct S2 { int a:24; short b:8; };
```

```
Itanium:   %struct.S2  = type { i32 }                 // a and b merged into one i32
Microsoft: %struct.MS2 = type { i32, i16 }            // separate units: i32 for a, i16 for b
```

Itanium merges `a` and `b` into a single `i32` since they share the same 4 bytes.
Microsoft gives each its own access unit matching the declared type.

## Conclusion

Phase 1 decides *where* bits go — it's specified by the ABI and determines `sizeof` and `alignof`.
Phase 2 decides *how* to access them — it's a compiler optimization that affects codegen but not the binary layout.
They answer different questions and often produce different-sized units.
The storage unit for a bit-field is determined by its declared type;
the access unit is determined by what's safe and efficient to load.
