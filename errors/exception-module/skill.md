---
name: goose-exception-module
title: The exception module — typed try/throw
category: errors
tags: [goose, exception, try, throw, typed errors, finally]
triggers: [goose exception module, exception try, exception throw, typed error, finally]
description: >
  exception::try<E>(body) { error => ... } and exception::throw(value)
  package the return-from machinery as typed exceptions. E is fixed at try
  (u8[] for owned text, an int, or your own struct/enum). Ownership rules,
  recursion restrictions, and the defer-based finally.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#exception
---

# The `exception` module

Typed exceptions implemented in Goose with `return … from`. Import
`exception`; its functions are in namespace `exception`.

```goose
fn try<E, Body, Catch>()            // Body(); on an error, Catch(error)
fn try<E, Body>()                   // Body(); discard an error
fn try<E, Body, Catch, Finally>()   // as the first, then Finally() on every way out
fn throw(value)                     // exit the nearest enclosing handler
```

```goose
import exception;

fn load() {
    // any number of calls below try can throw
    exception::throw(str("cannot load item ", 42));
}

exception::try<u8[]>(load) { error => print(error); };
```

## The error type

`E` is supplied explicitly at `try`: `u8[]` for owned text, an integer, or an
application's struct or enum (`Error..` for a variable-mode enum). The thrown
value must fit `E`, which must be usable as an array element. The handler
returns an empty `E[]` on success and a singleton containing the error on
failure — this uses the ordinary data stacks, needs no default `E`, and
allows `0`, `false` and `""` as errors. The catch runs after the handler has
returned.

## Ownership rules

- An existing non-fixed value needs `copy` when first thrown; fresh
  constructors and string slices can be supplied directly. Owned text is
  copied before the frames containing it exit.
- References carried by a long-distance return must point to **globals or
  static data**. Use owned `u8[]`, not a slice into a local string, and
  owned fields in error records.
- Calling `throw` in the catch throws to an **outer** `try`; use
  `exception::throw(copy(error))` to rethrow an owned variable-size error.

## What is still the language's rules

- Every throw must have a handler on every compile-time call path, including
  throws from a catch; an incompatible error type is a compile error.
- Recursive code can throw to an outer handler, but a `try` cannot itself be
  re-entered recursively (its non-fixed handler result would hold a data
  stack across the recursive call).
- The checker propagates calls that never resume their caller: a function
  declared to return a value may end with `exception::throw(error);`, and a
  throwing branch needs no result. But `throw` has no value type — use it as
  a statement or branch, not `return throw(error);`.
- Only calls to `throw` are caught. Runtime failures, `abort` and `exit`
  remain uncatchable. There is no stack-trace facility.

## `finally` is a `defer`

`try<E>(body, catch, finally)` runs `finally` after the body completes,
after a caught throw, and when a throw or a lexical return aimed at another
function leaves through the call. It is `defer finally();` ahead of the try,
so `finally` follows the defer rules: it cannot leave, and it may not throw,
catch, or call anything that does.

## One trailing block

Goose permits one trailing block per call, so name at least one of the
callbacks: `exception::try<u8[]>(load) { error => ... };` — or
`exception::try<u8[]>(load, on_error)` with a named function.

## Related skills

- [return-from](return-from/skill.md)
- [error-idioms](error-idioms/skill.md)
- [guard-block-defer](../control-flow/guard-block-defer/skill.md)
