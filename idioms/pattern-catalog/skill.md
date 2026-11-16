---
name: goose-pattern-catalog
title: Pattern catalog — the idioms Goose code is written in
category: idioms
tags: [goose, patterns, idioms, context struct, sentinel, builder, command table, scratch buffer]
triggers: [goose idioms, goose patterns, context struct pattern, scratch buffer pattern, command table]
description: >
  The recurring shapes: builder + push-reference linking, per-iteration
  scratch buffers, the context struct of references for recursion, nested
  function parsers, sentinel-based linked lists, enum-tag command tables,
  case functions as virtual calls, format/hash overloads as the trait
  replacements, index types for graphs, and pools for non-nesting
  lifetimes.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md
---

# Pattern catalog

The recurring shapes of Goose code, each linking to the skill with the
details.

## Builder + push-reference linking

`push` returns a reference to the element it just made — build graphs as
you go:

```goose
let first .= items.push(Item { 1, 0.5 });
parent.child .= pool.push(Node { key: k });
```

## Per-iteration scratch buffer

Declare the buffer inside the loop; scope exit frees it with one store:

```goose
for round in 3 {
    var scratch: u8[>..] = [];
    for i in 100000 { scratch.push(...); }
}
```

A reusable scratch buffer and a structure you can point into have the same
type — hand out slices freely where the compiler can see they die first.

## The context struct (`Ctx`) for recursion

State that must outlive recursive activations lives in the **non-recursive
driver**; the recursion takes a struct of references (fixed-size, cheap):

```goose
struct Ctx { source: const u8[:], nodes: Node[>..]&, text: u8[>..]&,
             scratch: i64[>..<]& }
```

Small code nests functions instead and uses the tables as free variables.
A grow-shrink `scratch` shared by the whole recursion: take `mark =
scratch.len`, push, end with `scratch.resize(mark)` (a *balanced call*).

## Recursive-descent parsers as nested functions

Input, cursor and pools are locals of `parse`; grammar rules are nested
functions reading them as free variables. Errors: one `guard` +
`return … from parse` per rule.

## Sentinel linked list with `self`

`slot_pool.alloc_with() { Node { val: 0, prev: self, next: self } }` — no
null links, every insert/remove is four stores (see linked-structures).

## Enum-tag command tables

A stored callback is an enum variant plus its payload; dispatch with case
functions; the list can be saved to disk or queued to a worker:

```goose
enum Action { Spawn { kind: i32 }, Heal { amount: i32 }, Log }
fn run(a: Action.Spawn, w: World&) { ... }   // one overload per variant
```

## Case functions = virtual calls

`fn area(s: Shape.Circle) -> f64` — dispatch on the tag through a jump
table, exhaustiveness checked at compile time.

## `format` / `hash` overloads = the trait replacements

`fn format(out: u8[>..]&, p: Person)` customizes printing everywhere;
`fn hash(k: key) -> u64` (with `hash_combine`) makes a type a dictionary
key; `==` is structural and built in; ordering is a comparator block:
`sort(people) { a, b => a.age < b.age }`.

## Index types for graphs

`type NodeId = u32 in nodes;` + CSR offsets (`lo..hi` into an edges array)
+ `checkfree fn` traversal: bounds validated at construction, zero checks
in the loop.

## Pools for non-nesting lifetimes

`slot_pool<T>` for slots (cache entries, entities, open documents —
handles as indices), `slice_pool<T>` for per-object buffers (child lists,
texts). `free` recycles, never frees.

## defer for non-memory resources

`guard gfx::open(...) else { return; } defer gfx::close();` — the cleanup
next to what it cleans up, on every way out. Memory needs none of this.

## `const for` as reflection

Dump every field, map every field, exhaust every variant — expansion at
specialization time, no macros:

```goose
const for <N, A> in fields<T> { print(N, " = ", v.A); }
```

## Zero-copy text processing

Read a file once into one buffer; every key, table row and report line is a
slice into it (`split`, `words`, `dictionary<const u8[:], V>`); word
frequency over a book with nothing copied (sample `15_word_freq`).

## Serialization as a side effect of layout

Self-relative links → `to_bytes`/`from_bytes` with verification; design
for saving by choosing self-relative over `in pool` links.

## Related skills

- [linked-structures](../../data-modeling/linked-structures/skill.md)
- [recursion](../../functions/recursion/skill.md)
- [slot-slice-pools](../../data-modeling/slot-slice-pools/skill.md)
