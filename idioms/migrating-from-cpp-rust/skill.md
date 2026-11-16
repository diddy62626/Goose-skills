---
name: goose-migrating-from-cpp-rust
title: Migrating from C++ and Rust — pattern by pattern
category: idioms
tags: [goose, migration, c++, rust, translation, patterns]
triggers: [migrate to goose, goose vs rust, goose vs c++, translate pattern, equivalent of vector]
description: >
  The translation table: new/Box::new → locals or push into an array;
  vector/Vec → T[>..]; string → u8 arrays; virtual functions → enums +
  case functions; traits → overloads/generics; shared_ptr → slot_pool;
  Result/? → trailing bool + return E from f; Arc<Mutex> → typed queues;
  serde → to_bytes; RAII → defer. Plus what does not translate and the
  nearest shape.
source: https://github.com/aardappel/goose/blob/main/docs/Goose%20for%20C%2B%2B%20and%20Rust%20Programmers%20Translating%20Common%20Patterns.md
---

# Migrating from C++ and Rust

Most code translates once you stop asking "where do I allocate this?" and
start asking "**which scope owns this, and what array does it live in?**"

## The quick reference

| You write in C++ / Rust | You write in Goose |
|---|---|
| `new T`, `make_unique`, `Box::new` | a local, or `push` into an array that outlives you |
| `vector<T>`, `Vec<T>` | `T[>..]` (grow-only); `T[>..<]` if you pop |
| `string`, `String` | `u8[>..]` builder; `u8[]` finished; `u8[..16]` inline |
| `string_view`, `&str`, `span`, `&[T]` | `T[:]` slice (checked, can't dangle) |
| `x.clone()`, copying a vector | `copy(x)`, always explicit |
| virtual method, `dyn Trait` | `enum` + `match`, or case functions |
| `vector<unique_ptr<Base>>` | `Shape[>..]` (fixed mode) or `Shape..[>..]` (packed) |
| trait bound, C++ concept | untyped / `<T>` generic, checked per instantiation |
| `impl Hash for K`, `operator<<` | a free `hash(k: K)` or `format(out, v: K)` overload |
| methods, `impl` blocks | free functions; `x.f(a)` is `f(x, a)` |
| `std::function`, stored `Box<dyn Fn>` | compile-time function value; stored behavior becomes an enum tag |
| `Box<Node>` tree, `Vec<Node>` + u32 indices | `Node[>..]` pool + `Node&<u32>?` links |
| `shared_ptr`, `Rc<RefCell<T>>`, object pool | `slot_pool<T>` + `alloc_ref`/`free` |
| struct owning three `Vec`s | a `root struct` whose three fields grow independently |
| `Result<T, E>`, `?`, `throw` | trailing `bool`, `T?`, `return E from f`, `exception::try` |
| RAII destructor | scope exit (memory); explicit `release` + `defer` (external handles) |
| `Arc<Mutex<T>>`, threads + shared data | `thread_fn` + typed queues of flat values |
| serde, protobuf | `to_bytes` / `from_bytes<T>` / `bytes_of` |
| `std::optional<T&>`, `nullptr` | `T?`, narrowed by `if`/`guard` |
| `std::tuple`, `(a, b)` returns | `-> A, B` multiple results (no tuple type) |

## The shape hierarchy, translated

```goose
enum Shape { Circle { r: f64 }, Rect { w: f64, h: f64 }, Dot }

fn area(s: Shape.Circle) -> f64 { 3.14159 * s.r * s.r }   // case functions:
fn area(s: Shape.Rect) -> f64   { s.w * s.h }             // the virtual call
fn area(s: Shape.Dot) -> f64    { 0.0 }
```

Adding an operation is easy (new case functions anywhere); adding a *kind*
stops compilation until every case handles it — the pure-virtual guarantee,
enforced at call sites. Pick storage mode at the use site: fixed mode for
indexable/mutable collections, variable mode for compact trees and records.

## Visitor / double dispatch

Exists to fake pattern matching — write the `match`. For collide-A-with-B,
nest two matches or write a case-function set whose body matches on the
second argument.

## Callbacks and closures

Passed down, never stored. `each_step(lo, hi, 3) { if it % 7 == 0 {
return it; } }` — a `return` in a block leaves your function, giving hand-
written iterators `for`'s control power. Stored callbacks become enum tags
plus data (which can then be saved to disk or sent to a worker).

## The habits that change

- `cond ? a : b` → `if` is an expression.
- `if (!x) return;` chains → `guard x else { … }`; `break value` exits a
  `block` with a value.
- `x++` in an expression → statements only.
- `if (n)` on an integer → error; `if n != 0`.
- `-7 % 3 == -1` → Euclidean: `-7 % 3 == 2` (`rem` for the C behavior).
- Implicit promotion to `int` → arithmetic at the operands' width; narrowing
  via `as`/`as!`.
- `for (auto& x : v) x *= 2;` → `for &x in xs { x *= 2; }`.
- `printf("%d %s", …)` → `print("n=", n, " s=", s)`.
- Labelled `break 'outer` → no labels; use a `block` or nested function.
- Recursion "just works" → mark the cycle `recursive fn`, write signatures,
  keep growable locals out of the recursive path.
- Separate compilation, `.so` plugins → whole-program compilation to C.

## What does not translate

| You want | Why | Nearest Goose shape |
|---|---|---|
| runtime plugins / open class hierarchy | no function pointers, no vtables, whole-program compilation | an enum of known kinds; truly external code via `extern fn` |
| closures stored in a struct / returned | function values stay compile-time | enum tag + captured data; case functions |
| `Vec<Vec<T>>` with independent inner growth | an array element can't be resizable | `slice_pool<T>` runs, flat array + offsets, or `T[..k]` rows |
| random access into variable-size records | offsets depend on earlier elements | fixed-size elements, or an index of references beside it |
| reference into a fixed-mode enum payload | could be overwritten | variable mode (`T..`), or copy-modify-reassign |
| `shared_ptr` with last-owner cleanup | no refcounts/destructors | pick an owner; `slot_pool` + explicit `free`; generation field |
| shared mutable state between threads | no shared memory by design | messages through typed queues |
| subprocesses, networking | not in the v1 library | `extern fn` to C |

## Related skills

- [common-pitfalls](../common-pitfalls/skill.md)
- [pattern-catalog](../pattern-catalog/skill.md)
- [linked-structures](../../data-modeling/linked-structures/skill.md)
