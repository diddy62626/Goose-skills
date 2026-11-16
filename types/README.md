# types

**Core concepts** — Scalars and arithmetic, casts, structs, enums/ADTs, aliases and constants.

5 skills. Load any of them as `types/<skill>/skill.md`.

- [`goose-aliases-and-named-constants`](aliases-and-named-constants/skill.md) — Type aliases and named constants: > `type Name = T;` is the same type, not a new one. A `let`/`const` global with a constant initializer is a named constant that adapts like a literal: usable as any integer type it fits, an array size, a fill count, or a match arm.
- [`goose-casts-and-conversions`](casts-and-conversions/skill.md) — Casts and conversions — implicit widening, `as`, `as!`: > What converts implicitly (widening, int→float), what needs `as` (debug range-checked) or `as!` (never checked), float↔integer conversion rules, redundant-cast warnings, and comparisons with u64.
- [`goose-enums-adts`](enums-adts/skill.md) — Enums (ADTs) — two storage modes, match, case functions: > Enums are the only dynamic polymorphism in Goose (no inheritance, traits or vtables). Every ADT can be stored fixed-mode (indexable, replaceable) or variable-mode (packed, pointable-into). match arms, &-binders, integer matches, and case functions (one overload per variant — the virtual-call idiom without a vtable).
- [`goose-scalars-and-arithmetic`](scalars-and-arithmetic/skill.md) — Scalars, numeric literals, and the arithmetic rules: > Integer and float types, width rules ("arithmetic happens at the operands' own width"), literal adaptation, the unification rules for mixed operands, Euclidean %, truncating /, overflow behavior per build, bitwise operators, memberwise arithmetic on structs, and the varint storage type.
- [`goose-structs`](structs/skill.md) — Structs — literals, field defaults, qualifiers, generic structs: > Struct declarations introduce nominal types; fields are packed with no padding. Literals by name or position, `..` for defaults, field defaults evaluated per construction, let/const/pad field qualifiers, variable-size and varint fields, the one resizable tail, root structs, generic structs, equality and memberwise arithmetic.
