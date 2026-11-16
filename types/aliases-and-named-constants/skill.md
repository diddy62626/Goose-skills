---
name: goose-aliases-and-named-constants
title: Type aliases and named constants
category: types
tags: [goose, type-alias, named-constant, let-global, adapt]
triggers: [goose type alias, named constant, let SIZE, constant expression, array size]
description: >
  `type Name = T;` is the same type, not a new one. A `let`/`const` global
  with a constant initializer is a named constant that adapts like a literal:
  usable as any integer type it fits, an array size, a fill count, or a match
  arm.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#type-aliases
---

# Type aliases and named constants

## `type Name = T;` — an alias, not a new type

```goose
type Id = u32;
type Names = u8[..16][>..];

let a: Id = 7;
let b: u32 = a;              // same type: fine
var ns: Names = [];
ns.push("goose");
```

Nominal types come only from `struct`/`enum` declarations. Aliases shine for
index types: `type NodeId = u32 in nodes;` reads better at every use.

## Named constants

A `let` or `const` **global** with no written type and a constant initializer
is a named constant. An integer named constant can be used exactly as its
literal could: it adapts to any integer type its value fits, serves as an
array size or fill count, and names a value in a `match` arm.

```goose
let SIZE = 16;
let HALF = SIZE / 2;
var grid: u8[SIZE] = [0; SIZE];
fn main() {
    let b: u8 = SIZE;                 // adapts, as the literal 16 would
    grid[HALF] = 1;
    print(match 8 { HALF => "half", _ => "other" });
}
```

A global **with** a written type (`let SIZE: i64 = 16;`), a local `let`, and
any `var` are ordinary values of their type and do not adapt to other types.

## What counts as a constant expression

The `k` of `T[k]`/`T[..k]` and the count of `[v; n]` accepts: literals, named
constants, arithmetic on them, and the `.len` of a global whose length is
fixed by its declaration (`const names = ["a", "bc"]; var counts:
i64[names.len];`). A type parameter is a type, never a size:
`fn f<N>(a: i64[N])` is an error. A local hiding a global's name is not a
constant either.

## Globals vs constants

- `var log: u8[>..];` (type only) — starts as the type's default.
- `let NAMES = ["ada", "grace"];` — with a constant initializer: static
  data; `const` globals of flat fixed type live in static data, shared and
  read-only.
- Initializers may call functions and use globals from imported modules
  (imported files initialize first, then declaration order within a file).

## Float constants adapt too

`let PI = 3.141592653589793;` used at both widths — an expression of float
literals and integers adapts to the destination's float type (see
scalars-and-arithmetic).

## Related skills

- [scalars-and-arithmetic](scalars-and-arithmetic/skill.md)
- [index-types](../data-modeling/index-types/skill.md)
