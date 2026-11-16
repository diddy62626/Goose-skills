---
name: goose-slices
title: Slices — T[:] views that cannot dangle
category: arrays
tags: [goose, slice, span, view, substring, lifetime, checked]
triggers: [goose slice, T[:], string_view equivalent, goose span, slice ranges, ^k index]
description: >
  T[:] is a reference plus a count (like span/string_view/&[T]) that every
  array kind converts to without copying. Slicing syntax including ^k
  from-the-end, coercion rules, checked roots (cannot outlive the owning
  variable, cannot survive a shrink), writability, and slices as dictionary
  keys.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#7-slices
---

# Slices (`T[:]`)

A reference plus an element count: a view of a contiguous run of `T`s inside
some array. The universal "process a range" parameter type, since every
array kind converts to a slice without copying its elements.

```goose
fn total(xs: i64[:]) -> i64 { var t = 0; for x in xs { t += x; } return t; }

total(fixed);         // a T[k]
total(grow);          // a T[>..]
total(grow[1..4]);    // a sub-range
total(grow[^2..]);    // ^k counts from the end: last two elements
```

## Slicing syntax

- `a[x..y]` — elements `x` up to `y` (exclusive); either bound may be left
  out.
- `^k` means "`len - k`": `a[^2..]` is the last two elements.
- Re-slicing a slice is always allowed and never grows it.
- `[..cap]` in a *construction* position (not slicing) makes an empty
  `T[..]` with that capacity.

## Coercion: any array becomes a slice of itself

When an array (or a reference to one) reaches a destination of slice type,
it becomes a slice of that whole array, as `a[..]` would — a parameter, an
annotated variable, an assignment, a field, a pushed element, a `return`, or
a branch of an `if` whose other branch is a slice:

```goose
let view: const i64[:] = fixed;   // fixed itself, not a copy
fixed[0] = 99;
print(view[0]);                   // 99
```

## Slices are checked — the root system

Every slice carries a static **root**: the variable that bounds the lifetime
of what it views. A slice can neither outlive that variable nor survive a
shrink that would free the elements. The checker tracks liveness, not scope:
a slice whose last use is before a `clear` is dead, and the `clear` is
allowed.

```goose
var line: u8[>..] = [];
line.append("hello world");
let w = line[..5];
print(w);
line.clear();                         // fine: w's last use is above
line.append("again");
let v = line[..];
// line.clear();                      // error: cannot clear while v is still used
print(v);
```

A slice of a grow-only array taken before a `push` stays valid, but does not
include the new elements.

## Writability

A slice follows the reference writability rule: a slice of a `var` array is
writable through; a string literal and a slice of a `const` value are not. A
parameter declared `T[:]` accepts either and is checked against the argument
(actual writes through a read-only argument are errors at that call).
Declare `const T[:]` to document read-only.

## Iterating and indexing

Slices of fixed-size elements index and iterate, bounds-checked against the
slice's own length. Slices of variable-size elements iterate only.

## Slices as keys

Because a slice never copies, it is the natural dictionary key — the table
stores addresses and lengths into text you already hold:

```goose
import dictionary;
var counts = dictionary<const u8[:], i32> {};
each_split(text, ' ') { counts.update(it, 0) { it += 1; }; };
```

`const` is needed because literals and `split` results are read-only views.

## Temporaries

A view of a temporary (`let u = make()[..];`) is an error: take the value
first (`let t = make(); let s = t[..];`).

## Related skills

- [array-family-overview](../array-family-overview/skill.md)
- [lifetimes-roots](../../references/lifetimes-roots/skill.md)
- [parsing-text](../../strings/parsing-text/skill.md)
