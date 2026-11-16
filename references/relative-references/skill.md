---
name: goose-relative-references
title: Relative references — narrow links, pool-relative, and `self`
category: references
tags: [goose, relative reference, links, pool, self, sentinel, serialization]
triggers: [goose relative reference, T&<u32>, in pool, self initializer, narrow links, linked list]
description: >
  A relative reference stores a link as a narrow offset (u8/u16/u32/u64/
  varint) instead of an 8-byte address. Self-relative (same array,
  position-independent, serializable) vs pool-relative (T&<u32 in pool>,
  other arrays can link in). T&<u32>? makes offset 0 the null. self
  builds sentinels; copying rules for values holding self-relative links.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#11-relative-references
---

# Relative references

A `T&` is eight bytes, and an address means nothing outside the process. A
**relative reference** stores the same link as a narrow offset:
`T&<u8>`, `T&<u16>`, `T&<u32>`, `T&<u64>`, `T&<varint>`. Loading one yields
an ordinary `T&`; storing a `T&` re-encodes it; otherwise it is used like any
reference.

## The two forms

- **Self-relative** `T&<u32>`: offset from the *field itself* to the target,
  which must live in the **same enclosing array**. The structure is
  position-independent → it can be copied, saved and loaded as a whole
  (serialization).
- **Pool-relative** `T&<u32 in pool>`: offset from the base of a named
  **global** array (or a mutable array field of a global `var` root, e.g.
  `Node&<u32 in pool.items>`). A store is a subtraction from a base already
  in a register, and **other arrays can hold links into the pool**.

`T&<u32>?` is the optional form: offset 0 is null, so a link costs four
bytes *including* its null. The stored pool offset is base+1, so a `u32`
link spans a 4 GB pool.

## A binary search tree in one array

```goose
struct Node { key: i32, count: i32 = 1, left: Node&<u32>?, right: Node&<u32>? }

fn insert(pool: Node[>..]&, key: i32) {
    if pool.len == 0 { pool.push(Node { key: key }); return; }
    var cur .= pool[0];                  // .= binds the element; = would copy it
    loop {
        if key == cur.key { cur.count += 1; return; }
        let next = if key < cur.key { cur.left } else { cur.right };
        if next { cur .= next; continue; }              // narrowed by `if`
        if key < cur.key { cur.left .= pool.push(Node { key: key }); }
        else             { cur.right .= pool.push(Node { key: key }); }
        return;                          // the push cannot invalidate cur: nothing moves
    }
}
```

The node is 16 bytes (two `i32`s + two 4-byte links) vs 24 plus an
allocation header for a `Box`/`unique_ptr` tree. Storing a self-relative
link requires the compiler to see that reference and destination derive from
the **same root array, exactly**.

## Copying rules (important)

A value that *contains* self-relative links **cannot be copied** in any way:
by assignment from a variable, as a by-value argument or match binder, as an
element copy, or by `copy(x)`. Construct it in place (push a literal), bind
by reference (`let n .= f();` — the error will suggest this), match with
`&`-binders, and take it by reference in case functions and `format`
overloads. A value whose links are all `in pool` copies like any other. The
one copy always allowed is a default value (a literal's `..` sets a relative
field to null).

A function/block may return a *fresh* value with relative links; it is
constructed at the receiver, and that receiver must share the root array the
links point into.

## `self` and sentinels

A non-optional relative reference has no null, so a structure whose links
should point at itself could not be built — **`self`** (legal only as the
whole initializer of such a field, in a struct/variant literal) denotes the
value being constructed:

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
```

Every insert/remove is the same four stores; every link load is a plain add,
no null test. Note the `.=` on results: `let two = insert_after(...)` would
copy the node out of the pool, which the compiler rejects at `remove`.

## Which form when

- **Self-relative**: position-independent blobs (a compact tree saved,
  mapped or moved whole); pools that are locals/parameters (no name to write
  in a type).
- **`in pool`**: relink-heavy structures in a global pool; links *into* the
  pool from other arrays; where you want the offset as an index
  (`&pool[i]` encodes as `i * sizeof(T) + 1`; `index_of` reads it back).

A function's *result* cannot be declared relative: declare `T&` or `T?`, and
a relative slot that receives it encodes it.

## An index for sequential arrays

```goose
var marks: Mark..[>..] = [];
var mark_at: (Mark..&<u32 in marks>)[>..] = [];
mark_at.push(marks.push(Mark.Label { text: "hello, goose" }));
match mark_at[i] { Label l => print(l.text), ... }
```

The array an `in` offset is measured from must be global; a local array uses
plain `(Mark..&)[>..]` at 8 bytes per entry.

## Related skills

- [linked-structures](../data-modeling/linked-structures/skill.md)
- [serialization](../data-modeling/serialization/skill.md)
- [slot-slice-pools](../data-modeling/slot-slice-pools/skill.md)
- [references-and-ufcs](references-and-ufcs/skill.md)
