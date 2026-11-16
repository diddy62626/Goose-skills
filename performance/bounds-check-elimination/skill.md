---
name: goose-bounds-check-elimination
title: Bounds-check elimination — help the compiler prove checks away
category: performance
tags: [goose, bounds check, bce, assert, checkfree, bce-lines, kernel]
triggers: [goose bounds check, bce, checkfree, --bce-lines, index check elided, kernel optimization]
description: >
  Every index is checked; the compiler removes checks it can prove
  unnecessary. What makes proofs possible (grow-only shrink points,
  Euclidean %, fixed sizes, assert of lengths at kernel top, index types),
  how to measure (--bce-lines, --stats), and checkfree fn as the contract.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#bounds-checks
---

# Bounds-check elimination (BCE)

Every index into an array or slice is checked, as is every conversion into
an index type. A failing check aborts the program with the line of the
check. The compiler removes the checks it can prove unnecessary — and the
language is designed to give that analysis what it needs.

## Why proofs are possible here

- A `T[>..]` shrinks only at a `pop`/`pop_n`/`resize`/`clear` the analysis
  sees, so **a bound established before a `push` still holds after it**.
- `%` is Euclidean, so a reduction `x % n` is always in range for length-n.
- Fixed sizes and capacities are constants.
- Every call site records what it proves about its arguments.

## The idiom that helps most

An `assert` of the lengths at the top of a kernel:

```goose
let W = 512;
fn blur_row(above: u8[:], here: u8[:], below: u8[:], out: u8[:]) {
    assert(above.len == W && here.len == W && below.len == W && out.len == W);
    for x in 1..W - 1 {
        let sum = above[x - 1] as i64 + above[x] + above[x + 1] +
                  here[x - 1]  + here[x]  + here[x + 1] +
                  below[x - 1] + below[x] + below[x + 1];
        out[x] = (sum / 9) as! u8;
    }
}
```

With one `assert` per row, every index inside the pixel loop is a loop
counter plus a constant, checked against a known length — all checks are
removed and the C backend gets the loop it would get from C. Without the
assertion, none are removed and the loop does not vectorize. `goose check
--stats` reports the tally:

```
bce: elided 11/11 index and 0/4 slice checks
```

## Measuring

- `goose check --stats` — the removed/total tally.
- `goose --bce-lines` — per line, checks removed vs kept; use when a kernel
  is slower than expected.
- `--no-bce` — keep every check (for measuring their cost; warns that
  `checkfree` functions go unverified).

## `checkfree fn` — make it a contract

A surviving check inside a `checkfree fn` is a **compile error** — in every
specialization and inlined copy, and through what it calls:

```goose
var nodes: i32[>..] = [];
type NodeId = u32 in nodes;

checkfree fn sum_at(ids: NodeId[:], weights: i32[>..]&) -> i64 {
    assert(weights.len >= nodes.len);          // once; the loop is then check-free
    var t = 0;
    for id in ids { t += nodes[id] + weights[id]; }
    return t;
}
```

What the compiler must prove includes: constant indices into fixed arrays;
an index of `a`'s index type into `a`; into `b` where `b.len >= a.len` was
asserted and neither array changed since; a `for` index into the array
whose length is its count; `a[i]` after a condition established
`0 <= i < a.len`; and repeats of passed checks. For indirect indexing
(`dist[adj[k]]`), index types (`u32 in nodes`) move the check to
construction — the check-free graph traversal in the benchmarks works this
way.

## Other aborts that stay

Division by zero, a limited array's capacity overflow, shrinking below
empty, a relative-reference offset not fitting its width, a failed
`assert`, and data-stack exhaustion abort in every build — all clean, with
the failing line.

## Related skills

- [index-types](../data-modeling/index-types/skill.md)
- [function-qualifiers](../functions/function-qualifiers/skill.md)
- [performance-model](performance-model/skill.md)
