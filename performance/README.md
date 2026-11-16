# performance

**Tooling** — The performance model, bounds-check elimination, benchmarking recipes, simd/inline.

4 skills. Load any of them as `performance/<skill>/skill.md`.

- [`goose-benchmarking`](performance/benchmarking/skill.md) — Benchmarking recipes — measuring Goose the way the suite does: > How to benchmark Goose credibly: build (never JIT) for timing, whole- process wall-clock including teardown, best of three after warm-ups, os::clock() for in-process timing, --stats/--bce-lines for diagnosis, and the methodology rules the official suite follows (matching interfaces, input generation included).
- [`goose-bounds-check-elimination`](performance/bounds-check-elimination/skill.md) — Bounds-check elimination — help the compiler prove checks away: > Every index is checked; the compiler removes checks it can prove unnecessary. What makes proofs possible (grow-only shrink points, Euclidean %, fixed sizes, assert of lengths at kernel top, index types), how to measure (--bce-lines, --stats), and checkfree fn as the contract.
- [`goose-performance-model`](performance/performance-model/skill.md) — The performance model — where Goose's speed comes from: > Two sources of speed: the memory model (no allocator on any path, one- store teardown, contiguous data, narrow links, variable-mode enums) and compiler optimization (BCE, inlining, simd fn). The benchmark headline numbers and what they do and do not claim, plus what loses (particles, calc).
- [`goose-simd-and-inline`](performance/simd-and-inline/skill.md) — simd fn and inline fn — vectorization knobs: > simd fn builds once per instruction-set level (baseline + AVX2 + AVX-512 under clang x86-64), picking the widest version at each call, with identical output at every level. inline fn forces inlining at every call. GS_SIMD levels, the TinyCC/MSVC caveats, and mul_add's FMA behavior.
