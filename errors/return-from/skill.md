---
name: goose-return-from
title: `return … from` — lightweight checked exceptions without unwinding
category: errors
tags: [goose, return from, errors, parser, unwinding, long-distance return]
triggers: [goose return from, error propagation, exceptions without unwinding, parser errors]
description: >
  return E from f returns E as the result of the innermost active call of f,
  unwinding every frame in between — while every function in between keeps
  its plain signature. Statically checked (no uncaught case), implemented as
  a hidden discriminant (no unwind tables), and the error value is built
  directly at the target's destination.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#return--from
---

# `return … from`

`return E from f` returns `E` as the result of the innermost active call of
`f`, unwinding every frame in between. The functions in between keep their
**plain signatures**: no error parameter, no `?` on their calls, nothing to
propagate.

## The parser idiom

```goose
fn load(text: u8[:]) -> User[>..], u8[] {          // users, error message
    var users: User[>..] = [];
    var lineno = 0;
    each_split(text, '\n') {
        lineno++;
        if it.len > 0 { users.push(parse_record(it, lineno)); }
    };
    return users, "";
}

fn parse_record(line: u8[:], lineno: i64) -> User {  // plain User, no Result
    let comma = find(line, ",");
    guard comma >= 0 else {
        return [], str("line ", lineno, ": expected name,age") from load;
    }
    let age, ok = parse_int(trim(line[comma + 1..]));
    guard ok else { return [], str("line ", lineno, ": bad age") from load; }
    return User { name: trim(line[..comma]), age: age as i32 };
}

fn report(text: u8[:]) {
    let users, err = load(text);
    if err.len > 0 { print("error: ", err); return; }
    print("loaded ", users.len, " users");
}
```

A failure eight calls deep needs no error plumbing in any of the eight
signatures.

## Three properties keep the cost low

- **Checked statically.** Every call site of a function containing
  `return … from load` must lie within the dynamic extent of a `load` call;
  the whole-program compiler verifies that in call-graph order. There is no
  "uncaught" case at runtime.
- **A hidden discriminant, not an unwinder.** Each frame on the path gains a
  flag ("this result is mine" / "propagate to `load`") and a caller checks it
  and returns at once in the propagate case. Normal epilogues run, so every
  data-stack watermark restores itself. No unwind tables, no `setjmp`, no
  destructors — but the `defer` bodies of blocks being left do run.
- **The result is built where `load`'s caller wants it.** `str(…)` builds the
  message directly at `load`'s return destination, bypassing the frames being
  unwound.

Success-path cost: one flag check per frame on the way out. That is why the
sample parsers use it for **every** syntax error, not just catastrophes.

## Resolution and restrictions

- `f` is resolved where the returning function is written: a nested function
  in scope, else the name's overload set; `from ns::parse` names a namespace
  explicitly. An unrelated function sharing the name never catches the
  return.
- The current compiler lets a long-distance return written in a named
  function carry references to globals and static data only — return owned
  values (like `u8[]` messages above).
- The same machinery lets a **block** `return` from its lexically enclosing
  function through a higher-order function (see function-values-blocks).

## Related skills

- [exception-module](../exception-module/skill.md)
- [guard-block-defer](../../control-flow/guard-block-defer/skill.md)
- [error-idioms](../error-idioms/skill.md)
