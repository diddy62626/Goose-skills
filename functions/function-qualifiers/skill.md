---
name: goose-function-qualifiers
title: Function qualifiers — inline, simd, checkfree, thread_fn, extern, export
category: functions
tags: [goose, inline fn, simd fn, checkfree, thread_fn, extern fn, export fn, qualifiers]
triggers: [goose inline fn, simd function, checkfree fn, thread_fn, export fn, extern fn order]
description: >
  The qualifier vocabulary and their fixed order: export/extern first, then
  checkfree, simd, inline, recursive, and fn/thread_fn. What inline forces,
  what simd builds (per instruction-set level), what checkfree guarantees,
  and which combinations are forbidden.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#function-qualifiers
---

# Function qualifiers

| Qualifier | Meaning |
|---|---|
| `recursive fn` | the entry of a recursive cycle |
| `extern fn`, `extern fn … = "sym"` | a C function |
| `export fn`, `export "sym" fn` | a Goose function callable from C |
| `thread_fn` | a worker's entry point |
| `checkfree fn` | no bounds check may remain in it |
| `inline fn` | inlined at every call, whatever its size |
| `simd fn` | built once per instruction-set level |

The qualifiers come in a fixed order: `export` or `extern` first, then
`checkfree`, `simd`, `inline`, `recursive`, and `fn` or `thread_fn`.

## `inline fn`

Asks for inlining at every call; without it the compiler decides from the
function's size and its number of calls. Changes nothing about what the
program computes. Cannot combine with `extern`, `export`, `simd`,
`recursive` or `thread_fn`. Useful to keep `mul_add` calls fused inside a
`simd fn` caller, and for small helpers on hot paths.

## `simd fn`

Built once for the target's baseline instruction set and once more for each
wider one it supports (AVX2 and AVX-512 on x86-64 under clang). At each
call, the widest version the CPU has is chosen. Suits a function holding a
hot loop the C compiler can vectorize: byte transforms, codecs, kernels over
arrays. Every version computes exactly what the baseline does — nothing
about a program's output depends on it. A backend may ignore it (the C
backend does under compilers other than clang, and under TinyCC). The
exception is `math`'s `mul_add`: fused rounding in the wider versions.

```goose
simd fn checksum(xs: u8[:]) -> u32 {
    var h: u32 = 0;
    for x in xs { h += x as u32 * 31 + 7; }
    return h;
}
```

## `checkfree fn`

Asks the compiler to **reject the program** if a bounds check is left
anywhere in the function's text after bounds-check elimination — in every
specialization and every inlined copy. What it calls (and blocks it calls)
is held to the same rule where it runs inside it. `assert` states
preconditions once, outside loops:

```goose
checkfree fn sum_at(ids: NodeId[:], weights: i32[>..]&) -> i64 {
    assert(weights.len >= nodes.len);      // once; the loop is then check-free
    var t = 0;
    for id in ids { t += nodes[id] + weights[id]; }
    return t;
}
```

The compiler must prove at least: a constant index into a fixed array; an
index of `a`'s index type into `a`; into `b` where `b.len >= a.len` was
asserted; a `for` index into the array whose length is its count; `a[i]`
after a condition established `0 <= i < a.len`; and repeats of passed
checks. The error names the check that remained and why. A helper that
indexes nothing needs no annotation; one that keeps a check is an error
naming the calls that lead to it.

## `thread_fn`

A worker's entry point; compiled as a separate program with its own data
stacks and globals (see concurrency skills). Parameters have no defaults.

## `extern fn` / `export fn`

The C boundary. `extern fn` binds a Goose signature to a C symbol
(`extern fn cube_root(x: f64) -> f64 = "cbrt";`); `export fn` makes a Goose
function callable from C (`export "c_add" fn add(a: i32, b: i32) -> i32`).
See the interop skills.

## Related skills

- [simd-and-inline](../../performance/simd-and-inline/skill.md)
- [bounds-check-elimination](../../performance/bounds-check-elimination/skill.md)
- [index-types](../../data-modeling/index-types/skill.md)
- [extern-fn](../../interop/extern-fn/skill.md)
