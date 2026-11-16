---
name: goose-diagnostics
title: Compiler diagnostics — reading errors, JSON output, stats
category: tooling
tags: [goose, errors, warnings, diagnostics json, instantiation chain, stats]
triggers: [goose error format, diagnostics json, instantiation chain, compiler output, bce lines]
description: >
  How the compiler reports problems: stderr with source line and caret,
  both-ends error messages (reference + binding site, holder + store site,
  instantiation + call chain), --diagnostics json shape for tools, --stats
  and --bce-lines output, color control, and where progress notes go.
source: https://github.com/aardappel/goose/blob/main/docs/command_line.md#6-what-goose-prints
---

# Compiler diagnostics

## Text output

Errors and warnings go to stderr, each with the line of source it is about
and a `^~~~` under the code where the compiler knows the column:

```
x.goose:5: warning: redundant `as i64`: an i32 converts to i64 implicitly here
    xs.push(((r as i64) << 32) | (c as i64));
                                  ^~~~~~~~
```

A run prints only what the program prints. A check that succeeds prints
nothing; a build prints only the `built` line.

## Read the whole error

Goose errors name **both ends** of the problem:

- a lifetime error: the reference *and* where it was bound;
- a shrink error: the holder *and* the line that stored into it;
- a generic error: the instantiation *and* the compile-time call chain
  (`in fn … instantiated from file:line` per frame);
- an overload error: the candidates *and* why each failed.

Use those locations to decide which binding, store, or scope to change.
This is the single most useful debugging habit in Goose.

## JSON output for tools

`--diagnostics json` prints errors and warnings on stdout as a JSON array
(after anything the program printed); what goose itself would print on
stdout (stats, `built`) moves to stderr:

```json
[
  {"severity": "error", "message": "expected a value of type i32, got const u8[:]",
   "file": "bad.goose", "line": 1, "col": 26, "endLine": 1, "endCol": 29,
   "related": [{"message": "in fn main() instantiated from bad.goose:1",
                "file": "bad.goose", "line": 1, "col": 1}],
   "text": "bad.goose:1: error: expected a value of type i32, got const u8[:]\n..."}
]
```

Each object: `severity` (`error`|`warning`), `message`, and where known the
`file` as the compiler names it, 1-based `line`, byte `col`, `endLine` and
`endCol` one past the last byte; `related` holds the follow-on lines in the
same form; `text` is the whole message as the terminal would show it. An
error not about the program (missing C compiler) has message and text only.
The exit code is unchanged.

## What the compile did

`--stats` prints a few lines on stdout (stderr for a run): declarations
parsed, specializations and optimizations, bounds checks removed, files
written. A check that gets past resolution prints `typechecked ok`.

`--bce-lines` reports, per source line, how many bounds checks the compiler
removed and how many it kept — use it when a kernel is slower than expected
(see bounds-check-elimination). `--specs` prints every specialization of
each function after optimization, as source.

## Color and progress

Color on a terminal; `--color never` or non-empty `NO_COLOR` turns it off;
`--color always` or `CLICOLOR_FORCE≠0` forces it on. Long work (fetching,
building package C, compiling the runtime) prints a one-line note on
stderr; the live progress line appears only on terminals (plain when
stderr is a file/pipe/CI — which includes agents running commands;
`GOOSE_PROGRESS=plain|live` overrides).

## Warnings worth knowing

- **Redundant casts**: ``redundant `as i64`: a u8 converts to i64
  implicitly here`` — the cast the rules make unnecessary.
- **Second returned local**: stops the first from being allocated at the
  destination (see in-place-construction).
- `--no-bce` warns that a program's `checkfree` functions are no longer
  verified.
- Writing `&` where a reference destination already binds: warning.

## Related skills

- [cli-reference](cli-reference/skill.md)
- [reading-errors](../idioms/reading-errors/skill.md)
- [bounds-check-elimination](../performance/bounds-check-elimination/skill.md)
