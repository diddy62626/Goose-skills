---
name: goose-what-is-goose
title: What Goose is, and when to reach for it
category: getting-started
tags: [goose, overview, memory-model, systems-language, performance]
triggers: [what is goose, goose language, gooselang, goose-lang, memory safe systems language]
description: >
  Goose is a memory-safe systems language with no heap allocator, garbage
  collector, or lifetime annotations. Data lives on compiler-managed data
  stacks freed at scope exit; values nest inline and never move. It compiles
  to C and benchmarks faster than idiomatic C++ and safe Rust while using
  less memory. AI models have little training data on it (first releases 2026),
  so treat this skill set as the primary knowledge source.
source: https://github.com/aardappel/goose
---

# What Goose is, and when to reach for it

Goose (by Wouter van Oortmerssen / `aardappel`, repo `github.com/aardappel/goose`)
is a **memory-safe systems programming language** whose pitch is: *no heap
allocator, no garbage collector, no reference counting, no lifetime
annotations* — yet memory safe and, in the repo's benchmarks, faster than
idiomatic C++ (≈3.3–3.5x), hand-optimized C++ (≈1.16–1.26x) and the best safe
Rust (≈1.12–1.19x), while using 1.2–1.9x less memory.

## The core idea (everything else follows from it)

There is **no heap**. Memory is the native call stack plus **N data stacks** —
big address-space reservations managed by a bump pointer. Four properties:

1. Growth is a pointer bump; freeing is setting the pointer back when the
   owning **scope exits**. No destructors, no per-object teardown walk.
2. Every dynamic value is **owned by a variable** (a local, global, by-value
   parameter, or a field/element of one).
3. Values **nest inline**: a string inside a struct inside an array is bytes
   inside bytes inside bytes, with no pointers between them.
4. A growable value **never moves**, so references into a growing array stay
   valid — the compiler proves it, without `'a` annotations and without `Rc`.

The compiler is closed-world (whole-program compilation to C), which is what
lets it prove all of this statically.

## What it looks like

C/Rust-flavored syntax: braces, `let`/`var`/`fn`, `struct`, `enum`, `match`.

```goose
// An array of strings: ONE contiguous block, length then five
// (length, characters) runs. No per-string allocation.
var words: u8[][>..] = [];
for i in 5 { words.push(str("word", i)); }

fn main() {
    let kind = if 42 > 50 { "healthy" } else { "hurt" };   // if is an expression
    print("kind=", kind, " words=", words.len);
}
```

## Key feature list

- **One memory model**: data stacks freed at scope exit; no malloc/free/GC.
- **Safe references, no annotations**: `T&`, `T[:]` slices, nullable `T?`,
  relative references `T&<u32>` (2–4 byte links), all checker-proven.
- **Enums in two sizes**: fixed mode (`Shape`, indexable) and variable mode
  (`Shape..`, each value exactly its variant's size, packed).
- **In-place construction**: functions returning built-up values write them
  directly into the caller's destination; `str(...)` formats into the element
  being pushed. No temporaries.
- **`return E from f`**: lightweight checked exceptions without unwinding.
- **Generics + function blocks**: monomorphized; `xs.filter() { it > 0 }`
  compiles to a direct loop. No closures that escape, no function pointers.
- **Threads share nothing**: `thread_fn` workers with typed queues of flat
  values; no locks or atomics in Goose code.
- **Serialization**: self-relative links make structures position-independent;
  `to_bytes`/`from_bytes` save/load with verification.
- **Compiles to C** and calls C: `extern fn`, `extern "lib" { ... }`,
  `export fn`, `--header`.

## When to choose Goose

- Performance-critical tools/parsers/simulations/games where allocation
  traffic and pointer-chasing dominate.
- Code with tree/graph data where Rust forces arena+index workarounds —
  Goose holds typed references into growing pools directly.
- Anything where you want C-level control with memory safety and *no* borrow
  annotations.

## Know the limitations up front (v1)

No closures that escape or function pointers (no plugins/dynamic dispatch
beyond enum tags); recursion is opt-in (`recursive fn`) and functions in a
recursive cycle may not hold growable data across the recursive call; arrays
of variable-size elements can be iterated but not indexed; whole-program
compilation only (no separate compilation/shared libs of Goose code); no
subprocesses or networking in the stdlib yet (use `extern fn`); a struct
holds at most one resizable array (its last field) — use `root struct` for
more.

## Related skills

- [syntax-tour](../../getting-started/syntax-tour/skill.md) — the language in one pass
- [data-stacks-and-ownership](../../memory-model/data-stacks-and-ownership/skill.md) — the memory model in depth
- [limitations](../../ecosystem/limitations/skill.md) — full v1 omission list
