---
name: goose-fixed-limited-frozen-arrays
title: Fixed, limited and frozen arrays — T[k], T[..k], T[..], T[]
category: arrays
tags: [goose, fixed array, limited array, frozen, inline capacity, fill literal, varint length]
triggers: [goose fixed array, T[..k], inline capacity, small string, fill [0; n], frozen array]
description: >
  Compile-time sizes (T[k]), inline-capacity arrays that live inside structs
  and array elements (T[..k], T[..]), and finished frozen data (T[],
  T[varint]). Fill literals, capacity semantics, sequential arrays, and
  compile-time size expressions.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#the-array-family
---

# Fixed, limited and frozen arrays

## Fixed arrays `T[k]`

`k` elements, `k` a compile-time constant, no metadata. A fixed array is a
fixed-size value: it copies silently, lives in structs, elements, payloads,
and can be assigned wholesale (`xs = [4, 5, 6, 7];`).

```goose
var fixed: i32[4] = [1, 2, 3, 4];
let tbl: i64[4] = [7; 4];             // fill literal; v evaluated once
let names: u8[..8][3] = ["a", "bb", "ccc"];   // fixed array of limited arrays
let sum = [1, 2, 3] + [10, 20, 30];   // memberwise arithmetic
```

A literal with no destination type is a `T[k]` (or `T[]` for non-fixed
elements): `[1, 2, 3]` on its own is an `i64[3]`. A literal of constants
adapts to a parameter of the shape it fits: `take([0, 0, 0, 7])` passes a
`u8[4]` to `fn take(c: u8[4])`.

## Limited arrays `T[..k]` and `T[..]`

Room for `k` elements (or a run-time-chosen capacity `[..cap]`), holding 0 to
k of them. They are **builders within their capacity**: `.push`, `.append`,
`format(p.name, "linus")`. They live inline anywhere a fixed value can —
that is their reason to exist.

```goose
var small: u8[..8] = [];
small.push('h'); small.push('i');
print(small.len, "/", small.cap);      // 2/8
var sized: i64[..] = [..16];           // capacity 16 chosen now, length 0
```

- `.cap` never changes; pushing past it **aborts** instead of reallocating.
- The stored length takes the smallest unsigned type that can count to `k`
  (a `u8[..300]` has a `u16` length). The count of a `T[u8]`-style length is
  also statically rejected or run-time aborted if it would overflow.
- References into them are allowed and **survive pops**: capacity and
  element type are fixed, so a slot once written stays a valid value of the
  element type forever. A stale reference reads whatever the slot holds now,
  never corrupt memory.
- Elements must be fixed-size.

`struct Person { name: u8[..16], age: i32 }` — a 16-byte inline string, and
an array of Persons is one block you can sort and copy as plain data.

## Frozen arrays `T[]`, `T[varint]`, `T[uN]`

Sized at construction, frozen after: built once, then read. Stored as
`[length][elements...]` inline in whatever holds them.

```goose
let frozen: i64[] = [7, 8, 9];
let packed: u8[varint] = "hi";         // 1-byte length before the bytes
var log: u8[][>..] = [];               // array of finished strings (sequential!)
```

- A variable-class field or element is **frozen once built**: to change one,
  rebuild its container. `for r in recs { r.name = "bb"; }` is an error when
  `name: u8[]` — use `u8[..32]` (assignable) instead.
- Arrays of variable-size elements (`T[]`, `T[varint]` elements) are
  **sequential**: iterate, do not index, no `pop`. Keep an index of
  references beside it for random access.

## Compile-time sizes

The `k` of `T[k]`/`T[..k]` and fill counts accept literals, named constants,
arithmetic on them, and the `.len` of a global whose length its declaration
fixes. No size parameters: `fn f<N>(a: i64[N])` is an error.

## Placement recap

| Kind | In a struct | As an element |
|---|---|---|
| `T[k]` | any position | yes |
| `T[..k]` | any position (struct stays fixed-size) | yes |
| `T[]`, `T[varint]`, `T[..]` | any position (struct becomes variable-class) | yes (array becomes sequential) |
| `T[>..]`, `T[>..<]` | last field only | never |

## Related skills

- [array-family-overview](array-family-overview/skill.md)
- [slices](slices/skill.md)
- [inline-nested-data](../data-modeling/inline-nested-data/skill.md)
- [index-for-sequential-arrays](index-for-sequential-arrays/skill.md)
