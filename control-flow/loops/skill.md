---
name: goose-loops
title: Loops — for-in, while, loop, for &x mutation, index binding
category: control-flow
tags: [goose, for, while, loop, iteration, index, mutation]
triggers: [goose for loop, iterate array, for &x, index in loop, while, loop]
description: >
  for x in range_or_array with the full form catalog: typed loop variables,
  half-open ranges, by-value vs for-&x mutation, the x,i index pair, in-place
  traversal semantics (pushes extend the walk; no shrink inside), while and
  loop, no labels.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#loops
---

# Loops

## The forms

```goose
for i in 3 { }              // 0, 1, 2
for i in 10..13 { }         // half-open: 10, 11, 12
for i: i32 in 0..n { }      // i is an i32; each bound must be, or widen to, one
for i: u8 in 0..256 { }     // every u8 — the end is one past the type's last value
for x in xs { }             // copies, for fixed-size elements; references for others
for &x in xs { x *= 2; }    // references: the mutation form
for x, i in xs { }          // with the index (an i64)
for x, i: u8 in grid { }    // the index at a narrower type where the length fits
while cond { }
loop { if done { break; } }
```

- A range's bounds unify like operands, and `i` runs at that type; `for i in
  n` is `0..n`. Bounds are evaluated once; an empty or reversed range does
  nothing.
- `continue` goes to the next element; `break` leaves the innermost loop or
  `block`. **No labels** — use a `block`, a nested function with `return`, or
  a flag.

## Copy vs reference binding

- `for x in xs` **copies** fixed-size elements; the copy cannot be written
  (`x.f = 1` is an error naming `&x`). Variable-size elements bind by
  reference (they never copy implicitly). An element that *is* a reference
  binds as the reference it holds.
- `for &x in xs` binds every element by reference — the mutation form. Over
  a `T[>..<]` the binding is a live reference, so no shrink may happen
  inside the loop.

```goose
var xs: i64[>..] = [1, 2, 3];
for &x in xs { x *= 10; }                            // [10, 20, 30]
for x, i in xs { if i == 0 { xs.push(x + 1); } }     // the push extends the walk
```

## In-place traversal semantics

A loop iterates its sequence **in place**. A resizable array's length is
re-read before each iteration, so pushes inside a by-value `for` add to the
traversal; a slice keeps its own length. Whatever the loop walks in place is
live for the whole body, so the array it lies in may not shrink there.

## Iterating other things

- `for c in "goose"` iterates the bytes (integers).
- `for k in 0..a.len` for index loops; with index types
  (`type NodeId = u32 in nodes;`), `for v in nodes.len` gives `v` that
  array's index type, so `nodes[v]` needs no bounds check.
- Dictionaries: `d.each() { key, val => ... };` (order unspecified).
- `each_split`, `each_word`, `each_record`, `each_match` — library loops
  that build no arrays; a `return` inside their block returns from the
  enclosing function.

## `while` narrowing

A `while` whose condition is an optional test narrows for the body and
re-tests before each iteration — the linked-list walk:

```goose
while cur { sum += cur.v; cur .= cur.next; }
```

## Counted u8 loop gotcha

`for i: u8 in 0..256` runs every u8. A `u8` counter incremented 256 times
wraps to 0 — counts that must not wrap use `i64` (the default loop type).

## Related skills

- [guard-block-defer](guard-block-defer/skill.md)
- [function-values-blocks](../functions/function-values-blocks/skill.md)
- [index-types](../data-modeling/index-types/skill.md)
