---
name: goose-json
title: json module — writer, tree parser, and streaming reader onto your types
category: stdlib
tags: [goose, json, parse, writer, reader, tree, serde]
triggers: [goose json, json parse, json writer, json reader, format_key, json each_member]
description: >
  JSON three ways: a writer appending values to a builder; a parser building
  a document tree (Node[>..] with relative links; keys are slices of the
  text, escapes decoded into a scratch builder); and a zero-build Reader
  mapping text directly onto the caller's types, serde-derive style, with
  sticky errors.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#json
---

# The `json` module

JSON as RFC 8259, three ways, all in namespace `json` (`import json;`).

## 1. Writing

```goose
out.push('{');
json::format_key(out, "name");           // "key":
json::format_string(out, player.name);   // quoted, " \ and controls escaped
out.push(',');
json::format_key(out, "pos");
out.push('[');
json::format_number(out, player.x);      // shortest form that reads back
out.push(',');
json::format_number(out, player.y);
out.append("]}");
```

The caller writes brackets and commas. Bytes from 0x80 up are written
unchanged (UTF-8 stays UTF-8). `format_number` matches serde_json's layout;
writes `null` for NaN and infinities. Also `format_int`, `format_bool`.

## 2. The document tree

```goose
var doc: json::Node[>..] = [];       // one node pushed per value, in text order
var decoded: u8[>..] = [];           // decoded text for strings that had escapes
let root, err = json::parse(text, doc, decoded);
guard root else { abort(err); }      // "json: expected ',' or '}' at byte 41"

let w, ok = json::as_i64(json::member(json::member(root, "window"), "width"));
let items = json::member(root, "items");
if items {
    json::each(items) { item =>
        let name, named = json::as_string(json::member(item, "name"));
        if named { print(name); }
    };
}
```

- Each node holds links to its next sibling and first child (relative
  `Node&<u32>?` links), its key, and its `Value` (`Null Bool Int Num Str Arr
  Obj`). Keys and strings are **slices of `text`**; escaped ones are slices
  of `decoded`. Nothing else is copied.
- A number without fraction/exponent fitting an `i64` becomes an `Int`;
  anything else a correctly-rounded `Num`.
- On malformed text, `parse` returns null and a message naming the byte;
  nodes before that point were already pushed.
- `member(n, key)` finds the first member named key (duplicates stay in the
  tree); accessors accept null, so lookups chain; `as_f64` accepts `Int` or
  `Num`.

## 3. The streaming Reader — JSON onto your types

```goose
struct Item { id: i64, price: f64 }

fn read_items(text: u8[:]) -> Item[>..] {
    var r = json::reader(text);
    var items: Item[>..] = [];
    json::each_element(r) {
        var it = Item { id: 0, price: 0.0 };
        json::each_member(r) { key =>
            if key == "id" { it.id = json::read_int(r); }
            else if key == "price" { it.price = json::read_number(r); }
            else { json::skip(r); }
        };
        items.push(it);
    };
    if !json::finish(r) { abort(json::message(r)); }
    return items;
}
```

- Parses as it goes and **builds nothing**; each block reads the value it
  expects and `skip`s the rest (also `peek(r)` — `'n'` means a `null` to
  handle an optional field).
- `read_string` returns a slice of the text (or of `scratch` for escaped
  strings) — pass a `u8[>..]&` scratch.
- Errors are **sticky**: the first unexpected text records what was
  expected and where, moves the cursor to the end, and every later read
  fails and every loop ends. Check once with `finish(r)` (also rejects
  trailing non-whitespace); `message(r)` formats the error.
- Keys arrive as written, including escapes — a key written with an escape
  never equals the same key written without one.

## Choosing a style

- Small configs / needs random access → tree.
- Large inputs, mapping onto your structs → Reader (serde-style).
- Emitting JSON → the writer functions.

## Related skills

- [csv-regex](csv-regex/skill.md)
- [error-idioms](../errors/error-idioms/skill.md)
- [parsing-text](../strings/parsing-text/skill.md)
