---
name: goose-structs
title: Structs — literals, field defaults, qualifiers, generic structs
category: types
tags: [goose, struct, literal, defaults, pad, generic, nominal-types]
triggers: [goose struct, struct literal, field default, pad field, generic struct, nominal type]
description: >
  Struct declarations introduce nominal types; fields are packed with no
  padding. Literals by name or position, `..` for defaults, field defaults
  evaluated per construction, let/const/pad field qualifiers, variable-size
  and varint fields, the one resizable tail, root structs, generic structs,
  equality and memberwise arithmetic.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#8-structs
---

# Structs

```goose
struct Unit { kind: i32, hp: i32 = 100, x: f32, y: f32, name: u8[] }
```

An ordinary struct is its fields in declaration order, **packed**: no
padding, unaligned access assumed cheap. A struct of an `i8[3]` and an `i32`
is 7 bytes. Two structs with the same fields are different types (nominal).

## Literals

Named fields or positional (in declaration order); the two forms do not mix.
Fields with a declared default may be left out. A literal ending in `..` may
leave out *any* field, which then takes its type's default (zero / false /
null / empty array). An optional field with no default is null when left out.

```goose
let a = Unit { kind: 1, x: 1.5, y: 2.5, name: "scout" };   // hp 100
let b = Unit { 2, 50, 0, 0, "" };                          // positional, all given
let c = Unit { kind: 3, .. };                              // hp 100, x 0, y 0, name ""
```

Named initializers may be out of declaration order only when the compiler can
tell reordering changes nothing (no calls with effects, indexing, loops).
Construction always proceeds in declaration order, directly into each field.

## Field defaults

Evaluated at each construction that leaves the field out, in field order,
among the explicit initializers. May name globals and call functions — never
the constructor's locals or sibling fields.

```goose
var serial = 0;
fn next() -> i64 { serial += 1; serial }
struct Rec { id: i64 = next(), tag: u8[..4] = "none", score: f64 = 1.0 }

let b = Rec { score: 2.0 };        // id 2 (fresh), tag "none"
let c = default<Rec>();            // fixed-size T from declared defaults
```

## Field qualifiers

- `let f: T` — not assigned after construction.
- `const f: T` — contents read-only too.
- `pad n` — n unused bytes; bare `pad` aligns the next field to its scalar
  width (for foreign layouts). Pad bytes are never read; `==` ignores them.

```goose
struct Hdr { magic: u8[4], pad, size: u32, let kind: u8, const name: u8[..8] }
```

## Variable-size fields and varint

Fields sit **inline** in declaration order. A struct with a `varint` or
`u8[]` field is variable-class; fields after it are reached by walking. A
`varint` field is written only at construction; reads give `i64`.

```goose
struct Item  { sku: u8[varint], qty: varint, cents: varint }
struct Order { id: varint, customer: u8[varint], items: Item[varint] }
// That order is 29 bytes with no pointer in it.
```

A variable-class field is frozen once built: to change one, rebuild the
container (or use `u8[..k]` for a fixed-capacity, assignable string).

## The resizable tail

An ordinary struct's **last** field may be a resizable array, which makes the
struct resizable (a local/global, by-value parameter, result — never an
element or middle field):

```goose
struct Stack { name: u8[..8], items: i64[>..<] }
```

For several independently growing fields, use a `root struct` instead.

## Root structs

`root struct Workspace<T> { items: T[>..] = [], ... }` — each field has
independent storage; owner must be a local/global variable; functions take it
by reference (`Workspace<T>&`). See the root-structs skill.

## Generic structs

```goose
struct Pair<T> { a: T, b: T }
let p = Pair<i64> { 1, 2 };
let q = Pair { a: 0.5, b: 1.5 };       // T inferred
```

Monomorphized; instantiations must be finite (a field naming the struct again
with a *larger* type argument is rejected).

## Equality and arithmetic

`==`/`!=` compare memberwise, excluding pads; reference/slice *fields*
compare by identity (address and length). The arithmetic operators apply
memberwise (see scalars-and-arithmetic). Ordering operators exist on numbers
only.

## Constructors

Struct literals do the job. For invariants, write `make_thing(...)` returning
the struct — it is built in place at the caller.

## Related skills

- [enums-adts](enums-adts/skill.md)
- [size-classes-and-placement](../memory-model/size-classes-and-placement/skill.md)
- [root-structs](../data-modeling/root-structs/skill.md)
- [inline-nested-data](../data-modeling/inline-nested-data/skill.md)
