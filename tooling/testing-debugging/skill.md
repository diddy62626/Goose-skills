---
name: goose-testing-debugging
title: Testing and debugging Goose programs
category: tooling
tags: [goose, testing, debugging, assert, debug build, aborts, expected output]
triggers: [goose testing, debug goose program, assert testing, GS_DEBUG, abort message, multi-test]
description: >
  How to test and debug Goose code: assert-based tests with panic-style
  aborts, --debug builds that name failing checks and values, clean runtime
  abort messages with source lines, GS_DEBUG/GS_STACK_STATS knobs, --multi-
  test for batch-checking files, and the repo's own test suite conventions
  (expected output, // error: markers).
source: https://github.com/aardappel/goose/blob/main/docs/command_line.md
---

# Testing and debugging Goose programs

## Assertion-style tests

There is no unit-test framework in the compiler; the idiomatic pattern is
`assert` plus printed expectations:

```goose
fn test_parse() {
    let n, ok = parse_int("42");
    assert(ok && n == 42);
    let f, fok = parse_flt("2.5");
    assert(fok && f == 2.5);
    print("parse tests ok");
}
```

A failed `assert` aborts cleanly with the message and the failing source
line — a test run is either "all asserts passed" or stops at the first
failure with its location. Wrap groups in functions and call them from
`main` (or keep each test file a root file with its own `main`).

## Debug builds: `--debug`

`goose build --debug` (or `--debug` on run/check) turns on the overflow,
range and tag checks (`GS_DEBUG=1`) and builds the C unoptimized with debug
information. Each failure names its source line; an overflow or a bad cast
**names the values involved**:

- signed overflow aborts (release wraps);
- `as` casts are range-checked (a changed value aborts, naming values);
- enum tag checks catch corrupt tags.

Use it to hunt arithmetic bugs; use release builds for timing.

## Runtime aborts are clean

Division by zero, `i64.min / -1`, capacity overflow, shrink below empty, a
relative-reference offset not fitting its width, a failed bounds check,
`assert`, and data-stack exhaustion all abort with a message and the line
of the check that failed. A check inside a standard library function names
the **line of the program's code that called the library**. There are no
segfaults to decipher.

## Program-visible knobs

- `-DGS_STACK_STATS=1` — each thread program reports its data stack use on
  stderr at exit.
- `--stacks` — how many data stacks the program and its workers use and the
  cap on `hardware_threads()`.
- `-DGS_STACK_RESERVE=…` / `--stack-reserve` — shrink the per-stack
  reservation to stress-test lifetime handling (a runaway stack aborts
  naming `--stack-reserve`).
- `-DGS_MAX_STACKS=1` — force the minimal stack count to catch a program
  accidentally needing more.
- Contained workers (`thread_spawn_contained`) route a failing worker's
  abort message to `thread_wait(id, msg)` instead of stderr — useful for
  fault-injection tests.

## Batch checking many files

```bash
goose check --diagnostics json a.goose b.goose   # per-file errors as JSON (separate runs)
goose --multi-test a.goose b.goose c.goose       # compiles each in one process,
                                                 # ends each with ==== exit <code> <file>
```

`--multi-test` gives each file exactly the exit code and output a separate
run would, in a fraction of the time — handy for lint-style sweeps over a
directory of samples or tests.

## The compiler's own suite (for contributors)

`python test/run_tests.py --profile baseline --cc native` after a build.
Fixtures live in `test/` by category (syntax, typing, lifetimes, codegen,
optimizer, runtime, storage, threads, stdlib, packages, errors, errors_tc,
lsp, …). Expectations: every runnable fixture needs `expected/<name>.out`;
expected aborts add `.aborts` and `.stderr` substrings; error fixtures
declare `// error: <substring>` and warning fixtures `// warning:
<substring>` lines; `// runtime-define: GS_MAX_STACKS=1` style first lines
pass `-D` to every compile. See docs/testing.md for the full contract.

## Related skills

- [run-build-backends](run-build-backends/skill.md)
- [error-idioms](../errors/error-idioms/skill.md)
- [cli-reference](cli-reference/skill.md)
