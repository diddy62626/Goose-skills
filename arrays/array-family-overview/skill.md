---
name: goose-array-family-overview
title: The array family — eight kinds, one table, which to pick
category: arrays
tags: [goose, arrays, T[k], resizable, growable, slices, metadata]
triggers: [goose array types, goose vector, T[>..], T[>..<], T[..k], which array, array family]
description: >
  Goose has no single array type: T[k] fixed, T[] frozen, T[varint],
  T[..k]/T[..] limited (inline capacity), T[>..] grow-only resizable (the
  workhorse), T[>..<] grow-shrink. All store [metadata][elements...] inline,
  packed, never a pointer to a separate block. Members (.len/.cap/.push/...)
  and the decision guide.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#5-the-array-family
---

# The array family

There is no single "array type". The family differs only in what can happen
to their size, and every member stores `[metadata][elements…]` **inline and
packed**, never as a pointer to a separate block.

## The kinds

| Type | Class | What it is | Metadata |
|---|---|---|---|
| `T[k]` | fixed | `k` elements, `k` a compile-time constant | none |
| `T[]` | variable | sized at construction, frozen after | a `u32` length |
| `T[u8]`…`T[u64]` | variable | same, length of that width | that length |
| `T[varint]` | variable | same, LEB128 length | 1 byte up to 127 elements |
| `T[..k]` | fixed | capacity `k` inline; 0..k elements | a length, as small as `k` allows |
| `T[..]` | variable | capacity chosen at construction | a capacity and a length |
| `T[>..]` | resizable | **grow-only** — the workhorse | a length, kept by the owner |
| `T[>..<]` | resizable | **grow-shrink** — stacks, queues, heaps | a length, kept by the owner |

```goose
var fixed: i32[4] = [1, 2, 3, 4];
let zeros: f64[8] = [0.0; 8];        // fill form
let frozen: i64[] = [7, 8, 9];       // sized once, never changes
var small: u8[..8] = [];             // inline anywhere a fixed value can
small.push('h'); small.push('i');
print(small.len, " of ", small.cap); // 2 of 8
var sized: i64[..] = [..16];         // capacity 16 chosen now, length 0
var grow: i64[>..] = [];             // the default growable array
var stack: i64[>..<] = [];           // the one that pops
```

## Which one to reach for

- **`T[>..]` is the default**: arenas, pools, string builders, tree storage,
  scratch refilled between phases. "Grow-only" guarantees references into it
  stay valid. It can still `pop`/`clear` where the compiler can prove
  nothing points into it.
- **`T[>..<]`** for a stack, queue or heap (`heap_push`/`heap_pop` from
  `std`). References into it may live in variables only, never in storage.
- **`T[..k]` / `T[..]`** when the array must sit inline inside something
  else: a 16-byte name inside a struct, a small list inside an array element.
- **`T[]` / `T[varint]`** for finished data: a string in a record, a node in
  a pool, built once then read.
- **`T[k]`** for a fixed shape: a 4x4 matrix, a 256-entry table.

Only `[>..]` and `[>..<]` are "resizable"; the one-per-stack rule applies
only to those two. The others can go anywhere a value of their class can.

## Members (UFCS applies: `a.push(v)` is `push(a, v)`)

| Member | On | Meaning |
|---|---|---|
| `.len -> i64` | every kind and slices | element count |
| `.cap -> i64` | `T[..k]`, `T[..]` | the capacity (never changes) |
| `.push(v) -> T&` | limited, both resizables | one element, built in place; returns a reference to it |
| `.append(src)` | limited, both resizables | elements of an array/slice/literal copied in |
| `.move_from(src)` | slices of fixed-size elements | overlap-safe equal-length copy |
| `.pop() -> T` | limited, grow-shrink (grow-only where a shrink is allowed) | removes and returns the last element |
| `.pop_n(n)` | same | drops the last n |
| `.resize(n, v)` / `.resize(n)` | same | grows with copies of v, or shrinks |
| `.clear()` | same | empties |
| `.index_of(r) -> i64` | arrays of fixed-size elements | index of the element r refers to |
| `.range_index(s, msg?)` | same | start of a wholly contained slice, checked |
| `.check_length(n, msg?)` | grow-only, grow-shrink | validates a count against the stack reservation |

Growth always supplies element values — no operation exposes an
uninitialized slot. Exceeding a limited array's capacity aborts (no
reallocation).

## Equality

`==`/`!=` compare arrays structurally: length first, then elements. Operands
of different array kinds compare as slices whenever the slices would have the
same type (`u8[>..]` vs a string literal is fine).

## Finishing a resizable

The compiler does not work out that an array is done growing; a resizable
keeps its stack until its owner's scope ends. For finished data that should
be compact and free of the one-per-stack rule, copy into a `T[]`:
`let done: i64[] = copy(grow);`.

## Related skills

- [grow-only-resizable](grow-only-resizable/skill.md)
- [grow-shrink-stacks-queues](grow-shrink-stacks-queues/skill.md)
- [fixed-limited-frozen-arrays](fixed-limited-frozen-arrays/skill.md)
- [slices](slices/skill.md)
