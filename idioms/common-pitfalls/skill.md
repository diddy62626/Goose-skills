---
name: goose-common-pitfalls
title: Common pitfalls — the gotchas that bite in the first afternoon
category: idioms
tags: [goose, pitfalls, gotchas, mistakes, beginner errors]
triggers: [goose gotchas, goose pitfalls, common mistakes, why does not compile, goose errors beginner]
description: >
  Every pitfall in one list: Euclidean %, width rules and u32+i32 errors,
  let vs const, for-copy vs for-&x, sequential arrays can't index, frozen
  variable fields, recursion growable-data rule, reference into temporary,
  dictionary reference liveness, string literal const, push inside print in
  recursive cycles, ++ is a statement, no if(n), tinycc limits.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md#18-limitations
---

# Common pitfalls

The mistakes the tutorial and migration guide say everyone makes first.

## Arithmetic

- **`%` is Euclidean**: `-7 % 3 == 2`, never negative. Want the C
  remainder? `rem(-7, 3)` from `std`. Tile/offset math:
  `div_euclid(-1, 10) == -1` and `-1 % 10 == 9`.
- **Widths don't promote**: `u8 + u8` is an 8-bit add (wraps);
  `u32 + i32` is a compile error — cast one side. `(a as i64) * b`
  deliberately picks the width.
- **Signed overflow** aborts in `--debug` and wraps in release; **unsigned
  always wraps**. A `u8` counter incremented 256 times is 0 again.
- **No integer-to-bool**: `if n { }` is an error — `if n != 0`.
- **`++`/`--` and assignment are statements**, not expressions — `let x =
  n++;` does not exist.
- `u64` compares with signed values only when the signed side is
  syntactically non-negative (literals, `.len`, a `let` of those).

## Bindings

- **`let` ≠ immutable contents**: `let xs = [1, 2, 3]; xs[0] = 9;` is fine.
  Read-only contents come from `const T`.
- **Non-fixed values never copy implicitly**: `let b = a;` (growable
  array) binds by reference — write `copy(a)` for an independent one.
- **`for x in xs` copies** fixed-size elements (writes don't stick);
  mutate with `for &x in xs`.
- A reference to a fixed-size value decays to a copy on plain binding:
  `let d = r;` copies; `let e .= r;` is the reference.
- **Reference into a temporary**: `let s = f()[..];` is an error — bind the
  value first.
- An optional must be **narrowed** before use (`if`/`while`/`guard`/
  `assert`); `T&?` is just `T?`.

## Data layout

- **Arrays of variable-size elements can be iterated, not indexed**:
  `u8[][>..]` has no `words[2]` and no `pop`. Need indexing? Use
  `u8[..16][>..]` or an index of references.
- **Variable-class fields are frozen**: `r.name = "bb"` errors when `name:
  u8[]` — use `u8[..32]` or rebuild the container.
- A struct holds **one resizable tail, last field only**; several growing
  fields → `root struct` (local/global only, never inside another struct).
- `u8[varint]`/`T[u8]` lengths are checked — a construction exceeding them
  is rejected (statically when known) or aborts.
- String literals are `const u8[:]`: they key a `dictionary<const u8[:],
  V>`, not a `dictionary<u8[:], V>`.

## Control flow and functions

- **`guard` without `else` only skips the rest of its block** — to leave a
  function or loop, put `return`/`break` in the else.
- No labels: exit two loops with a `block { … break … }` or a nested
  function's `return`.
- `match` over an integer needs `_`; over an enum every variant or `_`.
- Default parameters are trailing only, and never see the caller's locals.
- **Recursion**: mark `recursive fn`, write full signatures, and never hold
  a growable local (or a `str(...)` temporary) across a call back into the
  cycle — pass pools in, or scope scratch to end first.

## Containers and lifetimes

- A shrink (`clear`/`pop`/`resize` down) is rejected while a **reference
  into the array is still live** — the error names the reference and its
  binding site. Use the view for the last time first.
- **Dictionary references die at the next mutation**: finish with the
  `get` result before `insert`/`update`/`remove`/`clear` (or use
  `update`).
- `pool.free(i)` does not invalidate anything — a stale reference later
  reads a *different, valid value*. Track generations yourself if it
  matters.
- Values holding self-relative links **cannot be copied** — construct in
  place, bind with `.=`, take by reference.

## Toolchain

- Threads need `goose build` (TinyCC cannot place TLS); `goose run`
  refuses them.
- `extern fn` beyond C99 `<math.h>` doesn't link under `goose run`.
- Resource paths: `resource_dir()` is the source's directory under JIT,
  the executable's when built.
- Generic errors show the **instantiation chain** — read it bottom-up.
- Lifetime errors name **both ends** — read the whole message.

## Related skills

- [reading-errors](../reading-errors/skill.md)
- [migrating-from-cpp-rust](../migrating-from-cpp-rust/skill.md)
- [lifetimes-roots](../../references/lifetimes-roots/skill.md)
