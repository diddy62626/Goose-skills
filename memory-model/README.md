# memory-model

**Core concepts** — The data-stack memory model: ownership, size classes, copies, in-place construction.

4 skills. Load any of them as `memory-model/<skill>/skill.md`.

- [`goose-copies-and-aliasing`](copies-and-aliasing/skill.md) — Copies are visible — copy(), binding by reference, call semantics: > Fixed-size values copy silently like C; non-fixed values NEVER copy implicitly. `let b = a;` binds b to a by reference for growable arrays. `copy(a)` is the explicit O(n) copy. How this works for variables, fields, elements, and call arguments, plus the generic-code `.= for values` rule.
- [`goose-data-stacks-and-ownership`](data-stacks-and-ownership/skill.md) — The memory model — data stacks, ownership, scope exit: > Goose's entire memory system: N compiler-assigned data stacks with bump pointers, every dynamic value owned by a variable, freed en masse at scope exit, at most one resizable value live per stack, growth by pointer bump, nothing ever moves. Explains the rules and how to structure code around them.
- [`goose-in-place-construction`](in-place-construction/skill.md) — In-place construction — results are built at their destination: > The copy-free construction guarantee: a constructed non-fixed value is always built in its final location, through any depth of calls and through whichever branch an if/match takes. Returning a built-up value costs the same as an out-parameter; str(...) writes directly into the element being pushed. Plus the construction-in-progress restriction.
- [`goose-size-classes-and-placement`](size-classes-and-placement/skill.md) — Size classes — fixed, variable, resizable, root — and what may contain what: > Every Goose type is fixed, variable, resizable, or root. This decides where a value may live: which fields, which array element slots, which parameter and result positions. The complete placement rules with the struct-tail and element restrictions.
