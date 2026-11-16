---
name: goose-install-and-toolchain
title: Installing Goose, the toolchain, and running your first program
category: getting-started
tags: [goose, install, toolchain, cli, build]
triggers: [install goose, goose compiler, goose build, goose run, goose toolchain, GOOSE_HOME]
description: >
  Where to get the Goose compiler (releases page or build from source), what
  it needs (git, cmake, a C compiler), the one-executable design
  (compiler + runner + build tool + fetcher + formatter + LSP), GOOSE_HOME,
  and the first commands to try.
source: https://github.com/aardappel/goose#install-a-release
---

# Installing Goose and running your first program

## Get the compiler

- **Releases**: prebuilt compilers for Windows, Linux and macOS on
  `https://github.com/aardappel/goose/releases`.
- **Build from source**: clone `aardappel/goose`, it is a CMake C++ project
  (`cmake -S . -B build && cmake --build build`).

## What the machine needs

The compiler compiles Goose **to C** and builds that C:

| Mode | Needs |
|---|---|
| `goose run file.goose` (in-process TinyCC JIT) | Nothing extra on Windows; on Linux the C library headers (`libc6-dev`/`glibc-devel`); on macOS the Xcode command line tools SDK |
| `goose build file.goose` (executable) | Any C compiler found on the machine: `cc`/`gcc`/`clang` (or MSVC `cl` on Windows) |
| Threads | A real build (`goose build`) — TinyCC cannot place thread-local storage |
| Packages with C | `git`, and `cmake` if the package is CMake-built |

## One executable does everything

`goose` is compiler, runner, build tool, package fetcher, formatter and
language server in one binary:

```
goose hello.goose            # compile (in-process) and run; no files written
goose run hello.goose -- a b # same; args after -- go to the program
goose build hello.goose      # executable beside the source (hello / hello.exe)
goose check hello.goose      # parse + typecheck only; prints nothing when correct
goose fetch hello.goose      # fetch declared packages for offline use
goose format src/            # rewrite files in the standard layout
goose lsp                    # language server on stdin/stdout
goose toolchain              # report the C compiler, cmake, git, GOOSE_HOME
goose version                # compiler version + runtime hash
```

`goose build` options worth knowing: `-o out`, `--debug` (overflow/range/tag
checks + unoptimized C + debug info), `--shared` (shared library), `--dry-run`
(print the commands), `--cc clang`, `-v`. Run/build/check share `-O0/-O1/-O2`,
`--debug`, `-D NAME=VALUE`, `--stats`, `--bce-lines`, `--diagnostics json`.

## Hello world

```goose
fn main() {
    print("hello, goose");
}
```

```bash
goose hello.goose          # runs immediately, JIT
goose build hello.goose && ./hello
```

`print` takes any number of arguments of any type and writes one line in a
single write (thread output never interleaves). Arrays print as `[1, 2, 3]`,
structs as `Point { 1, 2 }`.

## Environment and files

| Variable | Meaning |
|---|---|
| `GOOSE_HOME` | Where fetched packages, build intermediates, config go. Default `~/.goose`. Deleting it loses nothing a build cannot recreate |
| `GOOSE_CC` | C compiler when `--cc` is not given |
| `GOOSE_STDLIB` | Standard library directory override |
| `NO_COLOR` / `CLICOLOR_FORCE` | Color control |

`~/.goose/config` holds per-machine settings, written like a Goose sidecar:
`cc = "clang";`. A program's sidecar `foo.build` (beside `foo.goose`) names
its own C and libraries.

## Exit codes

`run` gives the program's exit code (0 on return, `exit(code)`, 1 on abort or
compile error); `build`/`check`/`fetch` are 0/1; `format --check` is 0/1/2.

## Common mistakes

- Running a threaded program with `goose run` → refused ("compile with -o" /
  `goose build`).
- Expecting `extern fn` symbols beyond C99 `<math.h>` to link under `goose
  run` — anything else needs a build, the program's own C, or an
  `extern "lib" { ... }` block.
- `goose biuld x.goose` — a mistyped command is detected and suggests `build`.

## Related skills

- [cli-reference](../../tooling/cli-reference/skill.md) — every command and option
- [run-build-backends](../../tooling/run-build-backends/skill.md) — JIT vs build, debug builds
- [packages](../../tooling/packages/skill.md) — using git packages
