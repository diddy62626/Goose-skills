---
name: goose-index-types
title: Index types — `u32 in nodes` and checkfree functions
category: data-modeling
tags: [goose, index types, bounds check, checkfree, NodeId, graph, CSR]
triggers: [goose index type, u32 in array, NodeId, bounds check elided, checkfree, upto]
description: >
  An index type uN in a is an unsigned integer proven below a.len wherever
  observed; the check moves to where the index enters the program (x as
  NodeId aborts if out of range). Indexing a with its own index type needs
  no bounds check — the basis of checkfree graph traversal. Bound types
  (uN upto a) for range ends.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#15-index-types-and-checkfree-functions
---

# Index types and `checkfree`

Every index is bounds-checked and the compiler removes the checks it can
prove unnecessary. The check it cannot remove is an index read out of
*another* array — `dist[adj[k]]` — since whether it is in range depends on
one array's contents vs another's length. **Index types move that check to
where the index enters the program.**

## The types

- `uN in a` — an unsigned integer (`u8`/`u16`/`u32`/`u64`) that is **below
  `a.len`** wherever it can be observed.
- `uN upto a` — an unsigned integer at most `a.len`, for range ends.

`a` names a global `var` array (or a mutable array field of a global `var`
root, e.g. `u32 in graph.nodes`), grow-only with fixed-size elements. The
array is part of the type's identity: `u32 in nodes` is neither `u32` nor
`u32 in edges`. The value is the plain integer.

```goose
struct Node { lo: u32 upto edges, hi: u32 upto edges, weight: i32 }
var nodes: Node[>..] = [];
type NodeId = u32 in nodes;          // an alias reads better at every use
var edges: NodeId[>..] = [];
```

## Making and using one

- **Making:** `x as (uN in a)` aborts in **every build** unless
  `0 <= x < a.len`, with the error `a[x]` would give. It is a bounds check
  moved to this point, elided by the same analysis that would elide
  `a[x]`. `as!` cannot make one; a literal never adapts to one.
- **Using:** indexing `a` with an index of `a` needs **no check**. Values
  convert implicitly to their plain integer type; compound assignment on a
  place of an index type is an error, as is memberwise arithmetic on a
  struct holding one.
- **Loops:** `for i in lo..hi` with index/bound bounds of one array gives
  `i` that array's index type; so does `for i in n` with `n` of one.
- **Passing through:** declarations, parameters, results, fields, payloads
  and elements carry the type; a branch whose arms all have one array's
  index types keeps it (widest width). A reference to such storage is
  `(u32 in a)&`, never `u32&`.
- **Never shrunk:** an array an index type names cannot be shrunk anywhere,
  so a value once in range stays in range. No default value. `from_bytes`
  checks every index in an image against its array.

An index type takes no postfix: arrays of them are `(u32 in nodes)[>..]` or
`NodeId[>..]`; `for` binders parenthesize: `for v: (u32 in nodes) in …`.

## Worked example: check-free graph traversal

```goose
fn add_node(w: i32) -> NodeId {
    let none = 0 as (u32 upto edges);
    nodes.push(Node { lo: none, hi: none, weight: w });
    return (nodes.len - 1) as NodeId;
}

for v in nodes.len {
    let n = nodes[v];
    for k in n.lo..n.hi { total += nodes[edges[k]].weight; }   // no checks left
}
```

With a `checkfree fn` on the traversal, a surviving bounds check is a
**compile error** — the compiler verifies the whole kernel is check-free
(this is the October 2026 benchmark improvement: check-free BFS/CSR rows
got measurably faster).

## Related skills

- [function-qualifiers](../functions/function-qualifiers/skill.md)
- [bounds-check-elimination](../performance/bounds-check-elimination/skill.md)
- [linked-structures](linked-structures/skill.md)
