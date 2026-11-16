# errors

**Core concepts** — Error idioms, return … from, the exception module, asserts and aborts.

3 skills. Load any of them as `errors/<skill>/skill.md`.

- [`goose-error-idioms`](errors/error-idioms/skill.md) — Error idioms — trailing bool, T?, asserts, aborts, exits: > No Result type, no exceptions, no required convention: the application chooses. Two outcomes = trailing bool (-> T, bool); found-or-not = T? (a reference into the input, copies nothing); bugs/invariants = assert/ abort/exit, which never return and are uncatchable.
- [`goose-exception-module`](errors/exception-module/skill.md) — The exception module — typed try/throw: > exception::try<E>(body) { error => ... } and exception::throw(value) package the return-from machinery as typed exceptions. E is fixed at try (u8[] for owned text, an int, or your own struct/enum). Ownership rules, recursion restrictions, and the defer-based finally.
- [`goose-return-from`](errors/return-from/skill.md) — `return … from` — lightweight checked exceptions without unwinding: > return E from f returns E as the result of the innermost active call of f, unwinding every frame in between — while every function in between keeps its plain signature. Statically checked (no uncaught case), implemented as a hidden discriminant (no unwind tables), and the error value is built directly at the target's destination.
