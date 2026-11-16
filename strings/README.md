# strings

**Core concepts** — Strings as byte arrays, formatting text, zero-copy parsing and manipulation.

3 skills. Load any of them as `strings/<skill>/skill.md`.

- [`goose-formatting-text`](formatting-text/skill.md) — Formatting text — print/format/str and the format overload: > The three rendering builtins (print, format, str) take any number of arguments of any type with no format string. A fn format(out: u8[>..]&, v: T) overload defines a type's text form once. Controlled output via std's format_int/format_uint/format_flt, plus float printing rules.
- [`goose-parsing-text`](parsing-text/skill.md) — Parsing and manipulating text — split/trim/find/parse_int, zero-copy: > std's text functions operate on u8[:] views: nothing is copied. split returns slices into the input, split_once is strings.Cut, words splits at whitespace, parse_int/parse_uint/parse_flt return (value, ok), compare is bytewise, each_utf8/push_utf8 handle code points, ascii:: character classes.
- [`goose-strings-as-byte-arrays`](strings-as-byte-arrays/skill.md) — Strings are byte arrays — four roles, no string type: > There is no string type: a string is any array-family type with u8 elements. u8[>..] builder, u8[] finished, u8[..16] inline, u8[:] view. String literals are const u8[:] static data. Raw """ strings, UTF-8 as a library convention, and the inline-string-in-struct pattern.
