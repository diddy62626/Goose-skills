---
name: goose-vec-mat-quat
title: vec, mat, quat — the numeric geometry modules
category: stdlib
tags: [goose, vec2, vec3, mat4, quaternion, glsl, transform, graphics math]
triggers: [goose vec3, goose mat4, goose quaternion, float3, transform matrix, slerp]
description: >
  vec2/3/4<T> generic vectors (float2/3/4, double*, int* aliases) with
  language memberwise operators; mat as column-major GLSL matrices with
  transform builders (translate/rotate/perspective/ortho/look_at, right-
  handed, depth 0..1); quat rotations with angle_axis/slerp/from_euler and
  mat conversions; compose() builds model matrices.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#vec
---

# `vec`, `mat`, `quat` — geometry

Three modules over each other (`quat` imports `mat`). All generic over
`f32`/`f64`; the **argument types** pick the width, never the assignment
target. A literal adapts, but a typed `f64` argument widens the call —
write `radians(60.0 as f32)` for a `float4x4`. None of the functions keeps
a bounds check (`checkfree`-safe).

## vec — vectors

```goose
struct vec2<T> { x: T, y: T }    // vec3<T>, vec4<T>
type float2/float3/float4 = vec*<f32>;   double2/3/4; int2/3/4 = vec*<i32>
let float3_0 = float3 { 0.0, 0.0, 0.0 };   // _0 and _1 for all nine aliases

fn dot<T>(a, b) -> T          fn cross<T>(a, b) -> vec3<T>
fn length_sq<T>(v) -> T       fn length<T>(v) -> T      // float vectors
fn normalize<T>(v) -> vec3<T> fn distance<T>(a, b) -> T
fn lerp<T>(a, b, t) -> vec3<T>
fn xy<T>(v: vec3<T>) -> vec2<T>          // also xy(vec4), xyz(vec4)
```

Elementwise `+ - * /` are **built into the language** (memberwise
arithmetic), unary `-` on float/signed vectors, `*` and `/` broadcast a
scalar in either order (`v * 2.0`, `v / length(v)`, `1.0 / v`), and `v *=
scale` updates in place. The scalar converts to the component type (an
`f64` scaling an `float3` needs `as f32`). Per-component min/max is written
out (`min(a.x, b.x)`). There is no operator overloading; anything beyond
memberwise arithmetic is a named function (`dot`, `cross`).

## mat — matrices (GLSL conventions)

```goose
type float2x2 = float2[2];  float3x3; float4x4;   // and double*
let float4x4_identity

fn mul<T>(a: vec4<T>[4], b: vec4<T>[4]) -> vec4<T>[4]   // GLSL a * b: b first, then a
fn mul<T>(m, v) -> vec4<T>                              // GLSL m * v
fn mul<T>(v, m) -> vec4<T>                              // GLSL v * m
fn transpose / determinant / inverse / outer_product / row
fn to3x3(m4) / to4x4(m3)

// 3D builders (as 4x4s)
translate(t) scale(s) rotate(axis, angle) rotate_x/y/z(angle)
perspective(fovy, aspect, near, far) ortho(l, r, b, t, n, f) ortho(l, r, b, t)
look_at(eye, target, up)
transform_point(m, p)   // w = 1, divided by w
transform_vector(m, v)  // w = 0
```

- A matrix is a **fixed array of its columns**, as in GLSL: `m[i]` is
  column `i`, `m[3]` is a 4x4's translation, `m[i].y` is row 1 of column i.
- `a * b` of two matrices is **elementwise** (GLSL `matrixCompMult`); the
  product is `mul(a, b)`.
- `mul(a, b)` applies `b` first: a model matrix is
  `mul(translate(p), mul(rotate(axis, angle), scale(s)))` — glm's
  `translate(m, t)` is `mul(m, translate(t))`.
- Right-handed, into SDL_GPU's clip space: y up, depth 0 at `near` to 1 at
  `far`. The 4-arg `ortho` is near -1 / far 1 (2D scenes; z=0 at depth 0.5).
- A `mat4`'s bytes are what a std140/std430 block holds; a `mat3` pads
  columns to 16 bytes — send a 3x3 as `to4x4(m)`.

## quat — rotations

```goose
struct quat<T> { x: T, y: T, z: T, w: T }       // w the scalar part
type floatq = quat<f32>;  doubleq; let floatq_identity

mul(a, b)                 // Hamilton product (not *); b first, then a
rotate(q, v) -> vec3<T>
angle_axis(angle, axis)   // note: glm's argument order; normalizes axis
to_angle_axis(q) -> T, vec3<T>
rotation(from, to)        // shortest arc
quat_look_at(direction, up)
slerp(a, b, t)  nlerp(a, b, t)
from_euler(e)  euler_angles(q)      // x, then y, then z, radians
mat3_cast(q) mat4_cast(q) quat_cast(m)   // quat_cast of a rotation gives q or -q
conjugate(q) inverse(q) dot length normalize
compose(t, q, s) -> vec4<T>[4]      // the model matrix: scale, rotate, move
```

`a * b` on quats is elementwise (language rule); the Hamilton product is
`mul`. A rotation is a unit quaternion. `euler_angles` always rebuilds the
rotation when fed back to `from_euler` (glm's do not at yaw ±π/2).
`quat_look_at`/`look_at` are NaN where `up` is parallel to the direction.

## Related skills

- [math-module](../math-module/skill.md)
- [scalars-and-arithmetic](../../types/scalars-and-arithmetic/skill.md)
- [gfx module](../../interop/extern-lib-and-ffi/skill.md)
