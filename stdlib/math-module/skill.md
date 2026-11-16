---
name: goose-math-module
title: math module — libm at both widths, constants, helpers
category: stdlib
tags: [goose, math, libm, sqrt, PI, mul_add, fma, smoothstep]
triggers: [goose math module, goose sqrt, PI constant, mul_add, fma, is_nan, smoothstep]
description: >
  math provides libm at both widths (sin picks sin/sinf by argument type),
  the constants PI/TAU/E/INF/NAN that adapt to context, helpers (radians,
  is_nan, is_finite, fract, inverse_lerp, smoothstep, wrap_angle), and
  mul_add with its FMA caveats (cancellation and overflow differences vs
  x*y+z).
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#math
---

# The `math` module

libm at both widths: `sin(x)` picks `sin` or `sinf` by the argument's type.
`sqrt sin cos tan asin acos atan atan2 exp log log2 log10 pow hypot cbrt
exp2 expm1 log1p sinh cosh tanh floor ceil round trunc copysign`. These
return floats as C does. An integer argument converts to both widths
equally, so the call must name one: `sqrt(n as f64)`; `sqrt(n * 0.5)` calls
the f64 version.

- `expm1(x)`/`log1p(x)` keep a small x's digits where `exp(x)-1`/`log(1+x)`
  lose them; `copysign(x, y)` distinguishes `-0.0` from `0.0`.
- `sqrt` is the runtime's: correctly rounded, NaN below zero, no errno (so
  the C compiler can keep loop values in registers). Other errno-setting
  libm functions are wrapped under runtime names that depend only on their
  arguments — constant arguments still fold, and `sin`+`cos` of one value
  combine into `sincos` where the C library has it.

## Constants and helpers

```goose
let PI = 3.141592653589793;   let TAU = 6.283185307179586;
let E = 2.718281828459045;    let INF = 1.0 / 0.0;   let NAN = 0.0 / 0.0;

radians(deg)  degrees(rad)              // both widths
is_nan(x) is_inf(x) is_finite(x)
fract(x)                                 // x - floor(x); never negative; NaN for inf
inverse_lerp(a, b, v)                    // (v - a) / (b - a); not clamped
smoothstep(edge0, edge1, x)              // 0 / 3t²-2t³ / 1
wrap_angle(x)                            // into [-PI, PI); PI wraps to -PI
```

`INF` and `NAN` take the type the context asks for, as `PI` does
(`let inf32: f32 = INF;`). `NAN`'s sign is unspecified.

## `mul_add(x, y, z)` — read before relying on it

`x * y + z` rounded **once** where the code is built for FMA and **twice**
where it is not. Which one a call gets is up to the implementation: fused on
arm64, on x86-64 built for AVX2+, and in a `simd fn`'s AVX2/AVX-512
versions; unfused under MSVC, under TinyCC, and in plain x86-64 baseline
builds.

The two roundings can differ substantially, not just a last bit:

- **Cancellation**: `mul_add(x, x, -(1.0 + 2.0 * (x - 1.0)))` at
  `x = 1 + 2^-30` is `2^-60` fused but `0.0` unfused — a branch on it goes
  either way.
- **Overflow**: `mul_add(1.0e308, 2.0, -1.0e308)` is `1.0e308` fused, `inf`
  unfused.

Inside a `simd fn`, only calls Goose emits in that function's own body fuse
(its code and what it inlines into it). A plain helper keeps its own
rounding; marking the helper `inline fn` keeps its calls fused in `simd`
callers at every level except `-O0`. `x * y + z` written out always rounds
twice.

Everything in `math` keeps no bounds check — callable from `checkfree` functions.

## Related skills

- [vec-mat-quat](vec-mat-quat/skill.md)
- [function-qualifiers](../functions/function-qualifiers/skill.md)
- [simd-and-inline](../performance/simd-and-inline/skill.md)
