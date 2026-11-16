---
name: goose-scalars-and-arithmetic
title: Scalars, numeric literals, and the arithmetic rules
category: types
tags: [goose, integers, floats, literals, arithmetic, overflow, euclidean, varint, bitwise]
triggers: [goose int types, goose float, euclidean modulo, u8 arithmetic, signed overflow, varint, literal adapts]
description: >
  Integer and float types, width rules ("arithmetic happens at the operands'
  own width"), literal adaptation, the unification rules for mixed operands,
  Euclidean %, truncating /, overflow behavior per build, bitwise operators,
  memberwise arithmetic on structs, and the varint storage type.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#3-scalars-and-arithmetic
---

# Scalars and arithmetic

## The scalar types

`i8 i16 i32 i64`, `u8 u16 u32 u64`, `f32 f64`, `bool` (one byte), and
`varint` (a variable-length integer *storage* type for fields and elements;
reads give an `i64`, writes happen only at construction). `i64` is the
conventional type for counts, sizes, indices: it is what `.len`, integer
literals and builtins produce.

## Literals have no type of their own

A literal adapts to any integer type whose range holds it, and is an `i64`
where nothing constrains it:

```goose
let byte: u8 = 255;           // fine; 256 would be a compile error
let answer = 42;              // an i64
let wrapped: u8 = byte + 1;   // an 8-bit add: 0 (unsigned wraps)
let widened = byte + answer;  // u8 widens into i64: 297
```

Float literals adapt likewise: `n * 0.5` is computed in `f32` when passed to
an `f32` parameter, and is `f64` in `let h = n * 0.5;`. An `if`/`match` whose
branches are all literals adapts to its context the same way.

Hexadecimal `0xFF` (full unsigned 64-bit range accepted; above `i64.max` is a
`u64` constant), character literals `'a'` (the byte's value), digit
separators `1_000_000`, C99 hex floats `0x1.8p3`. No binary literals — use
hex or shifts.

## Mixing widths: the unification rules, in order

1. Equal types → that type.
2. A literal adapts to the other operand's type (error if it does not fit).
3. Exactly one operand widens implicitly into the other's type → the wider
   type (`u8 + i64` is an `i64` add).
4. An integer meeting a float converts to the float's type (`i32 * f32` is
   `f32`).
5. Anything else is a compile error asking for a cast: same-width
   signed/unsigned (`u32 + i32`), or `u64` with anything signed. **Nothing is
   silently promoted to 64 bits.**

Shifts are the exception: `x << k` keeps `x`'s type, count masked to width.
`>>` shifts in zeroes unsigned, copies the sign bit signed. Bitwise operators
bind tighter than comparisons: `x & mask != 0` is `(x & mask) != 0`.

## `%` is Euclidean

The result lies in `[0, |b|)` and is never negative, at every integer type.
So `x % n` is always a valid index into a length-`n` array, and the compiler
drops the bounds check on it.

```goose
print(-7 % 3, " ", 7 % -3, " ", -7 / 3);    // 2 1 -2
```

Integer `/` truncates toward zero, so `/` and `%` are not a matching pair for
negative dividends. Import `std` for `rem(a, b)` (pairs with `/`) and
`div_euclid(a, b)` (pairs with `%`):

```goose
import std;
print(-7 / 3, " ", rem(-7, 3));             // -2 -1
print(div_euclid(-7, 3), " ", -7 % 3);      // -3 2
```

## Overflow

- Unsigned arithmetic wraps modulo 2^width **in every build**, by definition
  (hashes and PRNGs rely on it).
- Signed overflow aborts in debug (`--debug` / `GS_DEBUG=1`), wraps in
  release.
- Division or remainder by zero always aborts; so does `i64.min / -1`.

```goose
var h: u64 = 14695981039346656037;
for c in "goose" { h = (h ^ c) * 1099511628211; }   // FNV-1a, wrapping by design
```

## `bool`

Comparisons produce it; `if`, `while`, `guard` require it. **No
integer-to-bool coercion**: `if n { … }` is an error — write `if n != 0`.
`&&` and `||` short-circuit.

## Memberwise arithmetic on structs and fixed arrays

`+ - * / %` and unary `-` apply memberwise to two values of the same struct
or fixed-array type whose scalar leaves are all integers or all floats; `*`
and `/` also take one scalar operand in either position. This is how the
library's vector types work; there is no operator overloading.

```goose
struct V3 { x: f32, y: f32, z: f32 }
var c = V3 { 1, 2, 3 } + V3 { 0.5, 0.5, 0.5 };
c *= 2;
let sum = [1, 2, 3] + [10, 20, 30];     // [11, 22, 33]
```

## Float printing and `%` on floats

A float prints in the shortest form that reads back as the same value of its
own type (`1.0`, `0.1`, `inf`, `-inf`, `nan`). Float `%` is C's `fmod`
(`-7.0 % 3.0` is `-1.0`).

## varint

A LEB128 integer field: 1 byte up to 63, two up to 8191, up to ten bytes;
small negatives are as short as small positives. Written only when its
containing value is constructed; reads give an `i64`. Use it in records that
are built once and walked.

## Related skills

- [casts-and-conversions](casts-and-conversions/skill.md)
- [aliases-and-named-constants](aliases-and-named-constants/skill.md)
- [common-pitfalls](../idioms/common-pitfalls/skill.md)
