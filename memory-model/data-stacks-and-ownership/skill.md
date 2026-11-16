---
name: goose-data-stacks-and-ownership
title: The memory model — data stacks, ownership, scope exit
category: memory-model
tags: [goose, memory, ownership, stacks, scope, data-stacks]
triggers: [goose memory model, data stacks, who owns this, scope exit, no heap, bump pointer]
description: >
  Goose's entire memory system: N compiler-assigned data stacks with bump
  pointers, every dynamic value owned by a variable, freed en masse at scope
  exit, at most one resizable value live per stack, growth by pointer bump,
  nothing ever moves. Explains the rules and how to structure code around
  them.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#4-where-values-live
---

# The memory model: data stacks, ownership, scope exit

A Goose program has the native call stack, some static data (string literals,
compile-time constants), and **N data stacks**, where N is a small number the
compiler computes for the whole program. A data stack is a multi-gigabyte
address-space reservation with a bump pointer. **There is no other memory
system**: no `malloc`, no free list, no GC, no reference counting, no
destructors.

## The two rules

1. Values are pushed in lifetime order and popped **en masse** when the owning
   scope exits. Freeing is a single store (set the top back).
2. At most one **resizable** value is live per stack, and it is the topmost
   thing on that stack for its entire lifetime — so growth is a pointer bump
   with no capacity check, and the value **never moves**.

Three growable arrays alive at once get three stacks; a stack is reused as
soon as a scope frees it. You never write any of this down: the language has
no syntax for stacks. The compiler proves it statically for the whole program.

## Scope exit is the free

```goose
for round in 3 {
    var scratch: u8[>..] = [];
    for i in 100000 { scratch.push((i % 256) as! u8); }
    print("round ", round, ": ", scratch.len, " bytes of scratch");
}   // 100 KB freed here: one store, no destructor, no walk
```

Refilling a fresh scratch buffer each iteration is idiomatic, not wasteful.

## Nothing moves — references survive growth

```goose
struct Item { id: i32, weight: f32 }

var items: Item[>..] = [];
let first .= items.push(Item { 1, 0.5 });          // reference to element 0
for i in 2..1000001 { items.push(Item { i as i32, 0.0 }); }
first.weight = 99.5;                               // still valid, a million pushes later
```

Neither `std::vector` nor `Vec` can promise this — both reallocate. This is
why Goose code holds typed references where Rust code holds `u32` indices
into an arena. `push` returns a reference to the element it just made, which
is the idiomatic way to link data you are building.

## Who owns what — the question to ask

Stop asking "where do I allocate this?" and ask "**which scope owns this, and
what array does it live in?**" Most of the time the answer is "the function
that builds it", and nothing more is needed. When a value must outlive the
function that made it, the caller owns it: the caller declares the array and
passes it down as `T[>..]&`; the callee pushes into it and takes the
reference that `push` returns. When lifetimes truly do not nest (cache
entries, graph nodes edited in place), use `slot_pool<T>` / `slice_pool<T>`
from the standard library.

## Globals own the bottom of a stack

The whole program runs inside an implicit outermost scope that owns the
globals. A global growable array (`var log: u8[>..];`) owns the bottom of a
data stack for the program's life — a natural whole-program arena. Globals
are initialized before `main`, imported files first, then in declaration
order.

## Rules that follow for struct fields

- An ordinary struct holds **at most one resizable array**, and only as its
  **last field** (a resizable value must be the top of its own stack).
- A `root struct` lifts this: each field has independent storage, several
  fields can grow separately, but the owner must be a local or global
  variable.
- Never an array element: a resizable value cannot live inside another array.

## Tuning knobs (rarely needed)

- `--stack-reserve size` — address space per data stack (default 2G, up to
  256T; committed as used, so a large reservation costs nothing until used).
- `--stacks` — report how many stacks the program and its workers use.
- `-DGS_STACK_STATS=1` — report data stack use on stderr at exit.
- `os::trim_memory()` — return unused pages above stack tops to the OS after
  a large workload (explicit; pages fault back in on growth).
- `GS_MAX_STACKS` (default 1024) — a program needing more fails to compile.

## Common mistakes

- Trying to return a reference into a function's own local (dies with the
  call) — return owned values, or take the output array as `T[>..]&`.
- Expecting an array element to be resizable — restructure, or use a
  `root struct` / pool.
- Assuming `free`/`clear` frees memory eagerly for the OS — it moves a
  pointer; call `os::trim_memory()` if RSS matters.

## Related skills

- [size-classes-and-placement](size-classes-and-placement/skill.md)
- [copies-and-aliasing](copies-and-aliasing/skill.md)
- [slot-slice-pools](../data-modeling/slot-slice-pools/skill.md)
