---
name: goose-extern-fn
title: extern fn — calling C from Goose
category: interop
tags: [goose, extern fn, c interop, ffi, bindings, cbrt, math.h]
triggers: [goose extern fn, call c, c function binding, extern declaration, ffi]
description: >
  extern fn binds a Goose signature to a C symbol with no body; the call
  compiles to a direct C call. What may cross: scalars, flat fixed-size
  values, T& pointers, T[:] as {data,len}, and u8[>..]& builders. What is
  rejected at the declaration. Where symbols resolve (JIT: math.h only;
  builds: your C and libraries).
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#extern-fn
---

# `extern fn` — calling C

A declaration without a body binds a Goose signature to a C function. A
string after `=` is the C symbol; without one, the Goose name is used. The
declaration is top-level, fully typed, not generic, and joins overload
resolution like any function. A call compiles to a **direct C call** with no
calling-convention extras.

```goose
extern fn cube_root(x: f64) -> f64 = "cbrt";    // libm, by its C name
extern fn hypot(x: f64, y: f64) -> f64;

extern fn crc32_bytes(s: const u8[:]) -> u32;   // a slice in, a scalar out
extern fn stats_of(xs: i32[:], out: Stats&);    // a struct filled through a pointer
extern fn c_version(out: u8[>..]&);             // a builder C appends to
```

## What may cross

Exactly what has a plain C shape:

- the integer and float scalars and `bool`;
- any **flat fixed-size** value (structs, fixed/static-capacity arrays,
  fixed-mode enums), by value as its packed C type;
- `T&` to such a `T`, as a pointer;
- `T[:]` of such a `T`, as a `{ data, len }` struct;
- `T[>..]&` of such a `T` — a builder C may write into and append whole
  elements to (`gs_bld_append` appends bytes to a `u8[>..]&`).

Results are one scalar or flat fixed-size value, or nothing. Everything else
(`varint`s, references inside structs, variable/resizable values by value,
optionals) is **rejected at the declaration** — never mis-marshalled at the
call. A parameter is as writable as its type says; read-only arguments need
`const`. C must not keep a borrowed argument beyond the call.

## Where symbols resolve

- `goose run` (in-process TinyCC): only the C99 `<math.h>` functions in both
  widths (all but `nexttoward`) — anything else is not found at link time.
- `goose build` (or the flag form `goose -o`): the program's own C (sidecar
  `foo.build`, `c_sources = [...]`) and any library named by `c_libs`.
- An `extern "lib" { ... }` block binds a shared library by name at program
  start — no C to compile or link (see extern-lib-and-ffi).

Your own C can also be spliced in with `--include header.h` (small shims
written against the generated types).

## Common patterns

```goose
struct Stats { lo: i32, hi: i32, mean: f64 }
extern fn stats_of(xs: i32[:], out: Stats&);        // out-parameter via pointer
extern fn crc32_bytes(s: const u8[:]) -> u32;       // slices as {data, len}
```

For a C function that appends to a Goose builder, declare the last parameter
`out: u8[>..]&` and implement it with `gs_bld_append` from `goose.h` (the
compiler puts that header on the include path of every native part).

## Design rule

Keep the boundary thin: declare only what you call, marshal flat data, and
push anything pointer-heavy into a few lines of C adapter (callbacks and
variadic functions cannot be bound at all — they need a C shim).

## Related skills

- [extern-lib-and-ffi](../extern-lib-and-ffi/skill.md)
- [sidecar-own-c](../sidecar-own-c/skill.md)
- [export-to-c](../export-to-c/skill.md)
