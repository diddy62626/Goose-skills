---
name: goose-guard-block-defer
title: guard, block-with-break, and defer
category: control-flow
tags: [goose, guard, defer, block, early exit, cleanup]
triggers: [goose guard, goose defer, early return, cleanup on exit, block break value]
description: >
  guard states a condition the rest of its block depends on (and narrows
  optionals). block { } gives break a value target without a loop. defer
  runs on every exit of its block — end, break, continue, return, return
  from — latest first, once per iteration in loops, with its three
  restrictions.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#guard
---

# `guard`, `block`, `defer`

## `guard`

`guard c;` runs the rest of its block only where `c` holds; `guard c else {
… }` runs the else block instead. It is `if` turned inside out, and it does
not jump anywhere by itself: in a function body the rest of the block is the
rest of the function; in a loop body it goes on to the next iteration. To
leave more than the block, put `return`/`break`/`continue` in the else
block. After a guard, the condition is known to hold — which includes
narrowing an optional: after `guard r else { return; }`, `r` is a `T&`.

```goose
fn half(n: i64) -> i64 { guard n % 2 == 0 else { return -1; } n / 2 }
fn tenfold(p: i64?) -> i64 { guard p else { return 0; } p * 10 }

for i in 5 { guard i % 2 == 0; print("even ", i); }
```

## `block` and `break` with a value

```goose
let verdict = block {
    guard p.alive else { break "gone"; }
    guard p.hp > 0 else { break "dead"; }
    "standing"
};
```

`block { … }` gives `break` a target without a loop; the construct's value
is the `break E`'s `E` or the trailing expression. A bare `{ … }` is only a
scope. Replaces labelled-break patterns; there are no labels in v1.

## `defer`

`defer s;` / `defer { … }` runs when the block it is written in ends, **whichever
way it ends**: falling off the end, `break`, `continue`, `return`, a block's
`return` through a higher-order function, or a `return … from` passing
through. A `defer` in a loop body runs at the end of **every** iteration. An
exit runs the defers written before it, **latest first**. The body sees
variables as they are at the exit; on a `return` it runs after the returned
value is built and before the block's storage is released.

```goose
guard gfx::open("demo", 640, 480, 0) else { return; }
defer gfx::close();                 // every way out of this block from here on

var depth = 0;
fn describe(items: i64[:]) -> i64 {
    depth++;
    defer depth--;                  // every way out restores it
    for x in items {
        defer count++;              // once per iteration, continue included
        if x < 0 { continue; }
        if x > 100 { return -1; }
    }
    return count;
}
```

The body is checked where it is written. Three things it may **not** do
(each a compile error):

1. **Leave** — no `return`/`break`/`continue` targeting anything outside it
   (including inside a block passed to a call in the body).
2. **Use long-distance returns** — may not call a function that does
   `return … from`, propagates or catches one (the body may run while one is
   in flight); the error names the call chain.
3. **Name the function's named result.**

A `defer` belongs in a function: a global initializer has no scope exit.
Nothing runs a defer where no exit happens: `abort`, a failed `assert` or
bounds check, `exit`, and a contained worker's abort end the code without
running any.

## `guard` + `defer` together: the resource idiom

```goose
guard gfx::open(...) else { return; }
defer gfx::close();
```

The cleanup stays next to what it cleans up and runs on the ways out that
the code after it does not mention. This is Goose's RAII for non-memory
resources (memory needs none — scope exit frees it).

## Related skills

- [expressions-if-match](expressions-if-match/skill.md)
- [loops](loops/skill.md)
- [return-from](../errors/return-from/skill.md)
