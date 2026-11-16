---
name: goose-linked-structures
title: Linked structures — trees, lists, ASTs in pools with typed links
category: data-modeling
tags: [goose, tree, linked list, bst, ast, pool, links, sentinel]
triggers: [goose tree, linked list, binary search tree, expression tree, node pool, arena]
description: >
  Where Rust uses Vec<Node> + u32 indices and C++ uses pointers, Goose puts
  nodes in one grow-only array and links them with typed (often narrow)
  references that survive growth. BST insert, doubly-linked list with a
  self sentinel, variable-mode enum ASTs at 23 bytes, and the
  recursion rules for walking them.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md#10-linking-things-without-pointers
---

# Linked structures: nodes in an array, links as references

Pushing never moves existing nodes, so a reference from `push` stays valid
forever and the compiler checks it. This replaces `Box<Node>` trees,
`unique_ptr` links, and Rust's `Vec<Node>` + `u32` index arena workarounds.

## Binary search tree

```goose
struct Node { key: i32, count: i32 = 1, left: Node&<u32>?, right: Node&<u32>? }

fn insert(pool: Node[>..]&, key: i32) {
    if pool.len == 0 { pool.push(Node { key: key }); return; }
    var cur .= pool[0];                     // .= binds the element; = would copy it
    loop {
        if key == cur.key { cur.count += 1; return; }
        let next = if key < cur.key { cur.left } else { cur.right };
        if next { cur .= next; continue; }  // `if` narrows the optional
        if key < cur.key { cur.left .= pool.push(Node { key: key }); }
        else             { cur.right .= pool.push(Node { key: key }); }
        return;                             // the push cannot invalidate cur
    }
}

recursive fn in_order(n: Node&, out: i32[>..]&) {
    let l = n.left;
    if l { in_order(l, out); }
    out.push(n.key);
    let r = n.right;
    if r { in_order(r, out); }
}
```

Features at work: transparent references (`cur.key`, no `*`/`->`), `.=`
bind/rebind and `.==` identity, `T?` nullable links narrowed by `if`/`guard`,
`Node&<u32>` four-byte self-relative links, and `recursive fn` with the
accumulator passed in.

## Doubly-linked list with a sentinel (`self`)

```goose
import slot_pool;
struct Node { val: i64, prev: Node&<u32 in pool.items>, next: Node&<u32 in pool.items> }

var pool = slot_pool<Node> {};
let head .= pool.alloc_with() { Node { val: 0, prev: self, next: self } };  // sentinel

fn insert_after(p: Node&, v: i64) -> Node& {
    let n .= pool.alloc_ref(Node { val: v, prev: p, next: p.next });
    n.next.prev .= n;
    p.next .= n;
    return n;
}

fn remove(n: Node&) {
    n.prev.next .= n.next;
    n.next.prev .= n.prev;
    pool.free(pool.items.index_of(n));
}

fn is_end(n: Node&) -> bool { n .== head }
```

Because the sentinel exists, no link is ever null; every insert/remove is
the same four stores; every link load is a plain add. A node is 8+4+4=16
bytes. `alloc_with` selects the slot before running its builder, so `self`
names the node in its final location.

## Expression trees / ASTs

Combine a variable-mode enum with narrow links — each node is exactly as
big as its variant:

```goose
enum Expr { Num { v: f64 }, Add { l: Expr..&<u16>, r: Expr..&<u16> } }

fn eval(e: Expr.Num&) -> f64 { e.v }
recursive fn eval(e: Expr.Add&) -> f64 { eval(e.l) + eval(e.r) }

var pool: Expr..[>..] = [];
let a = pool.push(Expr.Num { v: 1.5 });
let b = pool.push(Expr.Num { v: 2.0 });
let sum = pool.push(Expr.Add { l: a, r: b });    // the whole tree: 23 bytes
```

## Graphs

Use **index types** for adjacency: `type NodeId = u32 in nodes;` — bounds
validated at construction, check-free traversal afterwards. CSR-style
`lo..hi` offsets into an edges array are the fastest layout (see index-types).

## Choosing the pool

- Locals/parameters pool, saving to bytes later → self-relative links in a
  plain `T[>..]`.
- Global pool, relink-heavy, links in from other arrays, or
  free/reuse needed → `slot_pool<T>` with `in pool.items` links.
- Per-node growable runs (n-ary tree child lists) → `slice_pool<T>`.

## Related skills

- [relative-references](../references/relative-references/skill.md)
- [slot-slice-pools](slot-slice-pools/skill.md)
- [index-types](index-types/skill.md)
- [recursion](../functions/recursion/skill.md)
