---
name: goose-limitations
title: Limitations and v1 scope — what Goose does not have (yet)
category: ecosystem
tags: [goose, limitations, v1, omissions, roadmap, wasm, coroutines]
triggers: [goose limitations, goose missing features, v1 omissions, goose roadmap, why not]
description: >
  The full v1 omission list: no move for resizables, no nested owning roots,
  no labeled break, no namespace privacy, no subprocesses or networking,
  no escaping closures/function pointers, whole-program compilation only,
  pool semantics, plus proposals on the roadmap (coroutines, wasm,
  child_threads, ownership, debugging) and how each limitation is worked
  with.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md#18-limitations
---

# Limitations and v1 scope

You will meet all of these. Each with the rule that imposes it and the way
to work with it.

## Language limitations

- **You think about where data lives** — "who owns this, how long does its
  scope last". Most of the time: the function that builds it. Non-nesting
  lifetimes → library pools.
- **Recursive functions cannot keep growable data across a recursive
  call.** Scratch in a block that ends before the call is fine; anything
  longer is passed in. Restructures recursive-descent parsers: state in the
  non-recursive entry (or a `Ctx` of references).
- **Arrays of variable-size elements cannot be indexed.** Choose compact-
  and-walkable vs indexable, or spend memory on an index of references.
- **A fixed-mode enum cannot be pointed into; a variable-mode one cannot be
  overwritten.** Choose per use site.
- **No closures that escape, no function pointers, no dynamic dispatch
  beyond ADT tags.** The closed-world compiler is what makes roots and
  destinations static; it also stops plugin systems. Nearest shape: enum of
  known kinds; external code via `extern fn`.
- **Whole-program compilation.** No separate compilation, no shared
  libraries of Goose code.
- **A pool manages its elements loosely.** Nothing is freed and nothing is
  unsafe, but using a slot after handing it back reads whatever its next
  owner put there — a logic bug you avoid yourself (generations).
- **An ordinary struct has at most one resizable tail.** `root struct`
  bundles state with independently stored fields — owner local/global only.
- **No labeled break** — `block { … break … }`, nested function `return`,
  or a flag.
- **No namespace privacy** — encapsulation by convention.
- **No move for resizables, no nested owning root structs, no arrays of
  owners** (v1 deferrals).

## Library limitations (v1)

- **No subprocesses or networking** — arrive as `extern fn`s when needed.
- **No file handles**: whole-file operations only (`os` module).
- **No stack traces** for exceptions; `abort` messages carry a line, not a
  backtrace.
- **Non-standard FFI packages are outside worker isolation** — by
  convention their C is thread-safe unless documented otherwise; `gfx`/`ui`/
  `audio`/`physics` are main-thread-only (compile error from a `thread_fn`).
- **`reusable`/`reusable[]` are deprecated** in favor of `slot_pool`/
  `slice_pool`; owner transfer to workers is a remaining migration gap.

## On the roadmap (design proposals, not implemented)

Documented in `docs/design/` as proposals/research notes: stackless
**coroutines** (frames as ordinary fixed-size values, `resume`), **WebAssembly**
without address-space reservation, **child_threads** (`par for` fork-join),
an **`own` qualifier** (ownership without destructors), **debugging** (Goose
source in a debugger via a `goose dap` adapter over lldb-dap), **nested
growth** (building recursive trees in place on one stack), **SoA column
arrays**, and `abortfree`/`pure`-style contracts (llm_frontier).

Read a proposal as design thinking, not a feature promise.

## Related skills

- [what-is-goose](../getting-started/what-is-goose/skill.md)
- [common-pitfalls](../idioms/common-pitfalls/skill.md)
- [recursion](../functions/recursion/skill.md)
