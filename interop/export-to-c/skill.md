---
name: goose-export-to-c
title: export fn — Goose as a library called from C
category: interop
tags: [goose, export fn, goose_init, header, shared library, embed goose]
triggers: [goose export fn, call goose from c, goose_init, goose header, shared library embed]
description: >
  export fn makes a Goose function callable from C; the compiler emits a C
  wrapper. Build with --standalone --header hello.h -o hello.c (or --shared
  for a .so/.dll); the host calls goose_init once (which runs Goose main),
  then the exports; goose_exit releases and allows reload. Slice results
  come back as pointer + length out-parameter.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#export-fn
---

# `export fn` — being called from C

`export fn` is the reverse of `extern fn`: the compiler emits a C wrapper
under the function's name (or the given string). The function must be
top-level and non-generic, with explicit parameter types and at most one
typed result. Defaults, `return … from`, pool parameters and builders are
not supported at the boundary. Scalars and flat fixed-size values return by
value; a `const u8[:]` result comes back as a pointer plus an extra
`int64_t *` length parameter (bytes are **not** NUL-terminated).

```goose
var calls: i32 = 0;

export fn hello_world() -> const u8[:] {
    calls += 1;
    return "Hello, world!"
}

export "c_add" fn add(a: i32, b: i32) -> i32 { a + b }

fn main() {
    calls = 100;    // runs once, inside goose_init
}
```

## Building for a host

```sh
goose --standalone --header hello.h -o hello.c hello.goose
cc hello.c app.c -o app
```

`--header` implies a C host build: the C has **no `main` of its own** (and
there must be at least one `export fn`). `goose build --shared --header
hello.h hello.goose` builds the same thing straight into a shared library
(`libhello.so`, `libhello.dylib`, `hello.dll`).

## The host protocol

```c
#include "hello.h"
#include <stdio.h>

int main(int argc, char **argv) {
    int64_t length;
    goose_init(argc, argv);                     // once, before anything else
    const uint8_t *message = hello_world(&length);
    fwrite(message, 1, (size_t)length, stdout); // not NUL-terminated
    putchar('\n');
    printf("%d\n", c_add(20, 22));
    goose_exit();
    return 0;
}
```

- `goose_init` sets up the runtime and the globals, then runs the Goose
  `fn main()` — `main` is the place for a library's own start-up code (a
  library with nothing to start up may leave `main` out).
- `goose_exit()` releases what the start set up, and `goose_init` can start
  it again — how a host **reloads a library it has rebuilt**. It is a
  runtime error while a worker of the program still runs.
- Exported functions are **not thread-safe**: call them from the thread that
  called `goose_init`.
- Signatures that may cross are the extern-fn shapes, with a few more
  limits (spec §7.11).

## When to use

Embedding Goose into an existing C/C++ application; shipping a Goose
implementation behind a stable C ABI; hot-reloading game logic. For calling
C from Goose, see extern-fn; for libraries loaded by name at runtime, see
extern-lib-and-ffi.

## Related skills

- [extern-fn](extern-fn/skill.md)
- [extern-lib-and-ffi](extern-lib-and-ffi/skill.md)
- [run-build-backends](../tooling/run-build-backends/skill.md)
