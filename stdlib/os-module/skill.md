---
name: goose-os-module
title: os module — files, args, env, time, stdin, resources
category: stdlib
tags: [goose, os, files, read_file, write_file, args, env, time, resource_dir, stdin]
triggers: [goose file io, goose args, goose environment variable, goose time, resource dir, read stdin]
description: >
  A small interface to the runtime's OS layer: whole-file read/write
  (including atomic write), directory listing, stdin/stdout, args()/env(),
  time/clock/sleep, random_seed, trim_memory, and resource_dir() for
  application assets. Paths are UTF-8; no subprocesses or networking in v1.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#os
---

# The `os` module

```goose
import os;

fn read_file(path: const u8[:], out: u8[>..]&) -> bool      // appends the whole file; nothing on failure
fn write_file(path, data) -> bool
fn write_file_atomic(path, data) -> bool                     // old file or all of the new
fn append_file(path, data) -> bool
fn file_exists(path) -> bool      fn delete_file(path) -> bool
fn rename_file(from, to) -> bool                             // atomic replace within one volume
fn make_dir(path) -> bool         fn is_dir(path) -> bool    fn delete_dir(path) -> bool  // empty
fn list_dir(path, out: u8[>..]&) -> bool                     // appends each name + '\n', sorted, no . / ..
fn read_line(out) -> bool                                    // stdin, newline stripped; false at end
fn read_stdin(out)              fn read_stdin(out, max) -> bool
fn write_stdout(s)  fn write_stderr(s)  fn flush_stdout()
fn arg_count() -> i64   fn arg(i, out)   fn args() -> u8[:][>..]   // args()[0] is the program
fn env(name, out) -> bool
fn trim_memory() -> bool                                     // return unused data-stack pages to the OS
fn resource_dir() -> u8[>..]                                 // absolute dir, trailing separator
fn time() -> f64            fn clock() -> f64                // epoch / monotonic
fn time_ns() -> i64         fn clock_ns() -> i64
fn sleep(seconds: f64)      fn sleep_ms(ms: i64)
fn random_seed() -> u64                                      // entropy, for rng
```

## Program arguments

```goose
let a = args();                      // a[0] is the program (root file's path as given)
guard a.len > 1 else { print("usage: hello file..."); exit(2); }
for i in 1..a.len { print(i, ": ", a[i]); }
```

## Files: whole-file style

There are no open file handles to close — calls are whole-file operations.
`write_file_atomic` is for saves that must never be seen half-written
(writes to a `.tmp` file, flushes, renames over `path`, keeping permissions
on POSIX). `rename_file` does one-step replacement for files the program
wrote, within one volume. On Windows, replacement waits up to ~1s for a
file another program holds open.

```goose
var wad: u8[>..] = [];
if !read_file(str(resource_dir(), "data/map.wad"), wad) { abort("cannot read map.wad"); }

var names: u8[>..] = [];
if list_dir("saves", names) {
    each_split(names, '\n') { if it.len > 0 { load_save(str("saves/", it)); } };
}
if !write_file_atomic("saves/slot1.sav", image) { write_stderr("not saved\n"); }
```

## Application resources

Ordinary relative paths resolve against the **process working directory**
(Goose does not change it). For data shipped with the application,
`resource_dir()` returns an absolute UTF-8 directory with a trailing
separator: in a JIT run it is the entry source's directory; in a built
program, the running executable's directory (in a macOS `.app` bundle,
`Contents/Resources/`). Deploy data relative to that root; moving the
executable and its data together preserves lookup. Shared libraries named
with a path separator in `extern "..." { }` (`"bin/steam_api64"`) open
relative to the same root. The string form aborts on failure; the builder
overload returns false.

## Time

`time()` is wall-clock seconds since the epoch; `clock()` is monotonic and
high resolution — use `clock()` for benchmarking.

## Memory trimming

`trim_memory()` asks the OS to discard complete pages beyond every current
data-stack top in the calling thread program (globals and ended scratch
included). Live values, borrowed views, limited-array capacity and virtual
reservations are preserved; growth works normally (trimmed pages fault back
in). Explicit: use after a large workload (e.g. replacing a document), not
after every temporary. On Windows a stale view reading a discarded page
commits it again.

## Notes

- Paths are UTF-8 everywhere (converted to UTF-16 on Windows); refused if
  longer than 4095 bytes, containing NUL, or (Windows) invalid UTF-8.
- `read_stdin` appends bytes unchanged on all platforms (no CRLF
  translation, no Ctrl-Z stop); `read_line` strips `\r` before the newline.
- Environment values and arguments are UTF-8 (Windows arguments pass
  through the ANSI code page).
- **No subprocesses or networking in v1** — arrive as `extern fn`s when a
  program needs them.
- `exit(code)` and `abort(msg)` are builtins, so the checker knows they
  diverge.

## Related skills

- [error-idioms](../errors/error-idioms/skill.md)
- [serialization](../data-modeling/serialization/skill.md)
- [extern-fn](../interop/extern-fn/skill.md)
