---
name: goose-grow-shrink-stacks-queues
title: Grow-shrink arrays — T[>..<] for stacks, queues and heaps
category: arrays
tags: [goose, stack, queue, heap, pop, grow-shrink, priority queue]
triggers: [goose stack, goose queue, goose heap, T[>..<], heap_push, heap_pop, pop]
description: >
  T[>..<] shrinks from anywhere but keeps its interior references out of
  storage (variables only). Pop/pop_n/resize/clear, the std heap functions,
  balanced calls that let a recursion share one scratch stack, and the
  reference rules that make shrink checks cheap.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#grow-shrink-arrays-keep-their-references-in-variables
---

# Grow-shrink arrays (`T[>..<]`)

`T[>..<]` holds **fixed-size** elements and shrinks from anywhere: on a
local, through a reference, on a global, as a struct's tail. The trade-off:
**references and slices into it live in variables only.** They may be bound
to a local, passed down and returned, but never stored into a field, an
element, a global or a literal. That keeps the shrink check cheap (scan the
variables in scope): a shrink is an error while any such variable is still
used afterwards.

```goose
var stack: i64[>..<] = [];
stack.push(1); stack.push(2); stack.push(3);
let top = stack.pop();
stack.pop_n(2);                        // drop the last two at once
stack.resize(6, 0);                    // grow with copies of 0 / shrink
stack.clear();
```

## The natural heap and queue

`std`'s heap functions turn any grow-shrink array into a binary min-heap
(children of `i` at `2i+1`, `2i+2`):

```goose
import std;
var q: i64[>..<] = [];
q.heap_push(5); q.heap_push(1);
let smallest = q.heap_pop();           // 1
q.make_heap();                         // heapify O(n)
```

All come with block forms (`heap_push(xs) { a, b => a.age < b.age }` — "a
goes before b"). For a priority queue or a work stack, this is the
container. A limited `[..k]` field works as well.

## References into it

```goose
var stack: i64[>..<] = [];
for i in 5 { stack.push(i * 10); }
let top .= stack[4];
print(top);                            // last use of top
stack.pop();                           // fine
```

The binding is a live reference, so no shrink may happen inside a
`for &x in stack` loop body either.

## Balanced calls and the shared recursion scratch

A call is **balanced** for an array when the array is never shorter during
the call than when it began — e.g. take `let mark = xs.len`, push, end with
`xs.resize(mark)`. A balanced call frees nothing a view taken before it
points into, so it does not count as a shrink. This is how a recursion
shares one scratch stack:

```goose
recursive fn walk(scratch: i64[>..<]&, depth: i64) -> i64 {
    let mark = scratch.len;
    scratch.push(depth);
    let mine = scratch[mark..];        // my activation's entries
    var total = 0;
    if depth < 3 { total = walk(scratch, depth + 1); }
    total += mine[0];
    scratch.resize(mark);              // gone before it returns
    return total;
}
```

## Where it fits in the family

- Use `[>..<]` instead of `[>..]` when you pop, and when you want the
  stronger shrink freedom.
- Elements must be fixed-size; the array can sit as a struct's last field.
- `from_bytes` can build one, except whose elements hold relative references.

## Related skills

- [array-family-overview](array-family-overview/skill.md)
- [grow-only-resizable](grow-only-resizable/skill.md)
- [recursion](../functions/recursion/skill.md)
- [std-algorithms](../stdlib/std-algorithms/skill.md)
