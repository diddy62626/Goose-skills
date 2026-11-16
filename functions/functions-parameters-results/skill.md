---
name: goose-functions-parameters-results
title: Functions — parameters, defaults, multiple results, overloading
category: functions
tags: [goose, fn, parameters, defaults, multiple return values, overloading]
triggers: [goose function, fn syntax, default parameters, multiple returns, overload resolution]
description: >
  Free functions with UFCS method syntax. Parameters are let bindings (var
  for reassignable); reference/slice/by-value parameter kinds; trailing
  default parameters evaluated per call; multiple results as a calling
  convention (-> A, B; let a, b = f()); overload resolution rules and
  order-independence; inferred result types and in-place returns.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#17-functions
---

# Functions: parameters, results, overloading

## Shape

```goose
fn name(a: i64, var b: f64, xs: i32[:]) -> i64 { … }
fn generic<T>(a: T, b: T) -> T { … }
fn also_generic(a, b) { … }                  // untyped parameters are generic
fn scaled(a: i32, b: i32 = 0, c: f32 = 1.0) -> f32 { … }
```

Functions are free functions (no methods/impl), but `x.f(a)` calls
`f(x, a)` — UFCS gives method syntax everywhere. Top-level declarations are
order-independent.

## Parameters

- A parameter is a `let` binding: not reassignable; `var b: f64` makes a
  by-value parameter a reassignable local.
- Untyped → generic. `n: i64&` binds an lvalue of the pointee type without
  `&`. A slice parameter `xs: T[:]` takes every array kind. A by-value
  non-fixed parameter (`xs: u8[>..]`) is constructed in the argument slot
  and owned by the callee — pass `f(copy(xs))` or `f(make())`.
- **Defaults**, trailing only: evaluated at each call that leaves them out,
  after the written arguments. May name globals and call functions — never
  the caller's locals. A generic parameter's default adapts to the type the
  arguments bind and never decides a type variable itself.

```goose
var counter = 0;
fn bump(r: i64& = counter, by: i64 = 1) { r += by; }
bump(); bump(); bump(mine, 5);
```

## Multiple results: a calling convention, not a tuple

```goose
fn divide(a: i64, b: i64) -> i64, bool {
    if b == 0 { return 0, false; }
    return a / b, true;
}
let q, ok = divide(7, 2);
r, flag = divide(1, 0);          // assign to places that already exist
```

A call in an ordinary value context yields the first; `let a, b = f();`
binds all; `return f();` forwards all. Only `return` and a call carry
several values — a branch cannot end in `a, b`. By convention a trailing
`bool` says whether the rest means anything.

## Inferred results and in-place returns

A result type may be left out where inferable, taken from the first
`return` (or the body's final expression). A non-fixed lvalue other than the
function's own local, or an explicit `&x`, makes the result a reference; a
reference to a fixed-size value is returned as a copy. A non-fixed result is
**constructed directly in its destination**; when every `return` returns the
same local, that local *is* the destination from its declaration on
(see in-place-construction). Across a recursive cycle every signature must
be written out.

## Overloading

Resolved statically on parameter types: unique exact match first, then
generic exact match, then matches needing a coercion (array→slice, literal
fitting a type, implicit widening). A candidate's rank is its worst
argument's rank; ties are errors. A default does not count as an argument.
Tag dispatch (case functions) is tried only when no ordinary candidate
matches. A candidate converting an integer argument to a float comes last
(`f(2)` calls `f(i32)` beside `f(f64)`; `sqrt(n)` with integer `n` is
ambiguous — write `sqrt(n as f64)`). The expected result type never selects
an overload.

```goose
fn describe(x: i64) -> u8[>..]     { str("int ", x) }
fn describe(x: f64) -> u8[>..]     { str("float ", x) }
fn describe(xs: i64[:]) -> u8[>..] { str(xs.len, " ints, first ", xs[0]) }
```

## Nested functions

A function may be declared inside a function and reads/writes the enclosing
function's locals as free variables (see nested-functions).

## Related skills

- [generics](generics/skill.md)
- [function-values-blocks](function-values-blocks/skill.md)
- [in-place-construction](../memory-model/in-place-construction/skill.md)
