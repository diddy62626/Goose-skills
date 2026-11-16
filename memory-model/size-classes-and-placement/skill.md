---
name: goose-size-classes-and-placement
title: Size classes — fixed, variable, resizable, root — and what may contain what
category: memory-model
tags: [goose, size-classes, structs, placement, root-struct, varint]
triggers: [goose size class, fixed variable resizable, root struct placement, what may contain what]
description: >
  Every Goose type is fixed, variable, resizable, or root. This decides where
  a value may live: which fields, which array element slots, which parameter
  and result positions. The complete placement rules with the struct-tail and
  element restrictions.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#size-classes
---

# Size classes and placement

## The four classes

| Class | Meaning | Examples |
|---|---|---|
| **fixed** | byte size is a compile-time constant | scalars, structs of fixed fields, `T[k]`, `T[..k]`, references `T&`, slices `T[:]`, fixed-mode enums |
| **variable** | size chosen at construction, frozen after | `T[]`, `T[varint]`, `T[..]`, a struct with a `varint` or variable field, a variable-mode enum (`Shape..`) |
| **resizable** | size may change after construction | `T[>..]`, `T[>..<]`, a struct whose last field is one of them |
| **root** | `root struct`: fields with independent storage under one lifetime | `slot_pool<T>`, your own workspace structs |

An ordinary compound type's class is the largest of its parts' classes. A
type is **flat** when it holds no reference, slice or relative reference at
any depth and is not a root struct — only flat values may cross between
threads.

## Placement rules

- **Fixed** values go anywhere: locals, globals, any field position, array
  elements, enum payloads, parameters, results.
- **Variable** values go in locals/globals, parameters/results, enum
  payloads, and array elements (which makes the array *sequential*) and
  struct fields at any position (fields after a variable-size one are found
  by walking).
- **Resizable** values go in locals/globals, by-value parameters, results,
  and the **last field** of an ordinary struct (making the struct resizable).
  **Never an array element.**
- **Root** owners go only in local/global declarations. A root cannot be
  owned inside another struct, an array or slice (even fixed-size), an enum
  payload; it cannot be passed or returned by value. References to roots
  follow the ordinary reference rules (`Workspace<T>&`).
- **Zero-size** values (`T[0]`, `T[..0]`, a variant with no fields) exist
  only as values of their own, never as a field or element.
- No type contains itself by value; recursion goes through references or
  slices, which are fixed-size whatever they point at:
  `enum List { Nil, Cons { v: i64, next: List? } }` is fine.

## Element restrictions per array kind

- Elements of `T[>..<]`, `T[..k]`, `T[..]` must be **fixed-size** (a shrink
  or storage reuse must never change what type a location holds).
- Elements of `T[]` and `T[>..]` may be fixed or variable, never resizable.
- An array with variable-size elements is **sequential**: where an element
  starts depends on every element before it, so it can be iterated but not
  indexed, and has no `pop`. Keep an index of references beside it if you
  need random access.

```goose
var words: u8[][>..] = [];      // variable-size elements: iterate only
for w in words { print(w); }    // fine
// print(words[0]);             // error: cannot index, only iterate
```

## Choosing where data lives: worked examples

```goose
struct Order { id: varint, customer: u8[varint], items: Item[varint] }
var book: Order[>..] = [];      // one packed block of orders; iterate

struct Stack { name: u8[..8], items: i64[>..<] }   // one resizable tail: fine

root struct Workspace<T> {      // several independently growing fields
    items: T[>..] = [],
    scratch: i64[>..<] = [],
    calls: i64 = 0
}
```

A variable-class field or element is **frozen once built**: to change one,
rebuild its container. Fixed-size fields of the same record are assignable
freely.

## A struct that must match a foreign layout

Structs are **packed** (no padding); a struct of an `i8[3]` and an `i32` is
7 bytes. For foreign layouts add explicit `pad` fields; a bare `pad` aligns
the next field to its scalar width. Pad bytes are never read and `==`
ignores them.

```goose
struct Hdr { magic: u8[4], pad, size: u32, let kind: u8, const name: u8[..8] }
```

## Related skills

- [data-stacks-and-ownership](../data-stacks-and-ownership/skill.md)
- [root-structs](../../data-modeling/root-structs/skill.md)
- [inline-nested-data](../../data-modeling/inline-nested-data/skill.md)
- [common-pitfalls](../../idioms/common-pitfalls/skill.md)
