---
name: goose-parsing-text
title: Parsing and manipulating text — split/trim/find/parse_int, zero-copy
category: strings
tags: [goose, split, trim, find, parse_int, parse_flt, compare, utf8, zero-copy]
triggers: [goose split, goose parse_int, goose trim, goose find, string search, goose words]
description: >
  std's text functions operate on u8[:] views: nothing is copied. split
  returns slices into the input, split_once is strings.Cut, words splits at
  whitespace, parse_int/parse_uint/parse_flt return (value, ok), compare is
  bytewise, each_utf8/push_utf8 handle code points, ascii:: character
  classes.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#strings
---

# Parsing and manipulating text (std module)

All functions below are in `std` (import once: `import std;`). They operate
on `u8[:]` views of bytes that already exist; **nothing is copied**.

## Slicing text

```goose
let line = "  key = some value  ";
let body = trim(line);                    // ASCII whitespace, a sub-slice
let eq   = find(body, "=");               // position or -1; generic: finds subsequences
print("'", trim(body[..eq]), "' -> '", trim(body[eq + 1..]), "'");
```

`find`/`rfind`/`starts_with`/`ends_with` are generic over the element, so
they are the string functions too: `find(line, "://")`.

## Splitting

```goose
let parts = split("alice,30,paris", ',');   // array of slices INTO the input
print(parts.len, " parts: ", parts);        // 3 parts: ["alice", "30", "paris"]
let cmd = words("  go   north ");           // runs of whitespace, no empties
let key, value, ok = split_once(arg, '=');  // Go's strings.Cut; not found: s, "", false
each_split(text, '\n') { handle(trim(it)); };   // no array built at all
each_word(text) { w => ... };
```

- `split` keeps empty parts (single-char or string separator, matched from
  the left; an empty separator gives `[s]`).
- `words` drops empty parts — right for command lines.
- Every part is a `const u8[:]` view: a few bytes each, no new strings.

## Joining and replacing

```goose
join(out, parts, ", ");                     // parts: T[:] of any u8 array/slice type
format_replaced(out, s, old, with);         // replace all occurrences
```

## Parsing numbers

```goose
let n, ok = parse_int(trim(field));         // optional sign, whole string; base 2..36
let u, uok = parse_uint(s);                 // optional '+', no '-'
let f, fok = parse_flt(s);                  // correctly rounded, like strtod; inf/-inf/nan
```

A `false` result means "not a number": pair with `guard ok else { ... }`.

## Formatting integers to text (see formatting-text)

`format_int` / `format_uint` / `format_flt` write into a builder with base,
width, padding, decimals.

## Comparing

```goose
let c = compare(a, b);        // bytewise: -1, 0, 1
if a == b { }                 // == compares structurally (length, then bytes)
```

## UTF-8

```goose
each_utf8(s) { cp => ... };              // malformed bytes give 0xFFFD
push_utf8(out, cp);                      // writes the code point's UTF-8
```

Decoding follows the WHATWG decoder: each maximal broken subpart gives one
0xFFFD; the byte that broke a sequence starts the next one.

## ASCII character classes (namespace `ascii`)

```goose
ascii::is_digit(c); ascii::is_alpha(c); ascii::is_alnum(c);
ascii::hex_value(c);                        // 0-15 for 0-9a-fA-F, -1 otherwise
```

ASCII only — no byte from 0x80 up is in any class (like C's ctype in the C
locale).

## LEB128

```goose
format_uleb(out, v);                 // 1-10 bytes; used by serialized images
let v, n = parse_uleb(s);            // value and byte count; 0, 0 if malformed
```

Useful to write a length prefix before a `bytes_of` view, so a stream reader
knows the payload size.

## Related skills

- [strings-as-byte-arrays](strings-as-byte-arrays/skill.md)
- [slices](../arrays/slices/skill.md)
- [std-algorithms](../stdlib/std-algorithms/skill.md)
