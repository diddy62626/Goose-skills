---
name: goose-function-values-blocks
title: Function values and trailing blocks — HOFs without closures
category: functions
tags: [goose, higher-order functions, trailing block, it, lambda, callbacks, filter map]
triggers: [goose lambda, goose block, trailing block, it parameter, higher order function, callbacks]
description: >
  Function values are compile-time entities passed as generic parameters —
  never runtime data, never stored or returned. A trailing block { ... } is
  the last argument (implicit it, or a, b =>). Blocks read/write enclosing
  locals, and a return inside a block returns from the LEXICALLY ENCLOSING
  function — a hand-written iterator has for's control power.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#function-values
---

# Function values and trailing blocks

Function values are **compile-time entities** passed as generic parameters,
not runtime data. A function argument is a function name, a bound generic
function parameter, or a block literal; the `: fn(…)` bound on the parameter
is optional documentation. Every call of a function value is direct and
inlinable, and a function value **cannot escape**: storing one, returning
one, or putting one in data is a compile error. There are no closure
objects and no function pointers — a higher-order function compiles to
exactly the loop it looks like.

## The trailing-block sugar

```goose
fn foo<F: fn(i64)>(a: i64) { F(a); }
foo(1) { print(it) }                 // implicit parameter `it`
foo(1) { x => print(x) }             // named block parameters
foo(1, h)                            // a named function
```

- `{ … }` after a call's parentheses is the last argument. Goose allows one
  trailing block per call; a second function value is passed by name.
- A block reads and writes the enclosing function's locals, like a nested
  function.
- A block's parameters have no defaults (a named function keeps its).
- A block's value type is inferred without the destination its call meets,
  so a block's value cannot be `[]`.

## The standard library is written this way

```goose
let evens   = xs.filter() { it % 2 == 0 };
let squares = xs.map()    { it * it };
let total   = fold(xs, 0) { acc, x => acc + x };
sort(xs)    { a, b => a.age < b.age };
let big     = xs.find() { it > 10 };
```

`filter` and `map` build their result **directly in the variable receiving
it** — no intermediate array, no iterator chain. A HOF never states what its
function value returns: `map`'s result element type is whatever `F(x)` is in
that instantiation.

## Writing your own HOF

```goose
fn each_pair<T, F>(xs: T[:]) {
    for i in 0..xs.len - 1 { F(xs[i], xs[i + 1]); }
}

var gaps: i64[>..] = [];
each_pair(xs) { a, b => gaps.push(b - a); };     // writes an enclosing local
```

## `return` inside a block returns from the enclosing function

A `return` in a block returns from the **lexically enclosing named
function**, not from the function calling the block. A hand-written iterator
thereby has the same structuring power as `for`:

```goose
fn each_step<F>(lo: i64, hi: i64, step: i64 = 1) {
    var i = lo;
    while i < hi { F(i); i += step; }
}

fn first_multiple_of_7(lo: i64, hi: i64) -> i64 {
    each_step(lo, hi, 3) { if it % 7 == 0 { return it; } };  // returns from first_multiple_of_7
    return -1;
}
```

A reference returned this way may be one the calling function handed to the
block; it may not point into that function's own locals or the block's.

## Stored callbacks become enum tags

When C++ keeps `vector<std::function<...>>` of handlers, store *which*
behavior as data and dispatch with case functions — the payload carries what
a closure would have captured, and the list can then be saved or sent to a
worker:

```goose
enum Action { Spawn { kind: i32 }, Heal { amount: i32 }, Log }
fn run(a: Action.Spawn, w: World&) { ... }
fn run(a: Action.Heal,  w: World&) { ... }
fn run(a: Action.Log,   w: World&) { ... }
var on_tick: Action[>..] = [];
on_tick.push(Action.Heal { amount: 5 });
for a in on_tick { run(a, world); }
```

A name with several overloads passes them all; each call of the value
resolves among them. A function value passed as an argument may come after
fewer arguments than the callee has parameters, where the rest have defaults.

## Related skills

- [generics](../generics/skill.md)
- [nested-functions](../nested-functions/skill.md)
- [enums-adts](../../types/enums-adts/skill.md)
- [std-algorithms](../../stdlib/std-algorithms/skill.md)
