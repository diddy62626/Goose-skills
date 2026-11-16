---
name: goose-expressions-if-match
title: Expressions — if, match, block as values; branch type joining
category: control-flow
tags: [goose, if, match, expression-oriented, block value, literal adaptation]
triggers: [goose if expression, goose match, ternary, branch types, block value]
description: >
  if, match, block and bare {} are expressions; a block's value is its
  trailing expression. Used as values, ifs need else, match needs full
  coverage. Literal branches adapt; slices join as slices; statements vs
  expressions and semicolon rules; functions that never return.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#16-expressions-and-control-flow
---

# Expressions: `if`, `match`, `block`

The language is expression-oriented: `if`, `match`, `block` and a bare
`{ … }` are expressions, and a block's value is its trailing expression.
Assignment, `++` and `--` are statements. Operator precedence is Rust's.

## `if`

```goose
let hp = 30;
let kind = if hp > 50 { "healthy" } else if hp > 0 { "hurt" } else { "down" };
if hp > 0 { print(kind); }               // as a statement: no else needed
```

Used as a value, an `if` needs a final `else`, and its branches join into
one type. A literal branch adapts to the other's type; an integer branch
beside a float branch converts; a construct whose branches are all literals
is itself literal-like and adapts wherever it meets a type. Arrays/slices of
one element type join as a *slice*: one branch a string literal, the other a
`u8[..16]` field → the result is a `const u8[:]` viewing the field, nothing
copied.

```goose
let name = if item.count == 0 { "none" } else { item.name };   // const u8[:]
let k: u8 = if item.count > 0 { 1 } else { 2 };                // literals adapt to u8
```

To use a block-ended construct as an operand, parenthesize it:
`let v = (if a > 2 { 1 } else { 2 }) + 3;`.

## `match`

Exhaustive over enums (every variant or `_`) and over integers (literals,
named constants, half-open ranges, comma lists; `_` required; first match
wins). Arms bind payloads by value (`Circle c =>`) or by reference
(`Circle &c =>`, variable-mode scrutinee only). See enums-adts for the full
story.

## `block { … }` and bare blocks

`block { … }` gives `break` a target without a loop, so a chain of checks
bails out with a value. A bare `{ … }` is only a scope. All `break E` of one
construct must agree on E's type.

```goose
let verdict = block {
    guard p.alive else { break "gone"; }
    guard p.hp > 0 else { break "dead"; }
    "standing"
};
let doubled = { let t = p.hp + 1; t * 2 };
var n = 0;
let found = loop { n += 7; if n % 5 == 0 { break n; } };
```

## Statements and semicolons

An expression statement ends in `;`. A statement that *is* a block-ended
construct (`if`, `match`, `block`, a loop, `guard … else { }`, a nested
`fn`, a call with a trailing function block) needs none — it ends at its
closing `}`. An operator never continues such a statement. A stray `;` after
a block is harmless. Every bracketed comma-separated list accepts a trailing
comma.

## Branches that cannot supply a value

A trailing construct that cannot supply a value is a statement: an `if`
without else, a `match` with an arm ending in `return`, a `loop` none of
whose `break`s carries a value. A function whose result is inferred may end
in one and produce nothing.

## Functions that never return

`abort(msg)` and `exit(code)` never return, and the checker uses this: a
branch that ends in one gives no value, statements after one are not
checked, and it propagates through calls — a function may end in a call to
one without a dummy result.

```goose
fn fail(what: u8[:]) { abort(str("cannot ", what)); }
fn value(ok: bool) -> i64 {
    if ok { 42 } else { fail("compute") }     // no dummy result needed
}
```

An unknown recursive call is conservatively assumed to return.

## Related skills

- [guard-block-defer](guard-block-defer/skill.md)
- [enums-adts](../types/enums-adts/skill.md)
- [error-idioms](../errors/error-idioms/skill.md)
