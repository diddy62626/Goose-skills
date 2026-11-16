---
name: goose-enums-adts
title: Enums (ADTs) — two storage modes, match, case functions
category: types
tags: [goose, enum, adt, match, variable-mode, fixed-mode, case-functions, polymorphism]
triggers: [goose enum, algebraic data type, Shape.., variable mode enum, match syntax, case functions, virtual dispatch]
description: >
  Enums are the only dynamic polymorphism in Goose (no inheritance, traits or
  vtables). Every ADT can be stored fixed-mode (indexable, replaceable) or
  variable-mode (packed, pointable-into). match arms, &-binders, integer
  matches, and case functions (one overload per variant — the virtual-call
  idiom without a vtable).
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#9-enums-algebraic-data-types
---

# Enums (algebraic data types)

```goose
enum Shape { Circle { r: f64 }, Rect { w: f64, h: f64 }, Dot }
```

A tag plus one variant's payload; variant payloads declare like struct fields
(same defaults and `..` rules); a payload-less variant is a plain constant.
A generic enum's variant literals give type arguments:
`Opt<i64>.Some { v: 1 }`.

## Two storage modes, chosen at the point of use

| | Fixed mode `Shape` | Variable mode `Shape..` (trailing dots) |
|---|---|---|
| Size | tag + largest payload, for every value (17 bytes here) | exactly its own variant (Dot 1, Circle 9, Rect 17) |
| Array of them | indexable | sequential: iterate only (or keep an index beside it) |
| Overwrite element with a different variant | yes, in place | never |
| Reference into a payload | never (`match` binds a copy) | yes — edit in place |
| Variable-size payloads (strings) | not allowed | allowed |

```goose
var shapes: Shape[>..] = [];                   // fixed mode
shapes[2] = Shape.Rect { w: 1.0, h: 1.0 };     // different variant, in place

var packed: Shape..[>..] = [];                 // variable mode
for s in packed {
    match s {
        Rect &r => { r.w += 1.0; },            // edits the payload inside the array
        _ => {},
    }
}
```

Allowing either variant replacement or interior references, but never both,
is what keeps both modes sound. For a tree of mostly-small nodes the memory
difference is large — variable mode is the biggest single source of the
memory savings in the benchmarks.

## `match`

```goose
fn name(s: Shape) -> u8[:] {
    match s {
        Circle c => if c.r > 1.0 { "big circle" } else { "circle" },
        Rect r => if r.w == r.h { "square" } else { "rectangle" },
        Dot => "dot",
    }
}
```

- Exhaustive: every variant has an arm, or `_` covers the rest; no variant
  named twice.
- `Circle c =>` binds the payload **by value** (read-only copy);
  `Circle &c =>` binds **by reference** — legal only for a variable-mode
  scrutinee.
- An arm may list several variants (`Circle, Dot =>`) and then binds nothing.
- `match` is an expression; arms must agree on a type; a multi-statement arm
  is a block (`Rect &r => { … },`).

**Over integers** the arms are literals, named constants, half-open ranges,
or comma lists of those; `_` is required. A name is a variant when the
scrutinee is an enum, a constant when it is an integer. First match wins; an
arm fully covered by an earlier one is an error.

```goose
match t {
    GROUND => "ground",
    SHALLOW..DEEP => "water",
    LAVA, 10..12, -3 => "hot",
    _ => "rock",
}
```

## Case functions: `match` as an overload set

One overload per variant type, called with the enum. The call dispatches on
the tag through a jump table; exhaustiveness is checked exactly as for a
`match`. This is the virtual-call idiom without the vtable:

```goose
fn area(s: Shape.Circle) -> f64 { 3.14159 * s.r * s.r }
fn area(s: Shape.Rect) -> f64   { s.w * s.h }
fn area(s: Shape.Dot) -> f64    { 0.0 }

for s in shapes { print(s, " area ", area(s)); }
```

Variant types (`Shape.Circle`) are ordinary types: value, parameter, field,
`Shape.Circle&` reference. An overload taking `Shape.Rect&` needs a
variable-mode scrutinee. Dispatch is on one parameter position; other
arguments pass through; tried only when no ordinary overload matches.

## Enums with references (trees in a pool)

```goose
enum Expr {
    Num { v: f64 },
    Add { l: Expr..&<u16>, r: Expr..&<u16> },
}
var pool: Expr..[>..] = [];
let a = pool.push(Expr.Num { v: 1.5 });
let sum = pool.push(Expr.Add { l: a, r: b });    // Add node: 5 bytes
```

## Storing shared fields

Common data in a struct, varying part in an enum field:
`struct Entity { x: f32, y: f32, hp: i32, kind: Kind }`.

## Generic code over every variant

`const for <N, V> in variants<T>` expands per variant at specialization time;
`fields<V>` walks a variant's payload — this replaces derive macros and
reflection.

## Related skills

- [structs](../structs/skill.md)
- [const-for-reflection](../../control-flow/const-for-reflection/skill.md)
- [linked-structures](../../data-modeling/linked-structures/skill.md)
- [migrating-from-cpp-rust](../../idioms/migrating-from-cpp-rust/skill.md)
