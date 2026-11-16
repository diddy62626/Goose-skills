---
name: goose-inline-nested-data
title: Inline nested data — varint records, byte counting, sequential trade-offs
category: data-modeling
tags: [goose, varint, nested data, inline, records, compact, sequential]
triggers: [goose varint, inline nested data, compact records, bytes per record, sequential array]
description: >
  Structs keep variable-size parts inline in declaration order; varint
  fields shrink integers to their value's length. A whole order book is one
  packed array of orders with no pointers. The 29-byte order worked example,
  when to use variable-size elements (walked data) vs fixed (indexed data).
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md#8-nested-data-stays-inline
---

# Inline nested data

A struct may contain variable-size parts, and they sit **inline**, in
declaration order. Combined with `varint` fields (a LEB128 integer that
takes as many bytes as its value needs), a record becomes a run of bytes
with no indirection in it.

## The worked example

```goose
struct Item  { sku: u8[varint], qty: varint, cents: varint }
struct Order { id: varint, customer: u8[varint], items: Item[varint] }

// "SKU:qty:cents;..." -> items, built straight into the field that gets them
fn parse_items(field: u8[:]) -> Item[>..] {
    var items: Item[>..] = [];
    each_split(field, ';') {
        let p = split(it, ':');
        let qty, ok1 = parse_int(p[1]);
        let cents, ok2 = parse_int(p[2]);
        assert(ok1 && ok2);
        items.push(Item { sku: p[0], qty: qty, cents: cents });
    };
    return items;
}

var book: Order[>..] = [];
book.push(Order { id: 1001, customer: "alice",
                  items: parse_items("SKU-441:2:1999;SKU-7:1:500") });
```

That order (id 1001, customer `alice`, two items with 5- and 7-character
SKUs) takes:

| | bytes | allocations |
|---|---:|---:|
| Goose, as declared | **29** | **0** |
| C++ `std::string` + `std::vector<Item>` | 160 | 1 |
| Rust `String` + `Vec<Item>` | 153 | 4 |

`id`, `qty` and `cents` take one byte each rather than eight. The smaller
layout reduces cache traffic when traversing the order book.

## The trade-off: sequential arrays

Variable-size elements make the array **sequential**: offsets are
data-dependent, so you can iterate it but not index it.

```goose
for o in book { print(o); }     // fine
// print(book[0]);              // error: arrays of variable-size elements
                                //        cannot be indexed, only iterated
```

- Use variable elements for data you **traverse**: records, parsed nodes,
  log lines.
- Use fixed elements for data you **index and sort**, with a `u8[..k]` if an
  element needs a string.
- Need both? Keep a second array of references as an index (see
  index-for-sequential-arrays).

## varint sizing

A LEB128 integer: 1 byte up to 63, two up to 8191, and so on, up to ten
bytes; small negatives are as short as small positives. A `varint` field is
written only when its containing value is constructed; reads give an `i64`.
A stored length must hold the actual count (`u8[varint]` string fields, and
length types like `T[u8]` maxing at 255 elements, are checked statically
where possible, at run time otherwise).

## Frozen once built

A variable-class field or element is frozen once built: to change one,
rebuild its container. For mutable text in an indexed record use
`u8[..k]` (fixed capacity, assignable).

## Related skills

- [fixed-limited-frozen-arrays](../arrays/fixed-limited-frozen-arrays/skill.md)
- [index-for-sequential-arrays](index-for-sequential-arrays/skill.md)
- [structs](../types/structs/skill.md)
