---
name: goose-in-place-construction
title: In-place construction — results are built at their destination
category: memory-model
tags: [goose, construction, return-values, str, destination, no-temporaries]
triggers: [goose in place construction, return array by value, str built at destination, no temporary]
description: >
  The copy-free construction guarantee: a constructed non-fixed value is
  always built in its final location, through any depth of calls and through
  whichever branch an if/match takes. Returning a built-up value costs the
  same as an out-parameter; str(...) writes directly into the element being
  pushed. Plus the construction-in-progress restriction.
source: https://github.com/aardappel/goose/blob/main/docs/goose_spec.md#43-the-in-place-construction-guarantee
---

# In-place construction

A constructed value that is not fixed-size is built **directly in its final
location** — the caller's variable, the element being pushed, the argument
slot of another call, a field of a record being built inside another array —
through any depth of calls, and through whichever branch an `if` or `match`
takes. No temporary is ever built and then copied into place.

## Returning a built-up value is free

```goose
fn parse_items(field: u8[:]) -> Item[>..] {
    var items: Item[>..] = [];
    each_split(field, ';') { /* push into items */ };
    return items;                     // written straight into the caller's slot
}

var book: Order[>..] = [];
book.push(Order { id: 1001, customer: "alice",
                  items: parse_items("SKU-441:2:1999;SKU-7:1:500") });
// parse_items built the items directly in the new Order's field, inside book.
```

In Goose, returning a built-up value costs the same as writing through an
out-parameter, which is why out-parameters rarely appear. Prefer return
values.

## `str(...)` builds at its destination

```goose
words.push(str("word", i));         // formats the text directly into the new element
```

There is no temporary string that is then copied into the array.

## The `return local` shortcut

When every `return` of a non-fixed result returns the same local, that local
*is* the destination from its declaration on:

```goose
fn squares(n: i64) -> i64[>..] {
    var out: i64[>..] = [];
    for i in n { out.push(i * i); }
    return out;           // `out` *is* the caller's variable from its declaration on
}
let sq = squares(10);     // no copy
```

A function's own local is *moved* by `return`; a part of it (or of a
temporary) dies with the call and must be returned as `copy(x)`. The
compiler warns where a second returned local stops the first one from being
allocated at the destination.

## Construction in progress: nothing may grow the receiving array

While a value is being built in place inside an array (an element being
pushed, a call's result being appended, the new contents of a whole
assignment), nothing may grow that array — neither the expression being
built nor a function it calls:

```goose
// xs.push(f(xs))  where f pushes into its argument: REJECTED
```

This is what keeps "one resizable value per stack, at the top" provable.

## Fresh values with relative links

A function or trailing block can return a fresh value that holds
self-relative links; it is constructed at the receiver, including across
calls and with optimization disabled. The receiver must share the root array
the links point into (`nodes.push(linked(nodes[0], 2))` is fine;
`let n = linked(nodes[0], 2)` is an error — the local is separate storage).

## Related skills

- [copies-and-aliasing](copies-and-aliasing/skill.md)
- [relative-references](../references/relative-references/skill.md)
- [strings-as-byte-arrays](../strings/strings-as-byte-arrays/skill.md)
