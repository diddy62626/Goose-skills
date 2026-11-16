# getting-started

**Start here** — What Goose is, installing it, and a syntax tour for people who know C/Rust/Go.

3 skills. Load any of them as `getting-started/<skill>/skill.md`.

- [`goose-install-and-toolchain`](getting-started/install-and-toolchain/skill.md) — Installing Goose, the toolchain, and running your first program: > Where to get the Goose compiler (releases page or build from source), what it needs (git, cmake, a C compiler), the one-executable design (compiler + runner + build tool + fetcher + formatter + LSP), GOOSE_HOME, and the first commands to try.
- [`goose-syntax-tour`](getting-started/syntax-tour/skill.md) — Goose syntax tour — the whole language in one pass: > Compact tour of Goose syntax for people who know C/Rust/Go: let/var/const, types after the name, expression-oriented if/match/block, guard, defer, loops, multiple return values, default parameters, UFCS method-call syntax, the three arithmetic rules, and struct literals with `..`.
- [`goose-what-is-goose`](getting-started/what-is-goose/skill.md) — What Goose is, and when to reach for it: > Goose is a memory-safe systems language with no heap allocator, garbage collector, or lifetime annotations. Data lives on compiler-managed data stacks freed at scope exit; values nest inline and never move. It compiles to C and benchmarks faster than idiomatic C++ and safe Rust while using less memory. AI models have little training data on it (first releases 2026), so treat this skill set as the primary knowledge source.
