---
name: goose-generics
title: Generics — untyped parameters, <T>, monomorphization, no trait bounds
category: functions
tags: [goose, generics, monomorphization, type parameters, instantiation errors]
triggers: [goose generics, fn foo<T>, untyped parameter generic, type inference, monomorphized]
description: >
  An untyped parameter is generic; <T> says two parameters agree or names a
  type no parameter carries. Everything is monomorphized: a generic body is
  checked only at concrete instantiations, with the compile-time call chain
  reported. Type argument inference, explicit lists, literal parameters,
  floating context, and .='s move semantics in generic code.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#18-generics-and-function-values
---

# Generics

## The basics

```goose
fn twice(x) { x + x }                                    // untyped parameter: generic
fn span<T>(a: T, b: T) -> T { if a > b { a - b } else { b - a } }
fn zero<T>() -> T { default<T>() }                       // names a type no parameter carries

let half = zero<f64>() + 0.5;
```

`<T>` either says that two parameters must have the same type, or names a
type that no parameter carries. There are **no trait bounds**: a generic
body may do anything the concrete types allow, checked per instantiation.

## Monomorphization and error reports

Everything is monomorphized. A generic body is typechecked only at an
instantiation where every type is concrete; errors are reported at the
offending instantiation **with the compile-time call chain that produced
them** (each frame as `in fn … instantiated from file:line`). Read the chain
bottom-up to find which argument caused it.

## Type argument inference

- Inferred from argument types; where several arguments mention one
  variable, the typed ones bind it and a literal then adapts
  (`max(n, 0)` with a `u32` `n` is the `u32` instantiation).
- A float literal binds before an integer one; `[]` and `null` bind nothing.
- An explicit list `f<i64>(x)` binds the leading type parameters in order;
  required when no argument mentions a parameter (`qget<i64>()`,
  `zero<f64>()`).
- `f<` begins a type argument list only when `<…>` is followed by `(`, `{` or
  `.Variant {`; otherwise `<` is a comparison.

## Literal parameters

Where a literal is the only thing binding a type variable, the parameter is
a *literal parameter* inside the specialization — a constant of unknown
value that adapts wherever it meets a type: with
`fn push_n<A, T>(xs: A&, v: T, n: i64)`, `push_n(flags, 1, n)` pushes a `u8`
with no cast, and `push_n(flags, 300, n)` is an error naming the `push`.

## Floating context

An `f32` destination may refine an otherwise unconstrained float computation
in a generic call: `let scale: f32 = max(0.5, min(width / 960.0, height /
600.0));` computes both calls in `f32` (integer `width`/`height`). Applies
to a function returning bare `T` whose `T` parameters are bare values, given
literals and float expressions of literals and integers. A typed argument
fixes the type.

## `.=` moves whole values in generic code

Inside generic code, `.=` on a location that is not a reference assigns as
`=` does. Where `T` is a reference type, `.=` moves the reference itself
(where `=` would write through it); where `T` is a value, plain assignment.
This is how the library's `swap_at`/`sort`/`reverse` reorder arrays of
references, including relative ones.

## Structs are generic too

```goose
struct Pair<T> { a: T, b: T }
let q = Pair { a: 0.5, b: 1.5 };     // T inferred from fields
```

Instantiations must be finite (a field naming the struct again with a larger
type argument is rejected).

## Type parameters are types, never sizes

`fn f<N>(a: i64[N])` is an error; take a slice `T[:]` instead.

## Related skills

- [function-values-blocks](function-values-blocks/skill.md)
- [const-for-reflection](../control-flow/const-for-reflection/skill.md)
- [functions-parameters-results](functions-parameters-results/skill.md)
