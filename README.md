# Goose-skills

**AI-agent skills for the [Goose programming language](https://github.com/aardappel/goose)** — a memory-safe systems language with no heap allocator, garbage collector, or lifetime annotations (first releases 2026; too new to be in most model training data). These skills teach agents to read and write idiomatic Goose.

**78 skills · 17 categories** · format: `skill.md` + YAML frontmatter · grounded in the official tutorial, reference, spec, stdlib docs and command-line docs of aardappel/goose.

## How to use

- Each skill lives at `[category]/[skill]/skill.md`, e.g. `memory-model/data-stacks-and-ownership/skill.md`.
- `skills.json` is the machine-readable index (names, paths, descriptions, trigger phrases, tags, source links). Point a loader at it.
- Every skill has `triggers:` (phrases that suggest loading it) and a `source:` link to the upstream doc section it is grounded in. Verify anything time-sensitive against the upstream repo — Goose moves fast.
- Skills are tutorial + reference style: exact syntax, compilable examples, rules, pitfalls, and cross-links.

## The index

### `arrays/` — Core concepts

The array family: fixed/limited/frozen arrays, grow-only and grow-shrink resizables, slices.

- [`goose-array-family-overview`](arrays/array-family-overview/skill.md) — The array family — eight kinds, one table, which to pick
- [`goose-fixed-limited-frozen-arrays`](arrays/fixed-limited-frozen-arrays/skill.md) — Fixed, limited and frozen arrays — T[k], T[..k], T[..], T[]
- [`goose-grow-only-resizable`](arrays/grow-only-resizable/skill.md) — Grow-only resizable arrays — T[>..], the default container
- [`goose-grow-shrink-stacks-queues`](arrays/grow-shrink-stacks-queues/skill.md) — Grow-shrink arrays — T[>..<] for stacks, queues and heaps
- [`goose-slices`](arrays/slices/skill.md) — Slices — T[:] views that cannot dangle

### `concurrency/` — Working with data

Threads that share nothing: thread_fn, typed queues, worker recipes.

- [`goose-thread-model`](concurrency/thread-model/skill.md) — Threads — thread_fn workers that share nothing
- [`goose-typed-queues`](concurrency/typed-queues/skill.md) — Typed queues — qput/qget/qpoll/qlimit patterns
- [`goose-worker-recipes`](concurrency/worker-recipes/skill.md) — Worker recipes — parallel-for, fan-out/fan-in, contained supervision

### `control-flow/` — Core concepts

Expressions (if/match/block), guard, defer, loops, and const for/if reflection.

- [`goose-const-for-reflection`](control-flow/const-for-reflection/skill.md) — const for / const if — specialization-time reflection
- [`goose-expressions-if-match`](control-flow/expressions-if-match/skill.md) — Expressions — if, match, block as values; branch type joining
- [`goose-guard-block-defer`](control-flow/guard-block-defer/skill.md) — guard, block-with-break, and defer
- [`goose-loops`](control-flow/loops/skill.md) — Loops — for-in, while, loop, for &x mutation, index binding

### `data-modeling/` — Working with data

Inline nested data, index types, linked structures, root structs, pools, serialization.

- [`goose-index-for-sequential-arrays`](data-modeling/index-for-sequential-arrays/skill.md) — Index for sequential arrays — random access into packed data
- [`goose-index-types`](data-modeling/index-types/skill.md) — Index types — `u32 in nodes` and checkfree functions
- [`goose-inline-nested-data`](data-modeling/inline-nested-data/skill.md) — Inline nested data — varint records, byte counting, sequential trade-offs
- [`goose-linked-structures`](data-modeling/linked-structures/skill.md) — Linked structures — trees, lists, ASTs in pools with typed links
- [`goose-root-structs`](data-modeling/root-structs/skill.md) — Root structs — several independently growing fields under one owner
- [`goose-serialization`](data-modeling/serialization/skill.md) — Serialization — to_bytes / bytes_of / from_bytes, verified loading
- [`goose-slot-slice-pools`](data-modeling/slot-slice-pools/skill.md) — slot_pool and slice_pool — lifetimes that do not nest

### `ecosystem/` — Ecosystem

v1 limitations and scope, official resources and links.

- [`goose-limitations`](ecosystem/limitations/skill.md) — Limitations and v1 scope — what Goose does not have (yet)
- [`goose-resources`](ecosystem/resources/skill.md) — Resources — official docs, samples, community

### `errors/` — Core concepts

Error idioms, return … from, the exception module, asserts and aborts.

- [`goose-error-idioms`](errors/error-idioms/skill.md) — Error idioms — trailing bool, T?, asserts, aborts, exits
- [`goose-exception-module`](errors/exception-module/skill.md) — The exception module — typed try/throw
- [`goose-return-from`](errors/return-from/skill.md) — `return … from` — lightweight checked exceptions without unwinding

### `functions/` — Core concepts

Functions, generics, function values and blocks, nested functions, recursion, qualifiers.

- [`goose-function-qualifiers`](functions/function-qualifiers/skill.md) — Function qualifiers — inline, simd, checkfree, thread_fn, extern, export
- [`goose-function-values-blocks`](functions/function-values-blocks/skill.md) — Function values and trailing blocks — HOFs without closures
- [`goose-functions-parameters-results`](functions/functions-parameters-results/skill.md) — Functions — parameters, defaults, multiple results, overloading
- [`goose-generics`](functions/generics/skill.md) — Generics — untyped parameters, <T>, monomorphization, no trait bounds
- [`goose-nested-functions`](functions/nested-functions/skill.md) — Nested functions — shared-state parsers without context structs
- [`goose-recursion`](functions/recursion/skill.md) — Recursion — recursive fn, the growable-data rule, shared scratch

### `getting-started/` — Start here

What Goose is, installing it, and a syntax tour for people who know C/Rust/Go.

- [`goose-install-and-toolchain`](getting-started/install-and-toolchain/skill.md) — Installing Goose, the toolchain, and running your first program
- [`goose-syntax-tour`](getting-started/syntax-tour/skill.md) — Goose syntax tour — the whole language in one pass
- [`goose-what-is-goose`](getting-started/what-is-goose/skill.md) — What Goose is, and when to reach for it

### `idioms/` — Idioms

Migrating from C++/Rust, common pitfalls, reading errors, the pattern catalog.

- [`goose-common-pitfalls`](idioms/common-pitfalls/skill.md) — Common pitfalls — the gotchas that bite in the first afternoon
- [`goose-migrating-from-cpp-rust`](idioms/migrating-from-cpp-rust/skill.md) — Migrating from C++ and Rust — pattern by pattern
- [`goose-pattern-catalog`](idioms/pattern-catalog/skill.md) — Pattern catalog — the idioms Goose code is written in
- [`goose-reading-errors`](idioms/reading-errors/skill.md) — Reading compiler errors — the two-ends rule

### `interop/` — Working with data

The C boundary in both directions: extern fn, extern "lib", export fn, sidecars and CMake.

- [`goose-export-to-c`](interop/export-to-c/skill.md) — export fn — Goose as a library called from C
- [`goose-extern-fn`](interop/extern-fn/skill.md) — extern fn — calling C from Goose
- [`goose-extern-lib-and-ffi`](interop/extern-lib-and-ffi/skill.md) — extern "lib" blocks and the ffi module — binding shared libraries by name
- [`goose-sidecar-own-c`](interop/sidecar-own-c/skill.md) — Sidecar foo.build — a program or package with C of its own

### `memory-model/` — Core concepts

The data-stack memory model: ownership, size classes, copies, in-place construction.

- [`goose-copies-and-aliasing`](memory-model/copies-and-aliasing/skill.md) — Copies are visible — copy(), binding by reference, call semantics
- [`goose-data-stacks-and-ownership`](memory-model/data-stacks-and-ownership/skill.md) — The memory model — data stacks, ownership, scope exit
- [`goose-in-place-construction`](memory-model/in-place-construction/skill.md) — In-place construction — results are built at their destination
- [`goose-size-classes-and-placement`](memory-model/size-classes-and-placement/skill.md) — Size classes — fixed, variable, resizable, root — and what may contain what

### `performance/` — Tooling

The performance model, bounds-check elimination, benchmarking recipes, simd/inline.

- [`goose-benchmarking`](performance/benchmarking/skill.md) — Benchmarking recipes — measuring Goose the way the suite does
- [`goose-bounds-check-elimination`](performance/bounds-check-elimination/skill.md) — Bounds-check elimination — help the compiler prove checks away
- [`goose-performance-model`](performance/performance-model/skill.md) — The performance model — where Goose's speed comes from
- [`goose-simd-and-inline`](performance/simd-and-inline/skill.md) — simd fn and inline fn — vectorization knobs

### `references/` — Core concepts

References and UFCS, optionals and narrowing, lifetimes/roots, relative references, const.

- [`goose-const-writability`](references/const-writability/skill.md) — const and writability — read-only contents, slots, inference
- [`goose-lifetimes-roots`](references/lifetimes-roots/skill.md) — Lifetimes — roots, stores, and reading the errors
- [`goose-optionals-and-narrowing`](references/optionals-and-narrowing/skill.md) — Optionals — T?, null, and narrowing with if/while/guard/assert
- [`goose-references-and-ufcs`](references/references-and-ufcs/skill.md) — References — transparent, .= binding, &x, UFCS method syntax
- [`goose-relative-references`](references/relative-references/skill.md) — Relative references — narrow links, pool-relative, and `self`

### `stdlib/` — Working with data

The standard library module by module: std, dictionary, vec/mat/quat, math, os, binary/base64, json, csv/regex.

- [`goose-binary-base64`](stdlib/binary-base64/skill.md) — binary and base64 modules — external binary formats, checked reads
- [`goose-csv-regex`](stdlib/csv-regex/skill.md) — csv and regex modules — RFC 4180 records, Rust-regex-style matching
- [`goose-dictionary`](stdlib/dictionary/skill.md) — dictionary — the hash map (and set)
- [`goose-json`](stdlib/json/skill.md) — json module — writer, tree parser, and streaming reader onto your types
- [`goose-math-module`](stdlib/math-module/skill.md) — math module — libm at both widths, constants, helpers
- [`goose-os-module`](stdlib/os-module/skill.md) — os module — files, args, env, time, stdin, resources
- [`goose-std-algorithms`](stdlib/std-algorithms/skill.md) — std module — array algorithms, sorting, hashing, rng
- [`goose-vec-mat-quat`](stdlib/vec-mat-quat/skill.md) — vec, mat, quat — the numeric geometry modules

### `strings/` — Core concepts

Strings as byte arrays, formatting text, zero-copy parsing and manipulation.

- [`goose-formatting-text`](strings/formatting-text/skill.md) — Formatting text — print/format/str and the format overload
- [`goose-parsing-text`](strings/parsing-text/skill.md) — Parsing and manipulating text — split/trim/find/parse_int, zero-copy
- [`goose-strings-as-byte-arrays`](strings/strings-as-byte-arrays/skill.md) — Strings are byte arrays — four roles, no string type

### `tooling/` — Tooling

The goose CLI, run vs build, diagnostics, packages, editors/LSP, compiler internals, testing.

- [`goose-cli-reference`](tooling/cli-reference/skill.md) — CLI reference — every goose command and option
- [`goose-compiler-internals`](tooling/compiler-internals/skill.md) — Compiler internals — whole-program model, specialization, generated C
- [`goose-diagnostics`](tooling/diagnostics/skill.md) — Compiler diagnostics — reading errors, JSON output, stats
- [`goose-editors-lsp-format`](tooling/editors-lsp-format/skill.md) — Editors, the language server, and goose format
- [`goose-making-a-package`](tooling/making-a-package/skill.md) — Making and publishing a Goose package
- [`goose-packages`](tooling/packages/skill.md) — Packages — declaring, fetching, pinning dependencies
- [`goose-run-build-backends`](tooling/run-build-backends/skill.md) — run vs build — the two backends and when each works
- [`goose-testing-debugging`](tooling/testing-debugging/skill.md) — Testing and debugging Goose programs

### `types/` — Core concepts

Scalars and arithmetic, casts, structs, enums/ADTs, aliases and constants.

- [`goose-aliases-and-named-constants`](types/aliases-and-named-constants/skill.md) — Type aliases and named constants
- [`goose-casts-and-conversions`](types/casts-and-conversions/skill.md) — Casts and conversions — implicit widening, `as`, `as!`
- [`goose-enums-adts`](types/enums-adts/skill.md) — Enums (ADTs) — two storage modes, match, case functions
- [`goose-scalars-and-arithmetic`](types/scalars-and-arithmetic/skill.md) — Scalars, numeric literals, and the arithmetic rules
- [`goose-structs`](types/structs/skill.md) — Structs — literals, field defaults, qualifiers, generic structs

## Sources

All content is distilled from the official Goose documentation (https://github.com/aardappel/goose), cloned and read in full on 2026-11-16: tutorial.md, reference.md, goose_spec.md (structure), stdlib.md, command_line.md, packages.md, testing.md, editors.md, the C++/Rust translation guide, bench/summary.md, the design notes index and the samples list. Benchmarks quoted are from the October 16, 2026 run.

Goose is by Wouter van Oortmerssen (aardappel). Goose itself is MIT-licensed; this skills repo is MIT too.
