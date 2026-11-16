---
name: goose-serialization
title: Serialization — to_bytes / bytes_of / from_bytes, verified loading
category: data-modeling
tags: [goose, serialization, to_bytes, from_bytes, bytes_of, save, load, verification]
triggers: [goose serialization, save load data, to_bytes, from_bytes, bytes_of, position independent]
description: >
  Structures whose links are self-relative offsets are position-independent:
  saving is a write of bytes, loading a read — no serializer, no schema, no
  pointer fixup. from_bytes verifies framing, tags, lengths, canonical
  varints and every link before the bytes become a value; a hostile file
  gives false and an empty array, never a wild reference.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#23-serialization
---

# Serialization

A structure whose links are **self-relative** offsets means the same thing
wherever it sits in memory, so saving it is a write of its bytes and loading
it is a read. No serializer, no schema, no pointer fixup, no allocation per
node.

## The three builtins

| Builtin | Meaning |
|---|---|
| `to_bytes(a) -> u8[>..]` / `to_bytes(a, out)` | a **framed** image: varint byte count, then the element region; fresh or appended |
| `bytes_of(a) -> u8[:]` | the element region alone, as a read-only view — nothing copied |
| `from_bytes<T[>..]>(b: u8[:]) -> T[>..], bool` | a **verified** array from an untrusted image; `false` and empty if not one |

```goose
struct Node { word: u8[..24], count: i32, left: Node&<u32>?, right: Node&<u32>? }

var index: Node[>..] = [];
// ... build the tree ...

var image: u8[>..] = index.to_bytes();      // framed image of the array
write_file("index.bin", image);

var loaded, ok = from_bytes<Node[>..]>(image);
print(ok, ", ", loaded.len, " nodes");
```

## Verification — the interesting half

Bytes from outside the program are the only place a reference could enter
without the compiler having proved it, so `from_bytes` **verifies before the
bytes become a value**: the framing, every tag, every length, that varints
are canonical, and that every link lands on the start of an element in that
same image. A corrupt or hostile file gives `false` and an empty array,
never a wild reference.

```goose
image[image.len - 3] = 200;                  // tamper with a link
var bad, bok = from_bytes<Node[>..]>(image);
print(bok, " ", bad.len);                    // false 0
```

What it verifies is **safety, not integrity**: edited data bytes still
describe a well-formed structure, so add a checksum yourself when the file's
identity matters. What comes back is an ordinary array — index it, grow it;
references into it are ordinary references.

## Scope and limits

- `from_bytes` also builds `T[>..<]` and the `T[]` family — except a
  `T[>..<]` whose elements hold relative references (a shrink of the result
  would free what its own links name).
- Links into a *field* of an element, and `in pool` links, are not
  serializable. Design for saving → prefer self-relative links.
- A `bytes_of` view is never writable.
- Arrays of index types are checked against their array.
- For external formats, use the `binary`/`json`/`csv`/`base64` modules
  instead; `format_uleb`/`parse_uleb` write/read the length prefix of the
  framing.

## Streaming pattern with no copy

```goose
let payload = pool.bytes_of();            // a view, nothing copied
var head: u8[>..] = [];
format_uleb(head, payload.len);
write_file(path, head); append_file(path, payload);
```

## Related skills

- [relative-references](../../references/relative-references/skill.md)
- [slot-slice-pools](../slot-slice-pools/skill.md)
- [binary-base64](../../stdlib/binary-base64/skill.md)
