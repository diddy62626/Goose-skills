---
name: goose-root-structs
title: Root structs — several independently growing fields under one owner
category: data-modeling
tags: [goose, root struct, workspace, multiple arrays, owner, compiler state]
triggers: [goose root struct, struct multiple vectors, independently growing fields, root struct placement]
description: >
  An ordinary struct holds at most one resizable tail. A root struct lifts
  that: each field has independent storage, several fields grow separately,
  all share the owner's lifetime. Placement rules (local/global only),
  reference semantics, generic roots, and what roots cannot do (copy,
  serialize as a whole, live inside another struct).
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#root-structs
---

# Root structs

Like an ordinary struct, a `root struct` bundles related state behind named
fields — but each field has **independent storage**, so several fields can
grow separately. Fields can be fixed, variable or resizable, in any order,
and all share the lifetime of the owning variable.

```goose
root struct Workspace<T> {
    items: T[>..] = [],
    scratch: i64[>..<] = [],
    calls: i64 = 0
}

fn add<T>(s: Workspace<T>&, value: T) -> T& {
    s.calls += 1;
    let result .= s.items.push(value);
    s.scratch.push(s.calls);
    s.scratch.clear();                // does not invalidate result
    return result;
}

fn main() {
    var state = Workspace<i64> { .. };
    let first .= state.add(7);
    state.add(8);
    print(first, " ", state.items, " ", state.calls);
}
```

## The trade-off: placement

An owning root struct must be a **local or global variable** — it cannot be
a field, an array element, a copy, or a return value, and it cannot be a
temporary. This applies even to a root with no resizable fields. Functions
receive it by reference (`Workspace<T>&`).

## Reference semantics

- An unannotated binding from an existing owner also borrows it; `.=` is
  available to make that intent explicit.
- Owner references can be optional (`Workspace<T>?`), stored in ordinary
  structs or arrays, returned, and captured by nested functions — subject to
  normal lifetime and writability rules.
- Field access, UFCS, reflection (`fields<T>`) and formatting work through
  those references.

## Compact links and index types can name root fields

A mutable grow-only array field of a **global `var` root** can be named in
compact reference and index types: `Node&<u32 in pool.items>`,
`u32 in pool.items`, `u32 upto pool.items`. The name resolves to that
particular field's storage; another field or owner is a different array.
Only direct fields are supported (no aliases, references, nested paths).

## What a root cannot do

- No packed byte image (`bytes_of`), whole-value equality, arithmetic, or
  public C layout. Serialize eligible fields separately, e.g.
  `to_bytes(state.items)`.
- Cannot be copied, reassigned, passed or returned by value.
- Nested owning roots and fixed arrays of owners are deferred (v1).
- Workers can construct local roots but cannot receive owners or owner
  references, or use a root global directly or through another function.

## The stdlib pools are roots

`slot_pool<T>` and `slice_pool<T>` are root structs bundling an `items`
array with reuse metadata — the model for "lifetimes that do not nest" and
the replacement for the deprecated `reusable` builtins.

## When to use which

| Need | Use |
|---|---|
| One growable member | ordinary struct with a resizable last field |
| Several growable members, one owner variable | `root struct` |
| Code spread over files needing shared tables | locals of a driver + a struct of references (`Ctx`) |
| Elements created/destroyed out of scope order | `slot_pool<T>` |

## Related skills

- [size-classes-and-placement](../memory-model/size-classes-and-placement/skill.md)
- [slot-slice-pools](slot-slice-pools/skill.md)
- [recursion](../functions/recursion/skill.md)
