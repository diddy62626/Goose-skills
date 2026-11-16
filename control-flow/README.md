# control-flow

**Core concepts** — Expressions (if/match/block), guard, defer, loops, and const for/if reflection.

4 skills. Load any of them as `control-flow/<skill>/skill.md`.

- [`goose-const-for-reflection`](control-flow/const-for-reflection/skill.md) — const for / const if — specialization-time reflection: > const for expands a fragment once per entry of a compile-time sequence: a struct's fields (fields<T>), an enum's variants (variants<E>), a list of types (types<...>), or a list of constants. const if chooses fragments at specialization time with has_default<A> and same<T1,T2>. Goose's reflection, replacing derive macros.
- [`goose-expressions-if-match`](control-flow/expressions-if-match/skill.md) — Expressions — if, match, block as values; branch type joining: > if, match, block and bare {} are expressions; a block's value is its trailing expression. Used as values, ifs need else, match needs full coverage. Literal branches adapt; slices join as slices; statements vs expressions and semicolon rules; functions that never return.
- [`goose-guard-block-defer`](control-flow/guard-block-defer/skill.md) — guard, block-with-break, and defer: > guard states a condition the rest of its block depends on (and narrows optionals). block { } gives break a value target without a loop. defer runs on every exit of its block — end, break, continue, return, return from — latest first, once per iteration in loops, with its three restrictions.
- [`goose-loops`](control-flow/loops/skill.md) — Loops — for-in, while, loop, for &x mutation, index binding: > for x in range_or_array with the full form catalog: typed loop variables, half-open ranges, by-value vs for-&x mutation, the x,i index pair, in-place traversal semantics (pushes extend the walk; no shrink inside), while and loop, no labels.
