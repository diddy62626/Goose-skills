---
name: goose-copies-and-aliasing
title: Copies are visible — copy(), binding by reference, call semantics
category: memory-model
tags: [goose, copy, reference, aliasing, assignment, semantics]
triggers: [goose copy, goose assignment semantics, implicit copy, binds by reference, copy function]
description: >
  Fixed-size values copy silently like C; non-fixed values NEVER copy
  implicitly. `let b = a;` binds b to a by reference for growable arrays.
  `copy(a)` is the explicit O(n) copy. How this works for variables, fields,
  elements, and call arguments, plus the generic-code `.= for values` rule.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#copies-are-visible
---

# Copies are visible

## The rule

Fixed-size values (scalars, structs of fixed fields, fixed arrays, references,
slices) copy silently on assignment, like C. **Non-fixed values (variable or
resizable class) never copy implicitly**, because that copy is O(size) and
you should see it in the source.

```goose
var a: i64[>..] = [1, 2, 3];
var b = copy(a);      // an explicit, visible O(n) copy
b.push(4);
let c = a;            // no copy: c names the same array as a
c.push(5);
print(a, " ", b);     // [1, 2, 3, 5] [1, 2, 3, 4]
```

Without the `copy`, `var b = a;` binds `b` to `a` by reference.

## Where binding by reference happens

A variable/field/element of variable or resizable class **binds by reference**
when it reaches a destination that is:

- a reference, a slice,
- an untyped parameter,
- an un-annotated `let`/`var`,
- a `for` binding.

At a destination that needs a value of its own, it is an error. A value-typed
destination takes a fresh value (a literal, a call's result) or an explicit
`copy(x)`.

```goose
struct Item { id: i32, weight: f32 }
var p = Item { 1, 0.5 };
var q = p;                // fixed-size struct: a silent copy
q.id = 2;                 // p.id is still 1
```

## Calls follow the same rule

`f(xs)` passes a growable array by reference, without copying elements,
whether the parameter is `xs: i64[:]`, `xs: i64[>..]&`, or untyped `xs`.
A function that needs its own copy calls `copy`. A by-value non-fixed
parameter (`xs: u8[>..]`) is constructed in the argument slot and owned by
the callee like a local — pass `f(copy(xs))` or `f(make())`.

## Returning a local moves it

When every `return` of a non-fixed result returns the same local, that local
*is* the destination from its declaration on — `return xs;` copies nothing.
A *part* of the local, or of a temporary, dies with the call: return
`copy(x)` for those.

## Generic code: `.=` moves whole values

Inside generic code, `.=` on a location that is not a reference assigns as
`=` does, letting generic code move whole values of a type parameter's type.
Where `T` is a reference type, `.=` moves the reference itself (where `=`
would write through it); where `T` is a value, it is a plain assignment. The
library's `swap_at`, `sort`, `reverse` etc. are written this way, which is
why they can reorder arrays of references (including relative ones).

```goose
fn swap_at<A, T>(xs: A&, i: i64, j: i64) {   // sketch of the idiom
    let t .= xs[i];
    xs[i] .= xs[j];
    xs[j] .= t;
}
```

## `default<T>()` and copies

`default<T>()` builds a fixed-size `T` from its declared defaults; a default
value (for example a literal's `..` setting a relative field to null) is the
one copy that is always allowed of a value that holds self-relative links.

## Related skills

- [in-place-construction](in-place-construction/skill.md)
- [references-and-ufcs](../references/references-and-ufcs/skill.md)
- [data-stacks-and-ownership](data-stacks-and-ownership/skill.md)
