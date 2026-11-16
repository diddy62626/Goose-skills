---
name: goose-compiler-internals
title: Compiler internals — whole-program model, specialization, generated C
category: tooling
tags: [goose, compiler internals, whole program, monomorphization, generated c, runtime, emit-runtime]
triggers: [goose compiler internals, how goose compiler works, generated c, specialization, runtime object]
description: >
  Goose compiles the whole program (closed world) in call-graph order: it
  computes the data stacks, specializes functions per root/constness/
  types, generates C, and links a once-compiled runtime. What agents should
  know: no separate compilation, static function values, the roots
  analysis, BCE, --specs/--dump inspection, and the goose-in-goose
  self-hosting bootstrap.
source: https://github.com/aardappel/goose/blob/main/docs/implementation.md
---

# Compiler internals

What an agent or contributor needs to know about how the compiler works.

## Whole-program compilation

The compiler reads the **whole program** from the root file, in call-graph
order. This closed-world model is what makes the memory model provable:

- N data stacks are computed once for the program (the deepest point
  needs); a program needing more than `GS_MAX_STACKS` fails to compile.
- Functions are **specialized per distinct root, constness, and type
  arguments** — the precision of Rust lifetimes with no annotations.
- Function values are compile-time entities; every call is direct and
  inlinable.
- No separate compilation, no shared libraries of Goose code.

`-O0/-O1/-O2` control how much the Goose compiler inlines (constant folding
runs at every level); the generated C is compiled optimized by the C
compiler regardless.

## The pipeline and how to inspect it

```
parse → resolve → typecheck (roots, provenance, shrink rules, constness)
      → optimize (inlining, BCE, tail calls) → C generation
```

- `goose --dump file.goose` prints the parsed program back as source;
  `--roundtrip` checks dump→parse→dump stability.
- `goose --tokens` shows the token stream; `--parse` stops after parsing.
- `goose check --stacks` reports the program's data stacks.
- `goose --specs` prints every specialization after optimization, as source
  — the fastest way to see what your generic function compiled to.
- `goose --stats` reports declarations parsed, specializations and
  optimizations, bounds checks removed, files written.
- `goose --dump-docs` prints the documentation comments of the program's
  declarations, fields and variants (the run of `//` lines above each).
- `goose -o prog.c` writes the generated C — read it when a kernel's speed
  surprises you.

## Bounds-check elimination

BCE runs on the specialization. Sources of provable bounds: grow-only
shrink points are visible; `%` is Euclidean (results in range); fixed sizes
are constants; each call site records what it proves about its arguments;
`assert` states lengths once outside loops; index types (`u32 in nodes`)
move checks to construction. `--bce-lines` reports per line; `checkfree fn`
turns a surviving check into a compile error; `// bce:elide` / `// bce:keep`
annotations are verifiable with `--bce-test`.

## The runtime

The program's C holds the part of the runtime inlined into it; the rest
(the OS layer, data-stack reservation, threads, printing) is the runtime's
C, written by `--emit-runtime goose_runtime.c`, compiled **once per C
compiler and configuration**, and linked with every program. The runtime
hash in `goose version` must match — a program from another compiler
version fails to link against a stale runtime object. `--standalone` writes
one C holding everything.

## Self-hosting

The test suite contains a **Goose-in-Goose bootstrap**: a compiler written
in Goose, built through three generations with a C fixed point (stage 2 ==
stage 3) — the compiler's own test of the language's fitness for systems
work. `python test/run_tests.py --goose-in-goose-only --exe <goose>` runs
just that chain.

## Sources

`docs/implementation.md` (pass by pass: typechecker roots and provenance,
shrink rules, optimizer, BCE, code generation, runtime), `docs/design/`
(the reasoning per feature: index_types, const_for, relative_references,
serialization, namespaces, packages, jit_backend, …), `docs/testing.md`
(the fixture suite and runner).

## Related skills

- [cli-reference](../cli-reference/skill.md)
- [bounds-check-elimination](../../performance/bounds-check-elimination/skill.md)
- [diagnostics](../diagnostics/skill.md)
