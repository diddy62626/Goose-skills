---
name: goose-references-and-ufcs
title: References — transparent, .= binding, &x, UFCS method syntax
category: references
tags: [goose, references, ufcs, transparent, rebind, .equals]
triggers: [goose reference, T&, .= operator, .== identity, method syntax, free functions]
description: >
  T& is a machine address with no arithmetic and no way to dangle.
  References are transparent (behave as their target everywhere; no * or
  ->). .= binds/rebinds a reference itself, .== compares addresses. &x
  makes a reference; reference destinations bind without &. UFCS:
  x.f(a) calls f(x, a).
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#10-references-and-optionals
---

# References and UFCS

## `T&` — one address, never dangling

`T&` is a reference to a `T`: one machine address, **no pointer arithmetic,
no casts, no way to make it dangle**. What it may point to is checked by the
lifetime system (see lifetimes-roots).

## References are transparent

An expression that denotes a reference behaves as its target in every value
context: reads, arithmetic, comparisons (`r1 == r2` compares targets),
passing to by-value parameters, `print(r)`. There is no `*` and no `->`. A
write through a reference, `r = v` or `r += v`, writes the target.

Since the reference itself is invisible in ordinary expressions, separate
syntax means "the reference, not the target":

- `let r .= e;` / `var r .= e;` — bind `r` as a reference to `e` (variable,
  field, element, or a call that returns a reference).
- `r .= e;` — rebind an existing reference variable/field/element.
- `r1 .== r2` / `r1 .!= r2` — compare by address.

```goose
var xs: i64[>..] = [10, 20, 30];
var r .= xs[0];              // r refers to element 0
var c = xs[0];               // c is a copy of element 0
r = 11;                      // writes xs[0]
c = 99;                      // writes the copy
r .= xs[2];                  // r now refers to element 2
r += 1;
print(r .== xs[2], " ", r == xs[2]);   // true true
```

## Making references: `&x` and implicit binding

`&x` makes a reference to an lvalue; `&` of a temporary is an error; `&` of
a location holding a reference yields the stored reference (no references to
references).

Where the destination's declared type is a reference, an lvalue of the
pointee type binds **without** `&` (writing it is a warning):

```goose
fn inc(n: i64&) { n += 1; }
var count = 0;
inc(count);                        // binds by reference
let r: i64& = count;               // annotated reference variable
struct Cursor { line: i64& }
let cur = Cursor { line: count };  // reference field binds without &
```

Inference rules for un-annotated bindings:

- Explicit `&lvalue` initializer or `.=` form → a reference.
- A reference to a **fixed-size** value decays to a copy (`let d = r;`
  copies; `let e .= r;` stays a reference).
- A **non-fixed** lvalue binds by reference whatever the form (non-fixed
  values never copy implicitly).

## Functions returning references

A result type written `T&` returns a reference; an inferred result returns a
reference where it returns a non-fixed lvalue or explicit `&x`, and a copy
where it returns a reference to a fixed-size value. Bind
reference-returning calls with `.=`: `let e .= pool.push(v);`.

`let` reference variables cannot be **rebound**, but their target is as
writable as before: `let r .= xs[i]; r = 5;` writes the element.

## UFCS: method-call syntax for free functions

`x.f(a)` calls `f(x, a)` for any function. Resolution tries fields and
built-in members first, then functions starting in the namespace of `x`'s
type — so `p.norm1()` finds `geo::norm1` for a `geo::Pt` from any namespace,
even one with its own `norm1`; a plain call `norm1(p)` does not.

```goose
xs.sort();                 // sort(xs)
d.insert(k, v);            // dictionary insert
2.5.describe();            // describe(2.5)
r.rand_int(6);             // std's rng
```

There are no methods, no `impl` blocks; group a type's functions in its
namespace. The `format` hook follows the type the same way: `print(p)` finds
`geo::format` for a `geo::Pt`.

## What references may point to

Anything except a fixed-mode enum's payload. Into a grow-shrink array
`T[>..<]`: variables only, and not used after the next shrink. Into
grow-only arrays: valid as long as they can be named. Into limited arrays:
valid across pops (a stale one reads the slot's current value).

## Related skills

- [optionals-and-narrowing](../optionals-and-narrowing/skill.md)
- [lifetimes-roots](../lifetimes-roots/skill.md)
- [relative-references](../relative-references/skill.md)
