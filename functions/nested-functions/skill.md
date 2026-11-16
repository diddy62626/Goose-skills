---
name: goose-nested-functions
title: Nested functions — shared-state parsers without context structs
category: functions
tags: [goose, nested functions, closures, parser state, free variables]
triggers: [goose nested function, inner function, shared parser state, free variables]
description: >
  A fn declared inside a fn reads and writes the enclosing function's locals
  as free variables. Nested functions of one block may call each other in
  either order (recursive-descent parsers), overload like top-level ones,
  and are compile-time entities. When to nest vs pass a context struct.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#nested-functions
---

# Nested functions

A function may be declared inside a function, and it reads and writes the
enclosing function's locals as free variables, under those variables' normal
rules. Names resolve where the nested function is declared. The nested
functions of one block may call each other in **either order** — the
functions of a recursive-descent parser — but the enclosing code can call a
nested function only after its declaration. Nested functions of one name in
one block overload like top-level ones, so case functions can be nested
too. A nested function is a compile-time entity: passed as a function value,
never escapes.

## Shared state without a context struct

```goose
struct Employee { name: u8[..12], dept: u8[..8], salary: i32 }

fn main() {
    var staff: Employee[>..] = [];
    // ... push employees ...
    var budget = 0;
    var seen = 0;
    fn spend(e: Employee&) {          // writes main's locals
        budget += e.salary;
        seen++;
    }
    for &e in staff { if e.dept == "eng" { spend(e); } }
}
```

The sample parsers use this to share their input, cursor and pools: all are
locals of `parse`, accessible to the recursive-descent functions nested
inside it. This avoids passing a state struct through every call.

## Nested functions and recursion

Recursion must be marked `recursive fn` and signatures written out — nested
recursive functions follow the same rules (see recursion). A nested function
can reach growable locals of the enclosing function, but a function inside a
recursive *cycle* may not keep growable data across a call back into the
cycle: declare the tables in the **non-recursive driver** that owns the
nested functions.

## The context-struct alternative

When the code working on several growable tables is spread over many
functions or files, make each table a local of a driver function and pass a
single struct of references to them:

```goose
struct Ctx {
    source: const u8[:],
    nodes: Node[>..]&,
    text: u8[>..]&,
    scratch: i64[>..<]&,
}
```

A struct of references and slices is fixed-size and cheap to pass. When the
code fits inside the driver, nested functions see the tables as free
variables with no struct at all.

## Interaction with blocks

A trailing block passed to a call has the same power: it reads/writes
enclosing locals and its `return` leaves the enclosing function (see
function-values-blocks). Prefer a named nested `fn` when you pass the same
behavior in more than one place or want a name for it.

## Related skills

- [recursion](recursion/skill.md)
- [function-values-blocks](function-values-blocks/skill.md)
- [pattern-catalog](../idioms/pattern-catalog/skill.md)
