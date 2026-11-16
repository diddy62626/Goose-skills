---
name: goose-reading-errors
title: Reading compiler errors — the two-ends rule
category: idioms
tags: [goose, errors, debugging, lifetime errors, instantiation chain, overload errors]
triggers: [goose error message, lifetime error reading, instantiation chain, overload candidates]
description: >
  Goose errors name both ends of the problem. How to read lifetime errors
  (reference + binding site), shrink errors (holder + store site), generic
  errors (instantiation + call chain), overload errors (candidates + why
  each failed), and the decision each message is asking you to make.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#reading-the-errors
---

# Reading compiler errors

> When the checker rejects something, read the whole error. It names both
> ends of the problem — the reference *and* where it was bound, the holder
> *and* the line that stored into it, the instantiation *and* the call
> chain that produced it. Use those locations to decide which binding,
> store, or scope needs to change.

## Lifetime / shrink errors

```
error: cannot clear `line` while `v` (bound at x.goose:6) is still used
```

The two locations: where the reference/slice was **bound**, and the
**operation** it conflicts with. Your options, in order of preference:

1. Use the view for the last time before the shrink (reorder).
2. Move the shrink later (or into a block that ends first).
3. Move the declaration of the reference into a narrower scope.
4. Copy the data out (`copy`, or build a new string) if the lifetime
   genuinely must extend.

For a **holder** error ("`holder` may hold a reference into `line`, stored
at …"), the store site tells you which push/assignment put the reference
there; the fix is the same shape.

A "may point into X or Y" message lists the **candidates** — variables
whose storage could hold the target. Narrow them: a relative link `in
pool` points exactly into `pool`; keeping stores to a holder in one place
lets the checker track its contents exactly.

## Generic instantiation errors

A generic body is checked only at concrete instantiations; the error shows
the **compile-time call chain**:

```
in fn push_n<A, T> instantiated from build.goose:41
in fn main instantiated from build.goose:50
error: the push takes a u8
```

Read bottom-up: the innermost frame is the failing operation; the outermost
is your call. The usual causes: a literal that cannot adapt (`push_n(xs,
300, n)` into a `u8` array), an untyped parameter that inferred a type you
did not mean, or an operation the concrete type does not support. Fix at
the call (annotate the variable, cast, or pass the right type), not by
loosening the generic body.

## Overload errors

```
error: call is ambiguous
  candidate f(x: i32): exact match
  candidate f(x: f64): 2 converts
```

Candidates are listed with the **reason each failed or ranked**. Common
cases: `sqrt(n)` with an integer `n` (ambiguous between widths — write
`sqrt(n as f64)`); `f(a, b = 0)` beside `f(a)` making one-argument calls
ambiguous (defaults don't count as arguments); same-rank coercions. Cast an
argument or split the overload set.

## Warnings

- ``redundant `as i64`: a u8 converts to i64 implicitly here`` — delete the
  cast, or make it decide something (a width, a rounding) if you meant it.
- A second returned local stopping in-place construction — return one
  local, or restructure.
- `&` written where a reference destination binds anyway.

## When it still does not compile

- A checkfree error lists **the chain of calls** that lead to the
  surviving check — the innermost helper keeps it.
- A `return … from` outside its target's extent points at the call that
  needs wrapping.
- Thread errors name the **non-flat type** and the field that is not flat
  (`cells: u8[:]` inside a queued struct).
- Root-struct errors name the placement rule violated (owner as a field /
  element / by value).

## Related skills

- [common-pitfalls](common-pitfalls/skill.md)
- [lifetimes-roots](../references/lifetimes-roots/skill.md)
- [diagnostics](../tooling/diagnostics/skill.md)
