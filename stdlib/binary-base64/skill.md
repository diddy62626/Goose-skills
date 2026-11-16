---
name: goose-binary-base64
title: binary and base64 modules — external binary formats, checked reads
category: stdlib
tags: [goose, binary, base64, endian, reader, write_u32le, encode decode]
triggers: [goose binary format, read u32le, little endian, big endian, base64 encode decode]
description: >
  binary gives checked reads and appending writes of external formats,
  little- and big-endian, at offsets or with a cursor Reader and bounded
  subreaders; returned byte slices borrow the input. base64 is RFC 4648
  standard alphabet encode/decode into a builder.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#binary
---

# `binary` and `base64`

Both are namespaces (`import binary;`, `import base64;`). `binary` does
checked reads and appending writes of external formats, little- and
big-endian, written in Goose. It does **not** interpret Goose's own
serialization images (`from_bytes` does that).

## Reads at offsets

```goose
fn range_ok(data: const u8[:], at: i64, size: i64) -> bool
fn read_u8(data, at) -> u8, bool            // and read_i8
fn read_u16le(data, at) -> u16, bool        // and read_u16be, read_i16le/be
fn read_u32le(data, at) -> u32, bool        // and read_u32be, read_i32le/be
fn read_u64le(data, at) -> u64, bool        // and read_u64be, read_i64le/be
fn read_f32le(data, at) -> f32, bool        // and read_f32be, read_f64le/be
fn read_bytes(data, at, size) -> const u8[:], bool
fn read_name(data, at, width) -> const u8[:], bool   // fixed-width field up to first NUL
```

Negative values, truncated fields and offsets past the end fail with zero /
empty slice and `false`; range checking does not overflow on extreme
offsets or sizes. An empty range at the end is valid. Reads work at
unaligned offsets on either host byte order. A float read keeps the value's
bits exactly (NaN payloads included). Returned byte slices **borrow the
input** — lifetimes are checked normally.

## Cursor and bounded subreaders

```goose
struct Reader { data: const u8[:], pos: i64 }
var r = binary::reader(data);
let size, ok = binary::read_u32le(r);
var payload, pok = binary::subreader(r, size);   // bounded view with pos reset to 0
let version, vok = binary::read_u16le(payload);
let name, nok = binary::read_name(payload, 8);
```

Every read returns its own success flag. Success advances `pos`; failure
leaves `pos` unchanged (try a shorter read next). Set `pos` to seek; out-of-
range positions make reads fail. **No sticky error state.** `subreader`
consumes a bounded range from the parent — keep the parent in scope while
using the child or its slices. Offset reads for random access; cursors for
sequential records.

## Writes

```goose
var out: u8[>..] = [];
binary::write_u32be(out, 0x89504E47);    // a signature, big-endian
binary::write_u16le(out, 3);
binary::write_f32le(out, 1.5);
```

Each write appends the value's bytes in the order the same-named read takes
them (a write then that read gives the value back). Float writes keep bits
exactly. Writes cannot fail; `out` grows as an append does.

## base64 (RFC 4648 §4, standard alphabet)

```goose
var text: u8[>..] = [];
base64::encode(text, "foobar");              // "Zm9vYmFy"
var bytes: u8[>..] = [];
if !base64::decode(bytes, text) { abort("not base64"); }
```

`decode` accepts only whole 4-char groups (last may end `=`/`==`), rejects
line breaks, whitespace, unpadded text and URL-safe alphabets; on invalid
input returns `false` and leaves `out` unchanged. Both functions are
`simd fn`s — AVX2/AVX-512 versions where clang builds for x86-64.

## Where this fits

- Goose's own structures: `to_bytes`/`from_bytes` (see serialization).
- WAD-style external formats: sample `31_mini_doom.goose` shows both offset
  and cursor styles.
- Validate signatures, counts, indices and format-specific limits **after**
  checking the read — the module checks ranges, not semantics.

## Related skills

- [serialization](../data-modeling/serialization/skill.md)
- [json](json/skill.md)
- [simd-and-inline](../performance/simd-and-inline/skill.md)
