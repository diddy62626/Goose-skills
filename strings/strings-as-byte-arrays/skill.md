---
name: goose-strings-as-byte-arrays
title: Strings are byte arrays — four roles, no string type
category: strings
tags: [goose, strings, u8, builder, utf8, raw strings]
triggers: [goose string, u8 array string, goose text, raw string, triple quote, utf-8]
description: >
  There is no string type: a string is any array-family type with u8
  elements. u8[>..] builder, u8[] finished, u8[..16] inline, u8[:] view.
  String literals are const u8[:] static data. Raw """ strings, UTF-8 as a
  library convention, and the inline-string-in-struct pattern.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#6-strings-and-text
---

# Strings are byte arrays

There is no string type. A string is **any array-family type with `u8`
elements**, and which one you pick says what you want:

| Type | Role |
|---|---|
| `u8[>..]` | a **builder**: appending is a pointer bump |
| `u8[]` / `u8[varint]` | a **finished string**, stored inline in whatever holds it |
| `u8[..16]` | a **small string** inside a struct, no indirection |
| `u8[:]` | a **view** of bytes that live elsewhere |

A string literal is a `const u8[:]` — read-only static data. Copying it into
any `u8` array (`let t: u8[] = "fixed";`) gives a writable value of its own.

## Building text

```goose
var s: u8[>..] = [];
s.append("hello");
s.push(',');                       // one byte
format(s, " world! ", 6 * 7);      // appends the text of anything
print(s, " [", s.len, " bytes]");  // hello, world! 42 [16 bytes]
```

`u8[..16]` and the other limited arrays are builders too, up to capacity:
`format(p.name, "linus");`.

## Raw strings `"""…"""`

No escapes; the closing delimiter's indentation is removed from every line,
so the text can be indented to match the code:

```goose
print("""
    usage: wc [-l] file
      -l   count lines only
    """);
print("""C:\temp\new "quoted" \n""");   // backslashes literal, on one line too
```

## Strings inline in structs

```goose
struct Person { name: u8[..16], age: i32 }

// A format overload makes print/str/format render Person your way.
fn format(out: u8[>..]&, p: Person) { format(out, p.name, " (", p.age, ")"); }

var people: Person[>..] = [];
people.push(Person { name: "ada", age: 36 });
print(people);                          // [ada (36), grace (45), linus (20)]
```

An array of such structs is one block you can sort and copy as plain data.
For variable-size text in records use `u8[]`/`u8[varint]` (the record then
has sequential elements).

## A value that can be one of several strings

When one branch is a slice, a branch that gives an inline string field gives
a slice of that whole field, nothing copied, and the `if` is a
`const u8[:]`:

```goose
let who = if people.len > 3 { "a crowd" } else { people[1].name };
```

## Encoding

No encoding is enforced; UTF-8 is a library convention. `each_utf8` decodes
(WHATWG rules, malformed bytes → 0xFFFD) and `push_utf8` writes code points.
Strings are not checked to be valid UTF-8.

## Character literals

`'a'` is an *integer* constant (the byte's value): `str('x')` is `"120"`.
Append single bytes with `push`; compare bytes with integer operators.

## Related skills

- [formatting-text](../formatting-text/skill.md)
- [parsing-text](../parsing-text/skill.md)
- [slices](../../arrays/slices/skill.md)
