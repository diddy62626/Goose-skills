---
name: goose-std-algorithms
title: std module — array algorithms, sorting, hashing, rng
category: stdlib
tags: [goose, std, map, filter, fold, sort, find, heap, hash, rng, shuffle]
triggers: [goose std module, goose sort, goose filter map, goose find, hash function, random numbers]
description: >
  The std module: HOF algorithms (map/filter/fold built in place), searching
  (find/find_index/position/binary_search/lower_bound), in-place transforms
  (sort pdqsort, stable_sort merge), length changes (insert_at/remove_at/
  swap_remove/retain/dedup/heap_*), hashing (hash/hash_combine, user key
  types), and splitmix64 rng.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#std
---

# The `std` module — arrays, algorithms, hashing, rng

`import std;` (its names are global). Conventions: in-place element access
takes `xs: T[:]` (every array kind coerces); length changes take `xs: A&`;
"maybe an element" is `T?`; positions are `i64`, `-1` for none; fresh arrays
come back as `T[>..]` built in the caller's destination.

## Transforming (result built in the receiving variable)

```goose
let squares = xs.map()    { it * it };            // U[>..], U = block's result
let evens   = xs.filter() { it % 2 == 0 };        // T[>..]
let total   = fold(xs, 0.0) { a, x => a + x };
let s = sum(xs);                                   // from default<T>()
let both  = concat(a, b);
```

## Reading and searching

```goose
let big = xs.find() { it > 100 };          // T? reference into xs, or null
let i = xs.find_index() { it > 100 };      // i64, -1 if none
let j = position(xs, v);                   // first i with xs[i] == v
let has = contains(xs, v);
let any = xs.any() { it < 0 };  let all = xs.all() { it > 0 };
let n = xs.count() { it > 0 };
let m = max(xs);                            // asserts xs.len > 0
let k = max_by(people) { it.age };          // index of the largest F(x)
let last = last(xs);                        // T&, asserts non-empty
let lb = lower_bound(xs) { it < 100 };      // first i with !F(xs[i])
let pos, found = binary_search(xs, v);      // sorted by < (or a block)
let at = find(line, "://");                 // generic: works on strings too
each_rev(xs) { x => ... };
each_chunk(pixels, width) { line, y => ... };
```

## In-place transforms

```goose
fill(xs, 0);
copy_into(dst, src);                        // equal lengths, first to last
reverse(xs);
sort(xs);                                   // pdqsort-family, unstable, in place
sort(people) { a, b => a.age < b.age };     // comparator: "a goes before b"
stable_sort(xs);                            // natural merge sort, one temp of xs.len
to_lower(s); to_upper(s);                   // ASCII, in place
shuffle(xs, r);
```

Both sorts make a single pass over already-sorted or strictly descending
input and do not allocate for 20 elements or fewer. A comparator that is not
a strict weak ordering leaves an unspecified order but is always a
permutation.

## Changing length

```goose
push_n(xs, v, n);
insert_at(xs, i, v);                        // shifts [i..) up
let removed = remove_at(xs, i);             // shifts down, returns the value
let swapped = swap_remove(xs, i);           // O(1), reorders
retain(xs) { it > 0 };
dedup(xs);                                  // drops adjacent duplicates
heap_push(q, 5); let smallest = q.heap_pop(); q.make_heap();
```

A `[>..<]` is the natural heap/queue container; a limited `[..k]` field
works too. Functions that move elements do so with `.=`, so they can
reorder arrays of references (including relative ones).

## Hashing

```goose
fn hash(x: i8) -> u64                       // one overload per integer type
fn hash(s: u8[:]) -> u64                    // any u8 array coerces; 8 bytes per multiply
fn hash_combine(seed: u64, h: u64) -> u64
```

`dictionary` picks up a user key type's `hash` overload (declared globally
or in the key type's namespace); combine field hashes:

```goose
struct key { a: i64, b: i64 }
fn hash(k: key) -> u64 { hash_combine(hash(k.a), hash(k.b)) }
```

Equality is structural and built in — no `PartialEq` to derive.

## Random numbers (splitmix64)

```goose
var r = rng { os.random_seed() };           // or a fixed seed
let d = r.rand_range(6);                    // uniform [0, 6)
let roll = r.rand_range(1, 6);              // a die: 1..6 (lo, size)
let x = r.rand_flt();                       // [0, 1), 53 bits
let g = r.rand_gaussian();                  // standard normal
let u = r.rand_u64();
```

`rand_range` takes a **size**, not an end. `rand_int(r, n)` = `rand_range(r, n)`.

## Scalars and bits

`min max clamp abs sign lerp rem div_euclid next_pow2 popcount clz ctz
rotl rotr byte_swap is_pow2 ilog2 isqrt gcd lcm swap` — all check-free
(callable from `checkfree fn`). `ilog2(x)` is `-1` for `x <= 0`;
`isqrt` is exact at every `u64`.

## Related skills

- [dictionary](dictionary/skill.md)
- [grow-shrink-stacks-queues](../arrays/grow-shrink-stacks-queues/skill.md)
- [function-values-blocks](../functions/function-values-blocks/skill.md)
