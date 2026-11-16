---
name: goose-syntax-tour
title: Goose syntax tour — the whole language in one pass
category: getting-started
tags: [goose, syntax, basics, tutorial, variables, functions, control-flow]
triggers: [goose syntax, goose basics, goose let var, goose fn, goose for loop, goose match]
description: >
  Compact tour of Goose syntax for people who know C/Rust/Go: let/var/const,
  types after the name, expression-oriented if/match/block, guard, defer,
  loops, multiple return values, default parameters, UFCS method-call syntax,
  the three arithmetic rules, and struct literals with `..`.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md
---

# Goose syntax tour

Much of the syntax is familiar: braces, `let`, `fn`, `struct`, `match`. Types
follow the name and are usually inferred. This file is the map; the linked
skills hold the depth.

## Variables and bindings

```goose
let answer = 42;              // cannot be reassigned (contents still writable)
var count: i32 = 0;           // reassignable
const limits = [10, 20, 30];  // read-only contents
count += 1;
count++;                      // a statement, not an expression
let ratio = 0.75;
```

`let` forbids assigning the *name*, not writes through it:
`let xs = [1, 2, 3]; xs[0] = 9;` is fine. Read-only contents come from the
type (`const T`), not from `let`. A `let` without an initializer must be
assigned exactly once, checked on every path.

## Functions are free functions; UFCS gives method syntax

```goose
fn total(xs: i64[:]) -> i64 {
    var t = 0;
    for x in xs { t += x; }
    return t;
}
```

There are no methods and no `impl` blocks, but `x.f(a)` calls `f(x, a)` for
any function — `xs.total()` works, `xs.sort()` works, `2.5.describe()` works.

## Struct literals, defaults, `..`

```goose
struct Unit { kind: i32, hp: i32 = 100, x: f32, y: f32, name: u8[] }

let scout = Unit { kind: 2, x: 1.5, .. };   // hp 100, y 0, name ""
var selected: Unit;                         // all defaults (globals may be type-only)
```

A literal names its fields or lists them positionally; trailing `..` fills any
omitted field with its type's default (zero / false / null / empty array).

## Control flow: everything is an expression

```goose
let kind  = if hp > 50 { "healthy" } else { "hurt" };     // no ternary operator
let level = match hp { 0 => "dead", 1..50 => "low", _ => "fine" };
let verdict = block {
    guard p.alive else { break "gone"; }
    guard p.hp > 0 else { break "dead"; }
    "standing"
};
```

- `guard c;` runs the rest of the block only where `c` holds;
  `guard c else { … }` runs the else block instead (put `return`/`break` there
  to leave more than the block).
- `defer stmt;` runs when the enclosing block ends, whichever way it ends.
- `block { … }` gives `break` a target without a loop; a block's value is its
  trailing expression.

## Loops

```goose
for i in 3 { }              // 0, 1, 2
for i in 10..13 { }         // half-open: 10, 11, 12
for i: u8 in 0..256 { }     // i at a chosen type
for x in xs { }             // copies fixed-size elements
for &x in xs { x *= 2; }    // by reference: the mutation form
for x, i in xs { }          // with the index (i64)
while cond { }
loop { if done { break; } }
```

## Multiple return values are a calling convention

No tuple type. A call in value context yields the first result;
`let a, b = f();` binds all:

```goose
fn divide(a: i64, b: i64) -> i64, bool {
    if b == 0 { return 0, false; }
    return a / b, true;
}
let q, ok = divide(7, 2);
```

By convention a trailing `bool` says whether the other values mean anything.

## Default parameters (trailing only)

```goose
fn limit(x: i64, lo: i64 = 0, hi: i64 = 100) -> i64 {
    if x < lo { lo } else if x > hi { hi } else { x }
}
print(limit(250), " ", limit(-5, -10));   // 100 -5
```

Defaults are evaluated per call and may use globals — never the caller's
locals.

## Printing

```goose
print("answer=", 42, " ratio=", 0.75, " yes=", 3 > 2);
```

No format strings. `format(out, …)` appends to a builder, `str(…)` makes a
fresh string; a `format` overload customizes how a type renders.

## The three arithmetic rules that bite newcomers

1. **Arithmetic happens at the operands' own width.** `u8 + u8` is an 8-bit
   add. Wider widening is implicit; nothing narrows without `as` (checked in
   debug) or `as!` (never checked). `u32 + i32` is a compile error.
2. **`%` is Euclidean** — the result is in `[0, |b|)`, never negative:
   `-7 % 3 == 2`. Integer `/` truncates toward zero; import `std` for
   `rem(a, b)` (pairs with `/`) and `div_euclid(a, b)` (pairs with `%`).
3. **Unsigned arithmetic always wraps** (that is what hashes/PRNGs mean by
   unsigned math); signed overflow aborts in debug and wraps in release.

## Minimal program shape

```goose
fn main() {
    print("hello, goose");
}
```

`fn main()` in the global namespace of the root file is the entry point.
Program arguments come from the `os` module: `args()`.

## Related skills

- [scalars-and-arithmetic](../../types/scalars-and-arithmetic/skill.md)
- [expressions-if-match](../../control-flow/expressions-if-match/skill.md)
- [loops](../../control-flow/loops/skill.md)
- [functions-parameters-results](../../functions/functions-parameters-results/skill.md)
