---
name: goose-const-writability
title: const and writability — read-only contents, slots, inference
category: references
tags: [goose, const, writability, read-only, const u8[:], slots]
triggers: [goose const, read-only, const u8[:], writability, const slice, immutable]
description: >
  Everything is writable unless its type says otherwise. const T is a value
  whose contents cannot be written; writability is inferred per
  instantiation; parameters are generic over constness; slots (fields,
  elements, globals) say what they may hold. const is shallow.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#writability
---

# `const` and writability

Everything is writable unless its type says otherwise. `const T` is a value
whose contents cannot be written: a `const i64[3]` cannot have an element
assigned; `const T&` and `const T[:]` are a reference and a slice through
which the target cannot be written, grown or shrunk.

## Writability is inferred, not annotated

- **Read-only**: a string literal; `&x` or `x[..]` of a `const` value, of a
  field of one, or of a by-value `for`/`match` binding; a view into a
  temporary; a `bytes_of` view; whatever is read out of a slot declared
  `const`.
- **Writable**: everything else, including `&x` and `x[..]` of a `var` or
  plain `let`.

## let / var / const in one table

```goose
let xs = [1, 2, 3];            // name not reassignable; contents writable
xs[0] = 9;                     // fine
const ys = [1, 2, 3];          // = let of type const T: contents read-only
// ys[0] = 9;                  // error
var s: const u8[:] = "text";   // a read-only VIEW that can be re-pointed
s = s[1..];
```

`const x = e;` declares a `let` of type `const T`. A `var` of a `const` type
is the converse: reassignable but never written into.

## Parameters and results are generic over constness

A parameter declared `u8[:]` takes a writable or read-only argument, and the
body is checked with what it is given — a write through a parameter given a
literal is an error at that call, reported with the call chain. A parameter
declared `const u8[:]` is read-only whatever it is given: how a function
documents that it only reads. Adding `const` is implicit everywhere;
dropping it never is.

```goose
fn upper_first(s: u8[:]) { if s.len > 0 && s[0] >= 'a' { s[0] -= 32; } }
upper_first(buf);               // writable argument: fine
// upper_first("literal");      // error: writes through a read-only parameter
```

## Slots say what they hold

A field, element, global or annotated variable holds a read-only reference
only if its type is `const`:

```goose
struct Named { name: const u8[:] }   // can hold a literal / split() result
struct Buf   { bytes: u8[:] }        // cannot hold a literal
```

An un-annotated variable takes its initializer's constness:
`var t = trim(line);` for a read-only `line` is a `const u8[:]` cursor.
That is why dictionary keys are written `const u8[:]` — literals and the
read-only views `split` returns must be usable as keys.

## const is shallow

A reference read out of a field of a `const Box&` is as writable as the
field's own type says. Read-only data cannot be laundered into writable
data.

## What cannot be sorted/written

A `let` array can be sorted (`let` prevents rebinding, not element writes).
A `const` array or read-only slice cannot be sorted, written, grown or
shrunk. `bytes_of(a)` views are never writable — bytes written through them
would be links the checker never proved.

## Related skills

- [lifetimes-roots](../lifetimes-roots/skill.md)
- [dictionary](../../stdlib/dictionary/skill.md)
- [serialization](../../data-modeling/serialization/skill.md)
