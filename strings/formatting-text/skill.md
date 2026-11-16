---
name: goose-formatting-text
title: Formatting text — print/format/str and the format overload
category: strings
tags: [goose, print, format, str, format overload, rendering, format_int, format_flt]
triggers: [goose print, goose format, goose str, custom print, format overload, number formatting]
description: >
  The three rendering builtins (print, format, str) take any number of
  arguments of any type with no format string. A fn format(out: u8[>..]&,
  v: T) overload defines a type's text form once. Controlled output via
  std's format_int/format_uint/format_flt, plus float printing rules.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#rendering-text
---

# Formatting text

## The three builtins

Each takes any number of arguments of any type, inserting nothing between
them:

- `print(a, b, …)` — writes the text and a newline to stdout as a **single
  write** (thread output never interleaves).
- `format(out, a, b, …)` — appends to `out`, any growable `u8` array.
- `str(a, b, …)` — a fresh `u8[>..]` of the text, **built directly at its
  destination** (`words.push(str("item", i))` writes into the new element).

```goose
print("answer=", 42, " ratio=", 0.75, " yes=", 3 > 2);
```

## How values render

- Integers decimal; floats in the shortest form that reads back to the same
  value (`1.0`, `0.1`, `inf`, `-inf`, `nan`); `true`/`false`.
- A `u8` array prints as its bytes; **nested inside an aggregate it is
  quoted and escaped**.
- Arrays as `[1, 2, 3]`; structs/variants as positional literals with the
  type name (`Point { 1, 2 }`); a reference as what it points at; null as
  `null`.
- A character literal is an integer: `str('x')` is `"120"`.

## The `format` overload: a type's text form in one place

```goose
fn format(out: u8[>..]&, v: T) { ... }
```

Called wherever a `T` occurs in an argument of any of the three builtins —
including inside arrays and other structs. Looked up in the namespace `T` is
declared in, then globally, so the type decides how it is printed. A generic
type renders through a generic overload (`fn format<T>(out: u8[>..]&,
b: Box<T>)`) unless a non-generic one for the instance exists. A non-fixed
`T` is never copied implicitly, so the overload takes it by reference.

```goose
struct Person { name: u8[..16], age: i32 }
fn format(out: u8[>..]&, p: Person) { format(out, p.name, " (", p.age, ")"); }
print(people);    // [ada (36), grace (45)]
```

## Controlled number/text output (std)

```goose
import std;
format_int(out, v, 16, 8, '0');     // base 2..36, right-aligned, padded; aborts outside
format_uint(out, v, 10, 0, ' ');
format_flt(out, v, 2);              // fixed decimals, exact value rounded half up
```

- In base 10, `format_int`/`format_uint` write the same text as `format`
  (after padding) and run as fast.
- `format_flt` rounds the exact binary value: `format_flt(out, 0.1, 20)`
  gives `0.10000000000000000555`; `format_flt(out, 2.675, 2)` gives `2.67`
  (the double nearest 2.675 is just below it). Only exact halfway values
  round differently from `printf` (half up vs half to even): `0.125` →
  `0.13`.
- Plain `format` writes the shortest text that reads back as the same value.

## Hashing-compatible rendering tip

`hash(s: u8[:])` hashes any `u8` array, and floats hash as their bits — but
for text output of your own types, a `format` overload plus `str(...)` is
the whole story; there are no format strings or `<<` anywhere in the
language.

## Related skills

- [strings-as-byte-arrays](strings-as-byte-arrays/skill.md)
- [parsing-text](parsing-text/skill.md)
- [structs](../types/structs/skill.md)
