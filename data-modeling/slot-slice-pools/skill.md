---
name: goose-slot-slice-pools
title: slot_pool and slice_pool — lifetimes that do not nest
category: data-modeling
tags: [goose, slot_pool, slice_pool, freelist, reuse, shared_ptr, object pool, slots]
triggers: [goose slot_pool, goose slice_pool, object pool, free list, non-nested lifetimes, alloc_ref, free]
description: >
  When data does not die in scope order (cache entries, graph nodes, open
  documents), the stdlib's slot_pool<T> hands out reusable slots and
  slice_pool<T> reusable contiguous runs. free releases no memory and ends
  no lifetime — every element stays a live, well-typed value; stale
  references read a replacement value, a logic bug like a stale index, never
  corruption.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#slot_pool
---

# `slot_pool` and `slice_pool`

Some data has lifetimes that do not nest: cache entries, mutable graph
nodes, files deleted in arbitrary order. The standard library's pools handle
this. Each is a `root struct` bundling a grow-only `items` array with reuse
metadata. Declare the owner as a local or global; pass it as
`slot_pool<T>&` / `slice_pool<T>&`.

## slot_pool — individual slots

```goose
import slot_pool;

var pool = slot_pool<Item> {};          // struct Item { id: i32, weight: f32 }

let s0 = pool.alloc_index(Item { 10, 1.0 });   // slot index
let r  .= pool.alloc_ref(Item { 11, 1.0 });    // or a reference
pool.free(s0);                                  // slot 0 goes on the freelist
let s2 = pool.alloc_index(Item { 12, 2.0 });   // takes slot s0 back
pool.free_ref(r);                               // reference back to its slot
pool.recycle_all(pool);                         // every slot reusable again
```

Also: `alloc_with() { Node { value: 1, me: self } }` constructs a fresh
value **directly in the chosen slot** (supports self-relative links and
`self`); `alloc_index_with` returns the chosen index.

## The free semantics — read this twice

> **`free` releases no memory and ends no lifetime.** Every element remains
> a valid, live value of its type at all times. A reference to a freed and
> reused slot reads a *different `Item`* — a valid one, but not the one you
> were thinking of. That is a logic bug of the same class as keeping an
> index into an array you have since overwritten: never memory corruption,
> type confusion, or an out-of-bounds access.

Practical rules:

- After returning a slot, stop using references to its old contents. To
  detect staleness, keep a generation counter in the element and compare it
  (as with a Rust slotmap key).
- The pool never shrinks; `items.len` is the high-water count including free
  slots, and iterating `items` visits those too.
- Elements must be **fixed-size**.
- Do not edit `free_slots` metadata or shrink `items` directly.

Read/iterate elements through `pool.items`: `pool.items[s2]`,
`for item in pool.items`, `pool.items.index_of(ref)`.

## slice_pool — contiguous runs

Hands out *runs* of elements with a free-span tree that merges neighbours:

```goose
import slice_pool;
var pool = slice_pool<u8> {};

let a = pool.alloc_slice(128);          // a T[:], every element defaulted
let t = pool.alloc_copy(text);          // allocate + copy in one step
let bigger = pool.realloc_slice(a, 256); // in place when it can, else moved
pool.free_slice(t);
pool.trim();                            // u8 pools only: return free pages to the OS
```

- `alloc_slice(n)` takes the first free span that fits, else grows the
  array (tail free spans first); every handed-out element is reset to its
  default.
- `realloc_slice` preserves the first `min(s.len, n)` elements; growth may
  **move** the run — keep cursors as offsets into `items`, not slices, when
  a run can be reallocated.
- `free_slice`/`realloc_slice` accept subranges of allocated runs; empty or
  default slices can be freed harmlessly. A nonempty slice must be aligned
  and wholly within `items` (runtime-checked where the compiler cannot
  prove it).
- `trim` discards the complete pages inside every free span (they read back
  as zeros — the value `alloc_slice` gives anyway). Explicit, like
  `os::trim_memory`: call after closing large buffers, not after every free.

## The open-documents pattern (shared_ptr replacement)

```goose
var docs = slot_pool<Document> {};
var bytes = slice_pool<u8> {};

fn open_doc(id: u64, size: i64) -> i64 {
    return docs.alloc_index(Document { id: id, text: bytes.alloc_slice(size) });
}
fn close(slot: i64) {
    bytes.free_slice(docs.items[slot].text);
    docs.free(slot);
}
```

Hand out slot indices (not references) as your handles — simple APIs, and
`free` takes an index. Where ref-counted shared ownership would be used,
decide who owns the object and free it there; other holders accept the
stale-slot semantics or check a generation.

## Migration from `reusable`

`reusable var p: T[>..] = [];` → `var p = slot_pool<T> {};`;
`reusable[] var` → `slice_pool<T>`; `p[i]` → `p.items[i]`;
`u32 in p` → `u32 in p.items`. The deprecated builtins still work; removal
is a later step. Owner transfer to workers remains a migration gap — use
the deprecated builtins for worker snapshots, or the explicit-copy recipe in
sample `32_pool_worker_snapshots`.

## Related skills

- [root-structs](../root-structs/skill.md)
- [linked-structures](../linked-structures/skill.md)
- [concurrency-thread-model](../../concurrency/thread-model/skill.md)
