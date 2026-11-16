---
name: goose-run-build-backends
title: run vs build — the two backends and when each works
category: tooling
tags: [goose, jit, tcc, tinycc, build, debug, backend, in-process]
triggers: [goose run vs build, tinycc, jit backend, goose --debug, backend limits]
description: >
  goose run compiles to C and runs it in-process via TinyCC (no files, no
  C compiler needed beyond libc headers); goose build uses a real C
  compiler (optimized -O2 C, executables beside the source). What each can
  and cannot do: threads need a build, extern symbols resolve differently,
  timing needs a build, --debug for checked builds.
source: https://github.com/aardappel/goose/blob/main/docs/command_line.md#2-running-a-program
---

# run vs build: the two backends

## `goose run` — the in-process backend

Compiles the program to C, compiles that C **in memory** with the TinyCC
compiler built into `goose`, and runs it in the compiler's own process. No
build step, no files written.

```bash
goose hello.goose
goose run hello.goose -- input.txt --verbose
```

Needs nothing extra on Windows; on Linux the C library headers
(`libc6-dev`/`glibc-devel`); on macOS the Xcode CLT SDK.

What a run **cannot** do (a build can):

- **Threads** — TinyCC cannot place thread-local storage; `goose run`
  refuses threaded programs.
- **Optimization** — TinyCC does not optimize: a run is for trying things
  out; timing is done on a build.
- **`extern fn` resolution** — only C99 `<math.h>` functions in both widths
  (all but `nexttoward`). Anything else needs a build, the program's own C,
  or an `extern "lib" { ... }` block.
- A run keeps data **relative to the root file**: `resource_dir()` is the
  root file's directory, where a built program's is the executable's.
  Relative paths are relative to the working directory either way.

A program whose packages have C of their own still runs: that C is built
into a shared library once and loaded into the run.

## `goose build` — the native backend

```bash
goose build hello.goose && ./hello
goose build -o out/hello hello.goose     # executable elsewhere
goose build --debug hello.goose          # checked, with debug information
goose build --shared hello.goose         # libhello.so / hello.dll
goose build --dry-run hello.goose        # print the commands, run none
```

Writes the program's C, compiles it with a C compiler on the machine
(`--cc`, `GOOSE_CC`, `~/.goose/config`'s `cc = "clang";`, or first found —
`goose toolchain` shows which and why), and links with the runtime and the
native parts of the packages used. The executable goes beside the root file
unless `-o` or the sidecar's `output = "name";` says otherwise.

The C is **always compiled optimized** (`-O2`/`/O2`) whatever Goose `-O`
level was used — except under `--debug`, which turns on the overflow, range
and tag checks (`GS_DEBUG=1`) and builds the C without optimization with
debug information (`-O0 -g`). Debug builds are a tool for finding bugs:
each check failure names its source line, and an overflow or cast names the
values involved.

Everything a build makes goes under `GOOSE_HOME`: the program's C, objects
and logs in `build/prog/<name>-<hash>/`; the runtime object compiled once
per compiler and configuration into `build/runtime/`. A compile or link
failure shows the command, its output, and where the log is.

## Debugging output

A run prints only what the program prints; a check that succeeds prints
nothing; a build prints only `built <exe>`. `goose run` exits with the
program's exit code.

## Choosing

| Situation | Use |
|---|---|
| Trying code, quick iteration, scripts | `goose run` |
| CI typecheck | `goose check` (or `--diagnostics json`) |
| Threads, performance, `extern` beyond math.h | `goose build` |
| Hunting overflow/range/tag bugs | `goose build --debug` |
| Linking into a C host / hot reload | `goose build --shared --header` |
| Your own build system | `goose -o prog.c` + `--emit-runtime`, or `--dry-run` |

## Related skills

- [cli-reference](cli-reference/skill.md)
- [export-to-c](../interop/export-to-c/skill.md)
- [debugging](debugging/skill.md)
