---
name: goose-performance-model
title: The performance model — where Goose's speed comes from
category: performance
tags: [goose, performance, memory model, cache, benchmarks, teardown]
triggers: [goose performance, why goose fast, benchmark numbers, cache traffic, memory savings]
description: >
  Two sources of speed: the memory model (no allocator on any path, one-
  store teardown, contiguous data, narrow links, variable-mode enums) and
  compiler optimization (BCE, inlining, simd fn). The benchmark headline
  numbers and what they do and do not claim, plus what loses (particles,
  calc).
source: https://github.com/aardappel/goose/blob/main/bench/summary.md
---

# The performance model

Goose's performance comes from **the memory model** and **compiler
optimization** — know which lever you are pulling when tuning.

## What the memory model buys

- **No allocator on any path.** Growth is a pointer bump; there is no
  malloc, no free list, no GC pause.
- **No teardown cost.** A scope exit is a pointer store, not a walk — even
  for millions of elements. Whole-process wall-clock includes teardown,
  which takes time in C++/Rust and none in Goose.
- **Data is contiguous.** An array of records is *one* block, not one block
  per record — the cache does less work. Nested strings/enums stay inline.
- **Links are narrow.** 2- and 4-byte offsets where a pointer would be 8;
  `u32 in nodes` index types are 4 bytes.
- **Variable-mode enums** (`Shape..`) size each value to its own variant —
  a tree of mostly-small nodes no longer reserves room for its largest
  variant in every node. The biggest single source of memory savings in the
  benchmarks (`records`: 2.02–2.26x faster than Rust on 3.98x less memory).

## What needs the optimizer

Every index is bounds-checked; the compiler proves most checks unnecessary
and removes them — but "most" is not "all", and any check it cannot prove
stays. Give the analysis lengths it can use (see bounds-check-elimination).
Inlining happens at `-O2` by default; `inline fn` forces it; `simd fn`
builds per instruction-set level.

## The numbers (October 16, 2026 run)

Geometric means over sixteen benchmarks at large sizes, whole-process
wall-clock including input generation, construction and teardown:

| vs | speed | peak memory |
|---|---|---|
| idiomatic C++ | 3.51x / 3.54x (MSVC / clang in Goose's favor) | 1.95x less |
| expert (hand-optimized) C++ | 1.24x / 1.26x | 1.32x less |
| best safe Rust | 1.13x / 1.19x | 1.25x less |

Wins concentrate where compact representation matters: `records`,
`interp`, `sexp`, `strlist`, `bintrees` (206/204 ms vs C++ arena 393/407),
`respond` (inline DTO 507/491 vs Rust 1338).

**Honest losses and near-ties**: `particles` (flat float kernel — no
design-predicted advantage, and none), `calc` (0.82x vs Rust — the one loss
attributed to Goose's own design), `lru`/`scene` (lose under MSVC, level
under clang), `words` (dictionary uses less memory than the hand-written
table but runs more slowly). Compare **matching interfaces** when deciding
which result applies to your application. Empty-process floor ~13 ms.

## Practical implications

- Prefer inline nested data, variable-mode enums for mixed-size trees,
  narrow links, index types for graphs.
- The allocation question disappears; the remaining tuning is data layout
  and bounds checks.
- Flat float kernels are not Goose's win — don't port them expecting gains.
- Measure with `goose build` (TinyCC does not optimize), whole-process,
  best of three after warm-ups, like the suite does.

## Related skills

- [bounds-check-elimination](bounds-check-elimination/skill.md)
- [inline-nested-data](../data-modeling/inline-nested-data/skill.md)
- [enums-adts](../types/enums-adts/skill.md)
- [benchmarking](benchmarking/skill.md)
