---
name: goose-cli-reference
title: CLI reference — every goose command and option
category: tooling
tags: [goose, cli, commands, options, goose build, goose check, diagnostics, env vars]
triggers: [goose command line, goose cli, goose options, --diagnostics json, --stats, goose completion]
description: >
  The complete goose command surface: run/build/check/fetch/format/lsp/
  toolchain/version/completion/help; shared options (-O0..-O2, --debug, -D,
  --stack-reserve, --stacks, --bce-lines, --specs, --stats, --diagnostics
  json, --offline); runtime -D macros; environment variables; exit codes;
  and the compiler-internals flags.
source: https://github.com/aardappel/goose/blob/main/docs/command_line.md
---

# CLI reference

One executable is compiler, runner, build tool, package fetcher, formatter
and language server. The command comes first; `goose file.goose` runs it.
Options may come before or after the file. Everything after `--` belongs to
the program.

## Commands

| Command | What it does |
|---|---|
| `goose run file.goose` / `goose file.goose` | compile to C, compile that in memory with TinyCC, run in-process |
| `goose build file.goose` | executable (or `--shared` library) with a C compiler |
| `goose check file.goose` | parse + typecheck only; prints nothing when correct |
| `goose fetch file.goose` | fetch declared packages (`--update` re-fetches tags/branches) |
| `goose format file.goose\|dir…` | rewrite files in the standard layout (`--check` lists non-formatted) |
| `goose lsp` | Language Server Protocol on stdin/stdout (`--probe` prints protocol version) |
| `goose toolchain` | report C compiler, cmake, git, `GOOSE_HOME` (`--json`, `--refresh`) |
| `goose version` | compiler version + runtime hash |
| `goose completion bash\|zsh\|fish\|powershell` | print a completion script |
| `goose help [command]` | help (`--man` for a man page; `goose help c` for writing C; `goose help internals`) |

## Shared options (run, build, check, `-o`)

| Option | What it does |
|---|---|
| `-O0` / `-O1` / `-O2` | how much the Goose compiler inlines (default -O2); constant folding at every level; the C compiler optimizes at every level |
| `--debug` | overflow/range/tag checks (`GS_DEBUG=1`); a build also gets unoptimized C + debug info |
| `-D NAME[=VALUE]` | define a C macro in the generated program |
| `--stack-reserve size` | address space per data stack, 64K..256T (default 2G) |
| `--stacks` | report the program's data stacks and the thread cap |
| `--bce-lines` | per-line report of removed/kept bounds checks |
| `--no-bce` | keep every bounds check |
| `--specs` | print every specialization after optimization, as source |
| `--stats` | a few lines about what the compile did |
| `--color auto\|always\|never` / `--diagnostics text\|json` | output control |
| `--offline` / `--update` | package fetching control |
| `--stdlib dir` / `--cc name\|path` / `-v` | environment overrides |

## Build-only options

`-o exe` (executable elsewhere), `--debug`, `--cc`, `--standalone` (compile
the runtime in), `--shared` (a self-contained shared library; implies
`--standalone`; add `--header out.h` for the host API), `-v`, `--dry-run`
(print the commands, run none — useful for Makefiles).

## Writing the C yourself

`goose -o tour.c tour.goose` writes the program's C; `--emit-runtime
goose_runtime.c` writes the runtime's C (compile once per compiler+config,
link with every program; `goose version`'s runtime hash must match).
`--standalone` writes C that holds the whole runtime and builds on its own:
`cc -O2 tour.c -o tour -lm -pthread` (add `-ldl` before glibc 2.34). For
`audio`/`gfx`/`physics`/`ui`, link the response file: `cc cube.c
goose_runtime.o -o cube @$(goose --gfx-link cc) -lm -pthread`.

## Runtime `-D` macros

| Macro | Default | Sets |
|---|---|---|
| `GS_DEBUG` | 0 | the overflow/range/tag checks (`--debug` sets 1) |
| `GS_STACK_STATS` | 0 | report data stack use on stderr at exit |
| `GS_STACK_RESERVE` | 2 GB | per-stack address space |
| `GS_THREAD_STACK` | 8 MB | native stack per worker thread |
| `GS_STACK_GAP` | 1 MB | unmapped gap after each stack (runaway growth aborts) |
| `GS_STACK_BUDGET` | 32 TB | address space all thread programs share; caps `hardware_threads()` |
| `GS_MAX_STACKS` | 1024 | most data stacks one thread program may use (a program needing more fails to compile) |
| `GS_SIMD` | 2 | highest CPU level for `simd fn` versions (clang, x86-64) |

## Environment and files

`GOOSE_HOME` (default `~/.goose`: config, `pkg/`, `build/`), `GOOSE_CC`,
`GOOSE_STDLIB`, `GOOSE_PROGRESS` (plain/live), `NO_COLOR`,
`CLICOLOR_FORCE`, `GOOSE_TCCLIB`. `~/.goose/config` holds `cc = "clang";`.
A program's sidecar `foo.build` sits beside `foo.goose`.

## Output formats

Errors and warnings go to stderr with the source line and a `^~~~` caret:

```
x.goose:5: warning: redundant `as i64`: an i32 converts to i64 implicitly here
    xs.push(((r as i64) << 32) | (c as i64));
                                  ^~~~~~~~
```

`--diagnostics json` prints a JSON array on stdout (severity, message, file,
1-based line/byte col, endLine/endCol, related lines, full text); what goose
itself would print on stdout goes to stderr instead. Progress notes are
one-liners when stderr is not a terminal.

## Exit codes

`run`: the program's (0 / `exit(code)` / 1 on abort; 1 if it does not
compile). `build`/`check`/`fetch`/`-o`: 0 or 1. `format`: 0 formatted, 1
`--check` found one, 2 parse/read/write/option error. `toolchain`: 1 when
there is no C compiler.

## Compiler-internals flags (`goose help internals`)

`--tokens`, `--parse`, `--check`, `--dump` (parsed program as source),
`--dump-docs` (documentation comments of declarations), `--dump-file`,
`--roundtrip`, `--multi-test file…` (compile each file as its own run,
`==== goose --multi-test: exit <code> <file>` markers), `--bce-test`
(verify `// bce:elide`/`// bce:keep` annotations), `--gen-runtime-header`,
`--compile-shader`.

## Related skills

- [run-build-backends](../run-build-backends/skill.md)
- [diagnostics](../diagnostics/skill.md)
- [editors-lsp-format](../editors-lsp-format/skill.md)
