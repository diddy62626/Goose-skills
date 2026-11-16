---
name: goose-casts-and-conversions
title: Casts and conversions — implicit widening, `as`, `as!`
category: types
tags: [goose, cast, conversion, as, narrowing, warnings]
triggers: [goose cast, goose as, goose as!, narrowing conversion, redundant cast warning]
description: >
  What converts implicitly (widening, int→float), what needs `as` (debug
  range-checked) or `as!` (never checked), float↔integer conversion rules,
  redundant-cast warnings, and comparisons with u64.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#conversions
---

# Casts and conversions

Within a kind, a conversion the compiler can prove value-preserving is
implicit; anything that could lose a bit or flip a sign needs a cast.

## The table

| Conversion | How |
|---|---|
| to a wider integer of the same signedness (`i8 → i32`, `u16 → u64`) | implicit |
| unsigned to a strictly wider signed type (`u8 → i16`, `u32 → i64`) | implicit |
| `f32 → f64` | implicit |
| any integer (incl. `u64`) to `f32`/`f64` | implicit, rounding to nearest |
| a literal or named constant into any type its value fits | implicit |
| any integer but `u64` into a `varint` store | implicit |
| narrowing, same-width signedness change, signed→unsigned, `u64`→signed, `f64→f32`, float→integer | `x as T` or `x as! T` |

## `as` vs `as!`

- `x as T` converts between any two numeric types. **Range-checked in a
  debug build** (`--debug`): aborts if the value changed, naming values and
  line. Truncates/wraps/rounds toward zero silently in release.
- `x as! T` never checks, not even in debug: for when losing bits is
  intended.

```goose
let big = 300;
let n: u8 = big as! u8;       // 44, by design
let f = 2.75;
let i = f as i64;             // 2 (truncate toward zero)
let neg = -1;
let u = neg as! u32;          // 4294967295
```

Float→integer is fully defined: truncate toward zero, then wrap into the
target's width; NaN gives 0. Conversions **to** a float are never checked in
either form.

## Redundant casts warn

A cast that changes nothing gets a warning, so the compiler points out casts
the rules make unnecessary:

```goose
// b: u8
let w: i64 = b as i64;   // warning: redundant `as i64`: a u8 converts to i64 implicitly here
```

A cast that *decides* something gets no warning: `let id = b as i32;` (picks
a type), `(a as i64) * b` (picks an operation's width), or how a float is
rounded.

## Comparing u64 with signed values

`u64` has no signed type to widen into. A comparison between `u64` and a
signed operand is allowed only when the signed operand is *syntactically*
known non-negative: a non-negative literal, a `.len`/`.cap`, or a `let` bound
to one of those (and not also bound to a writable reference). Otherwise cast.
The comparison is then a single unsigned one.

```goose
let h: u64 = 18446744073709551615;
var xs: i64[>..] = [1, 2, 3];
let n = xs.len;
print(h > 5, " ", h > xs.len, " ", h > n);   // all allowed
```

## Casts to index types

`x as (uN in a)` aborts in every build unless `0 <= x < a.len` — a bounds
check moved to this point (see index-types). `as!` cannot make one, and a
literal never adapts to one.

## Integer ↔ float ergonomics

Integers become floats implicitly (`2` works where `2.0` would); a float
becomes an integer only through a cast. An integer argument converts to both
float widths equally, so libm calls must name one: `sqrt(n as f64)`.

## Related skills

- [scalars-and-arithmetic](../scalars-and-arithmetic/skill.md)
- [index-types](../../data-modeling/index-types/skill.md)
- [bounds-check-elimination](../../performance/bounds-check-elimination/skill.md)
