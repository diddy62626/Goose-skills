---
name: goose-benchmarking
title: Benchmarking recipes — measuring Goose the way the suite does
category: performance
tags: [goose, benchmark, timing, methodology, whole-process, clock]
triggers: [goose benchmark, time goose code, benchmark methodology, goose speed measurement]
description: >
  How to benchmark Goose credibly: build (never JIT) for timing, whole-
  process wall-clock including teardown, best of three after warm-ups,
  os::clock() for in-process timing, --stats/--bce-lines for diagnosis, and
  the methodology rules the official suite follows (matching interfaces,
  input generation included).
source: https://github.com/aardappel/goose/blob/main/bench/design.md
---

# Benchmarking recipes

## Never time a JIT run

TinyCC does not optimize. **Always benchmark `goose build` executables.**
The suite builds the standalone form (`goose -o prog.c` +
runtime, or `goose build`).

## Whole-process wall clock

The official numbers are **whole-process wall-clock time including input
generation, construction and teardown**. Goose teardown is one pointer
store, while C++/Rust destructors take real time — excluding teardown
understates Goose's advantage:

```bash
goose build --cc clang bench.goose
for i in 1 2 3; do time ./bench; done     # best of three after two warm-ups
```

The suite does not pin cores and uses the best of three runs after two
warm-ups; empty-process floor is ~13 ms (keep that in mind for sub-100 ms
benchmarks).

## In-process timing

```goose
import os;
let t0 = os.clock();
// ... the work ...
print("elapsed: ", os.clock() - t0, " s");
```

`os::clock()` is monotonic and high-resolution. Prefer it for micro-
benchmarks inside a program; keep the whole-process number as the headline.

## Diagnosing a slow kernel

1. `goose check --stats` — how many bounds checks were elided.
2. `goose --bce-lines` — which lines kept checks; add `assert`s of lengths.
3. `goose --specs` — did the loop inline? Did the generic specialize as
   expected?
4. `goose -o out.c` — read the generated C; is the loop the shape you wrote?
5. Try `-O1` vs `-O2` (inlining changes), `--cc clang` vs `gcc` vs `msvc`.
6. `simd fn` on a hot loop (clang on x86-64 builds AVX2/AVX-512 versions).

## Methodology rules from the official suite

- **Matching interfaces**: compare Goose inline-DTO vs Rust inline-DTO, not
  vs streaming; state which rows you compare.
- Include input generation and teardown in the measurement.
- Report peak memory alongside time (whole-process; Goose often wins memory
  more decisively than time).
- Treat small differences cautiously, especially in cache-sensitive graph
  and LRU workloads; cross-date timing changes cannot be attributed to
  compiler improvements alone.
- Flat float kernels (particles) are a known non-win — pick benchmarks that
  exercise compact data structures if you want to see Goose's design wins.

## What the suite measures

Sixteen benchmarks (`bench/`), three sizes, against idiomatic C++, expert
C++, and best safe Rust; per-backend (MSVC/clang) tables in
`bench/results.md`, with design notes and historical diagnostics kept
explicitly marked in `bench/notes.md`.

## Related skills

- [performance-model](../performance-model/skill.md)
- [bounds-check-elimination](../bounds-check-elimination/skill.md)
- [simd-and-inline](../simd-and-inline/skill.md)
