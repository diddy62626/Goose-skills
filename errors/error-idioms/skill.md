---
name: goose-error-idioms
title: Error idioms — trailing bool, T?, asserts, aborts, exits
category: errors
tags: [goose, errors, bool convention, optional, assert, abort, exit]
triggers: [goose error handling, parse_int ok, assert, abort, exit code, error convention]
description: >
  No Result type, no exceptions, no required convention: the application
  chooses. Two outcomes = trailing bool (-> T, bool); found-or-not = T?
  (a reference into the input, copies nothing); bugs/invariants = assert/
  abort/exit, which never return and are uncatchable.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#stopping-the-program
---

# Error idioms

There is no exception mechanism, no `Result` type, and **no specified error
convention — the application chooses**. Goose gives lightweight tools
chosen by how far the error travels:

| Situation | Goose |
|---|---|
| Two outcomes, caller handles it right away | trailing `bool`: `-> T, bool` |
| Found or not | `T?` — a nullable reference into the input, copies nothing |
| Fails deep inside, handled far up | `return E from f` (see return-from) or `exception::try` |
| Bug or invariant | `assert(c)`, `abort(msg)`, `exit(code)` |

## Trailing bool

```goose
fn divide(a: i64, b: i64) -> i64, bool {
    if b == 0 { return 0, false; }
    return a / b, true;
}
let q, ok = divide(7, 2);
guard ok else { return; }
```

Library convention: `let n, ok = parse_int(s);`, `let f, ok = parse_flt(s);`,
`read_file(path, out) -> bool`. Positions use `i64` with `-1` for none.

## Optional reference

```goose
let big = xs.find() { it > 100 };   // T? into xs, null if none
if big { print(big); }              // narrowed: big is a T& inside
guard big else { return 0; }
```

`dictionary::get` returns `V?`; `contains` distinguishes absent from null.

## assert / abort / exit

- `assert(c)` aborts unless `c` holds, and **narrows an optional** it is
  given.
- `abort(msg)` aborts with a message.
- `exit(code)` ends the program with an exit code.

All three **never return**, and the checker takes this into account: a
branch that ends in one gives no value, statements after one are not
typechecked, and this propagates through calls (including wrappers around
`exception::throw`) — a value-returning function can end by calling one
without a dummy result. An unknown recursive call is conservatively assumed
to return. **None of them is catchable.**

```goose
fn fail(what: u8[:]) { abort(str("cannot ", what)); }
fn value(ok: bool) -> i64 {
    if ok { 42 } else { fail("compute") }    // fine: fail never returns
}
```

## Don't port sentinel-heavy code

`-1` for "no position" is fine (library convention). For anything richer,
prefer the trailing `bool` or `T?` over magic values — `T?` forces the
caller to narrow before use.

## Runtime aborts you did not ask for

Division/remainder by zero, `i64.min / -1`, signed overflow in debug,
`as` range violations in debug, a limited array's capacity overflow,
shrinking below empty, a relative-reference offset that does not fit its
width, a failed bounds check, and data-stack exhaustion all abort cleanly
with a message and the failing line. A check inside a standard library
function names the line of the program's code that called it.

## Related skills

- [return-from](../return-from/skill.md)
- [exception-module](../exception-module/skill.md)
- [optionals-and-narrowing](../../references/optionals-and-narrowing/skill.md)
