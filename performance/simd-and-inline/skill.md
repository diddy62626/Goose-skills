---
name: goose-simd-and-inline
title: simd fn and inline fn — vectorization knobs
category: performance
tags: [goose, simd fn, inline fn, avx2, avx512, vectorize, mul_add]
triggers: [goose simd, simd function, avx2, inline fn, vectorize loop, GS_SIMD]
description: >
  simd fn builds once per instruction-set level (baseline + AVX2 + AVX-512
  under clang x86-64), picking the widest version at each call, with
  identical output at every level. inline fn forces inlining at every call.
  GS_SIMD levels, the TinyCC/MSVC caveats, and mul_add's FMA behavior.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#function-qualifiers
---

# `simd fn` and `inline fn`

## `simd fn`

```goose
simd fn checksum(xs: u8[:]) -> u32 {
    var h: u32 = 0;
    for x in xs { h += x as u32 * 31 + 7; }
    return h;
}
```

Built once for the target's baseline instruction set and once more for each
wider one it supports (**AVX2 and AVX-512 on x86-64, under clang**). At each
call, the widest version the CPU has is chosen. Every version computes
exactly what the baseline does — nothing about a program's output depends
on it (the one exception: `math`'s `mul_add`, which rounds once in the
fused versions).

Suits a function holding a hot loop the C compiler can vectorize: byte
transforms, codecs, kernels over arrays. The stdlib's `base64` encode/decode
are `simd fn`s and run several times faster under AVX2/AVX-512 than SSE2.

Caveats:

- The C backend ignores it under compilers other than clang, and under the
  in-process TinyCC backend (a JIT run gets baseline code).
- `GS_SIMD` (default 2) caps the highest level built: `1` = no AVX-512,
  `0` = baseline only.
- Which version runs is a property of the **CPU**, so timings across
  machines vary — that is by design, not noise.

## `inline fn`

```goose
inline fn clamp01(x: f32) -> f32 { if x < 0 { 0 } else if x > 1 { 1 } else { x } }
```

Inlined at every call whatever its size. Without it, the compiler decides
from the function's size and call count. Changes nothing about what the
program computes. Cannot combine with `extern`, `export`, `simd`,
`recursive` or `thread_fn`.

Use it when: a small helper sits on a hot path and you can see (via
`goose -o out.c`) that it was not inlined; or to keep `mul_add` calls fused
inside a `simd fn` caller — a plain helper's rounding follows the helper's
own build, an `inline fn` helper's calls fuse in `simd` callers at every
level except `-O0` (where Goose inlines nothing).

## Related skills

- [function-qualifiers](../../functions/function-qualifiers/skill.md)
- [math-module](../../stdlib/math-module/skill.md)
- [bounds-check-elimination](../bounds-check-elimination/skill.md)
