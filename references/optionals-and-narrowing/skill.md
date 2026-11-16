---
name: goose-optionals-and-narrowing
title: Optionals — T?, null, and narrowing with if/while/guard/assert
category: references
tags: [goose, optional, nullable, null, narrowing, T?]
triggers: [goose optional, goose null, nullable reference, narrowing, T?, guard null]
description: >
  T? is a nullable reference with no space cost (null is address 0). It is
  not transparent: narrow it with if/while/guard/assert or a comparison with
  null; inside the narrowed region it IS a T&. Covers rebinding semantics,
  while-loops over linked lists, and the null-returning library idioms.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#optionals
---

# Optionals (`T?`)

References are never null. `T?` is an **optional `T`**: a nullable reference,
represented as a reference whose null is address 0 — **no space cost**. It
composes with any type (`i64?` is an optional integer, by reference), and on
a reference type it only makes that reference nullable (`T&?` is `T?`). The
literal `null` is its empty value; an optional struct field with no declared
default is null when left out.

## Narrowing is required

An optional is **not transparent**: use it only after `if`, `while`,
`guard`, `assert`, or a comparison with `null` has narrowed it. Inside the
narrowed region it *is* a `T&`: transparent, writable through, bindable with
`.=`. Rebinding the variable ends the narrowing — except in a `while` whose
condition is the test, which re-tests before each iteration.

```goose
fn first_at_least(xs: i64[:], min: i64) -> i64? {
    for &x in xs { if x >= min { return x; } }
    return null;
}

let big = first_at_least(xs, 5);
if big { print("first >= 5 is ", big); } else { print("none"); }
print(none == null);
```

## Walking a linked list

```goose
struct Node { v: i64, next: Node? }
var cur: Node? = a;
var sum = 0;
while cur {                      // narrowed for the body, re-tested each time
    sum += cur.v;
    cur .= cur.next;
}
```

## guard and assert narrow too

```goose
fn tenfold(p: i64?) -> i64 { guard p else { return 0; } p * 10 }
assert(o);                        // narrows for the rest of the block
```

After `guard r else { return; }`, `r` is a `T&` for the rest of the block.

## Rebinding

An optional rebinds with `.=` to a reference (which narrows it) or to
`null` (which does not). A narrowed optional writes through like any
reference; there is no way to write through one without narrowing.

## The library idioms

- **"Maybe an element" is a `T?`**: `xs.find() { it > 100 }` returns a
  reference into the input, or null — copies nothing.
- `dictionary::get` returns `V?`; `contains` distinguishes an absent key
  from a null value.
- Positions are `i64` with `-1` for none (`find`, `position`) — a different
  convention, used where an index rather than a reference is natural.
- Two-outcome scalars return a trailing `bool` (`let v, ok = parse_int(s);`)
  instead of an optional.

## Do not confuse

- `if (n)` on an integer is an error — write `if n != 0`. `if` on an
  *optional* narrows it.
- `T?` vs `T&`: the former costs nothing but must be narrowed; the latter is
  already a reference the compiler proved non-null.
- `from_bytes` and external formats: an optional relative link
  `T&<u32>?` uses offset 0 as null — four bytes including its null.

## Related skills

- [references-and-ufcs](references-and-ufcs/skill.md)
- [error-idioms](../errors/error-idioms/skill.md)
- [relative-references](relative-references/skill.md)
