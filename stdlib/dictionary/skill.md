---
name: goose-dictionary
title: dictionary — the hash map (and set)
category: stdlib
tags: [goose, dictionary, hashmap, hash set, keys, update, get_or_insert]
triggers: [goose dictionary, goose map, hash table, dictionary get insert update, string keys]
description: >
  dictionary<K, V> is an open-addressing map with linear probing, power-of-
  two capacity, backward-shift deletion and 31-bit hash tags per slot. Keys
  need hash + ==; string keys are const u8[:] views (nothing copied). get/
  insert/get_or_insert/update/remove/each/reserve, and the shrink rule for
  references into the slot array.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#dictionary
---

# `dictionary`

```goose
import dictionary;

fn get<K, V>(d: dictionary<K, V>&, key: K) -> V?               // value reference, or null
fn contains<K, V>(d: dictionary<K, V>&, key: K) -> bool
fn insert<K, V>(d: dictionary<K, V>&, key: K, val: V) -> bool   // overwrites; true if new
fn get_or_insert<K, V>(d: dictionary<K, V>&, key: K, v0: V) -> V&
fn update<K, V, F>(d: dictionary<K, V>&, key: K, v0: V)         // F(val&); inserts v0 first if absent
fn remove<K, V>(d: dictionary<K, V>&, key: K) -> bool
fn clear<K, V>(d: dictionary<K, V>&)
fn reserve<K, V>(d: dictionary<K, V>&, n: i64)                  // room for n entries, no rehash
fn each<K, V, F>(d: dictionary<K, V>&)                          // F(key, val&); order unspecified
```

Open addressing, linear probing, power-of-two capacity, backward-shift
deletion. Keys and values are fixed-size; a key type needs `hash` and `==`.
Each slot keeps 31 bits of the key's hash as a tag, so keys are compared
only where tags agree, and growth/removal never re-hashes. Doubles in place
at 2/3 load; `reserve(n)` sizes for n entries at that load. `insert`,
`update`, `get_or_insert` look a key up once whether or not it is there.

## String keys are views — nothing copied

```goose
var counts = dictionary<const u8[:], i32> {};
each_split(text, ' ') { counts.update(it, 0) { it += 1; }; };
let n = counts.get("the");
if n { print(counts.count, " distinct words; 'the' x", n); }
```

Keys are `u8[:]` slices into text the caller keeps — `const u8[:]` for
literals and views of read-only data (that is why the key type is `const
u8[:]`), or inline `u8[..k]`. A **set** is `dictionary<K, bool>`.

## The one rule to respect: the slot array is grow-shrink

A reference into it (a `get`/`get_or_insert` result, an `each` binder) must
have its **last use before the next `insert`/`update`/`remove`/`clear`** —
otherwise the checker rejects the mutation, naming the reference and where
it was bound. The test is liveness, not scope: finish with the reference
first; no block needed around it. `update` does the lookup and the change
in one call, which avoids the issue entirely.

```goose
counts.each() { w, n => if n > 100 { print(w, " ", n); } };
```

## Reference values and interior pointers

A value can be an optional reference: `Cell?`, or a link into a pool
(`Cell&<u32 in cells>?`). Unused slots hold `default<V>()`. `get` and
`get_or_insert` return the stored reference itself; the blocks of `update`
and `each` receive it. Writing through it writes the pointee; **replacing**
it requires an `insert`. Such a reference points at its pointee rather than
into the slot array, so it may stay live across the next mutation:

```goose
var owner = dictionary<i64, Cell?> {};
owner.insert(7, cells[3]);
let c = owner.get(7);
owner.insert(7, cells[4]);       // cells[3] unchanged; c still refers to it
if c { c.v += 1; }
```

`get` returns null both for a null value and an absent key; `contains`
tells them apart.

## User key types

```goose
struct key { a: i64, b: i64 }
fn hash(k: key) -> u64 { hash_combine(hash(k.a), hash(k.b)) }
// == is structural and built in
var seen = dictionary<key, bool> {};
```

## Related skills

- [std-algorithms](../std-algorithms/skill.md)
- [const-writability](../../references/const-writability/skill.md)
- [lifetimes-roots](../../references/lifetimes-roots/skill.md)
