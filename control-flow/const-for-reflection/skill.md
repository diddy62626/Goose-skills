---
name: goose-const-for-reflection
title: const for / const if — specialization-time reflection
category: control-flow
tags: [goose, const for, reflection, fields, variants, types, has_default, same, metaprogramming]
triggers: [goose const for, iterate fields, variants<T>, compile time reflection, has_default, same type]
description: >
  const for expands a fragment once per entry of a compile-time sequence:
  a struct's fields (fields<T>), an enum's variants (variants<E>), a list of
  types (types<...>), or a list of constants. const if chooses fragments at
  specialization time with has_default<A> and same<T1,T2>. Goose's
  reflection, replacing derive macros.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#const-for-expansion-at-specialization-time
---

# `const for` and `const if`: specialization-time reflection

`const for` specializes a fragment once per entry of a compile-time
sequence: the fields of a struct, the variants of an enum, a list of types,
or a list of constants. Each copy of the body is ordinary code — no runtime
iterator, no metadata.

## `fields<T>` — a struct's fields

Binds each field's name (a string constant) and an *accessor*, usable as
`v.A` or as the label `A:` in a literal of that struct type:

```goose
struct Point { x: i64, y: i64, label: u8[..8] }

fn show_fields<T>(v: T&) {
    const for <N, A> in fields<T> { print(N, " = ", v.A); }
}

fn scaled<T, F>(v: T&) -> T {
    T { const for <N, A> in fields<T> { A: F(N, v.A), } }
}
```

This is "print every field" and "map every field" without macros.

## `variants<E>` — an enum's variants

Binds each variant's name and the variant `V`, written wherever a variant's
name would be: `V` as a type (and `fields<V>` for its payload), `V { … }` to
build one, `V c` / `V &c` as a match arm, `V` as a value when it has no
payload. A `match` fragment contributes arms, checked like written arms
(every variant once, `_` last):

```goose
fn show<T>(value: T) {
    match value {
        const for <N, V> in variants<T> {
            V v => {
                print(N);
                const for <F, A> in fields<V> { print("  ", F, " = ", v.A); }
            },
        }
    }
}
```

## `types<…>` and constant lists

```goose
const for <U> in types<i8, u16, f32> { print(default<U>()); }
const for <C> in [1, 2, 4] { let a: u8[C] = [0; C]; print(a.len); }
```

A statement fragment is a scope of its own per entry; a constructor fragment
contributes initializers. `break`/`continue` cannot cross an expansion.

## `const if` — selection at specialization time

Chooses one fragment list while a function is specialized; the others
generate nothing. Stands where a `const for` stands: in a statement list, a
constructor's initializer list, or a match's arm list. Chain with
`else const if`; a plain `else if` after a `const if` is an error.

```goose
// Copy of src with every field that declares a default reset to it.
fn reset_defaults<T>(src: T&) -> T {
    T { const for <N, A> in fields<T> { const if !has_default<A> { A: src.A, } }, .. }
}

fn describe<T>(v: T) {
    const if same<T, i64> { print("an integer ", v); }
    else const if same<T, Shape> { print("a shape"); }
    else { print("something else"); }
}
```

Conditions: `true`/`false`, a named `bool` constant, the queries
`has_default<A>` and `same<T1, T2>`, `!`/`&&`/`||` over conditions, or a
comparison of two constants of one kind. A local, a call, a field read —
anything computed at runtime — is an error, not a runtime branch.

The branch not chosen is parsed but never checked — the point: a `const if`
over a field can write to it in one branch and leave it alone in the other.

## Use cases

- Serialization/reflection dumps (`--dump-docs`-style tooling in Goose code).
- Per-field or per-variant generic algorithms (formatting, validation,
  memberwise transforms).
- Specializing by type (`same<T, i64>`) without trait machinery.

## Related skills

- [enums-adts](../../types/enums-adts/skill.md)
- [structs](../../types/structs/skill.md)
- [generics](../../functions/generics/skill.md)
