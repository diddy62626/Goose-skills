---
name: goose-grow-only-resizable
title: Grow-only resizable arrays — T[>..], the default container
category: arrays
tags: [goose, arena, builder, push, append, growable, references survive]
triggers: [goose growable array, T[>..], arena, push returns reference, clear, resize down]
description: >
  T[>..] is Goose's workhorse container: push/append bump the stack, elements
  never move, references into it stay valid forever, and shrinks are allowed
  exactly where the compiler proves nothing points into it. Patterns for
  arenas, builders, scratch buffers, and the shrink rules.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#grow-only-arrays-shrink-where-nothing-points-in
---

# Grow-only arrays (`T[>..]`)

The default growable array. Elements sit on top of a data stack; `push` and
`append` bump the pointer; nothing ever moves.

```goose
var grow: i64[>..] = [];
for i in 5 { grow.push(i * i); }          // [0, 1, 4, 9, 16]
grow.append([2, 3, 4]);                   // append an array, slice or literal
let last .= grow.push(100);               // push RETURNS a reference — the linking idiom
grow[grow.len - 1];                       // index as usual
```

## Why references into it are the point

```goose
struct Item { id: i32, weight: f32 }
var items: Item[>..] = [];
let first .= items.push(Item { 1, 0.5 });
for i in 2..1000001 { items.push(Item { i as i32, 0.0 }); }
first.weight = 99.5;                      // still valid, a million pushes later
```

This replaces Rust's arena + `u32` index workarounds: hold the typed
reference directly. `let e .= pool.push(Node { ... })` then
`parent.child .= e` links structures in one store each.

## Shrinking: allowed only where nothing points in

A `T[>..]` never shrinks while a reference or slice into it can be live.
`pop`, `pop_n`, `resize` downward, `clear`, and whole-array assignment are
legal exactly where the compiler sees that:

- no reference/slice *variable* rooted at the array is used after the shrink
  (in its own block, an enclosing block, or a later loop iteration),
- no *holder* (struct with a reference field, array of slices) still in use
  had a reference into the array stored into it,
- the shrink's own statement does not use a reference into the array that it
  evaluated earlier (`use(a[0], a.pop())` is an error; `a[0] + a.pop()` is
  fine — it reads its left operand first).

The test is **liveness, not scope**: a slice whose last use is before the
shrink is dead, and the shrink is fine.

```goose
var line: u8[>..] = [];
for i in 3 {
    reset(line);                       // nothing points into line here
    format(line, "line ", i + 1);
    let head = line[..4];
    print(head, "|", line[4..]);       // head's last use
}
```

A scratch buffer refilled per iteration can hand out slices of itself freely.

## Shrink checks through calls

A shrink through a reference parameter, or of a global, is checked at every
level: each function records which globals and which parameters' targets it
shrinks, and a call is checked as if it made those shrinks in the caller.
`pop`/`pop_n`/`resize` need fixed-size elements (a sequential array cannot
find its last element).

## Idioms

- **Builder**: `var out: u8[>..] = [];` then `format(out, ...)` /
  `out.append(...)`.
- **Refill between phases**: declare inside the loop; scope exit frees with
  one store.
- **Index of a reference**: `xs.index_of(r)` — needs a reference the checker
  can prove points into that very array.
- **Validate big counts first**: `xs.check_length(n)` rejects counts whose
  elements alone would exceed a data-stack reservation.
- **Compact when done**: `let done: i64[] = copy(grow);` — finished data
  frees the one-per-stack rule.

## Whole-array assignment

`xs = [4, 5];` means *clear, then construct*: the right side is built where
the old elements were, so it may not use the array at all (`x = f(x)` is an
error — build in a variable and assign `copy()` of it).

## Related skills

- [array-family-overview](../array-family-overview/skill.md)
- [grow-shrink-stacks-queues](../grow-shrink-stacks-queues/skill.md)
- [linked-structures](../../data-modeling/linked-structures/skill.md)
