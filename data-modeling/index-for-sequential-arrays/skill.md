---
name: goose-index-for-sequential-arrays
title: Index for sequential arrays — random access into packed data
category: data-modeling
tags: [goose, index array, sequential, variable-size elements, references index]
triggers: [goose index array, random access packed, iterate but not index, index of references]
description: >
  An array of variable-size elements can be walked but not indexed. Keep a
  second array of 4-byte links (in pool, global) or 8-byte plain references
  (local) beside it, pushed alongside each element, to reach any element in
  any order.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#an-index-for-a-sequential-array
---

# An index for sequential arrays

An array of variable-size elements (`u8[][>..]`, a `Mark..[>..]`) can be
walked but not indexed: where an element starts depends on every element
before it. Since other arrays can hold links into a pool, keep the index
yourself: one 4-byte link per element, pushed alongside it.

## The pattern

```goose
enum Mark {
    Circle { r: f64 },
    Rect { w: f64, h: f64 },
    Label { text: u8[varint] },
    Dot,
}

var marks: Mark..[>..] = [];
var mark_at: (Mark..&<u32 in marks>)[>..] = [];

fn main() {
    mark_at.push(marks.push(Mark.Label { text: "hello, goose" }));
    mark_at.push(marks.push(Mark.Circle { r: 1.5 }));
    mark_at.push(marks.push(Mark.Dot));
    mark_at.push(marks.push(Mark.Rect { w: 2.0, h: 3.0 }));
    for i in [3, 0, 4, 1, 2] {
        match mark_at[i] {
            Circle c => print(i, ": circle ", c.r),
            Rect r => print(i, ": rect ", r.w, " by ", r.h),
            Label l => print(i, ": label \"", l.text, "\""),
            Dot => print(i, ": dot"),
        }
    }
}
```

`push` builds each mark **in place** at the end of `marks` and returns a
reference to it. When that reference is stored in `mark_at`, it becomes the
mark's offset from the base of `marks`. `mark_at[i]` is then a load and an
add — the marks are reachable in any order. The five marks, including their
strings, take 44 bytes in one block, no pointers, no allocations.

## Local arrays use plain references

The array an `in` offset is measured from must be a **global** (or a
mutable array field of a global `var` root, like `pool.items`). A local
array is indexed the same way with plain references at 8 bytes per entry:

```goose
var local_index: (Mark..&)[>..] = [];
local_index.push(local_marks.push(Mark.Dot));
```

## When to just use fixed elements

If elements have a real bound (`u8[..16]` names), make them fixed-size and
the array is indexable directly — no second array needed. The index pattern
earns its keep when elements vary widely (a tree of mostly-small nodes,
mixed records) and you still need random access or sorting.

Sorting: sort the **index** (`sort(mark_at) { a, b => ... }` by payload
fields), leaving the compact array untouched.

## Related skills

- [relative-references](../references/relative-references/skill.md)
- [inline-nested-data](inline-nested-data/skill.md)
- [fixed-limited-frozen-arrays](../arrays/fixed-limited-frozen-arrays/skill.md)
