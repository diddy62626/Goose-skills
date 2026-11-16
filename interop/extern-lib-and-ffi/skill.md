---
name: goose-extern-lib-and-ffi
title: extern "lib" blocks and the ffi module — binding shared libraries by name
category: interop
tags: [goose, extern lib, shared library, dylib, SDL2, ffi, cstr, opaque handles]
triggers: [goose extern lib, bind dll, bind so, SDL2 goose, ffi cstr, opaque handle]
description: >
  extern "SDL2" { fn SDL_Init(flags: u32) -> i32; } opens a shared library
  at program start and binds the members called — no C to compile. Handles
  are u64, pointers to Goose-owned data are T&, structs cross packed, and
  ffi::cstr/bytes copy library-owned memory into Goose storage. Sidecar
  dylibs maps platform filenames.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#extern-lib-blocks
---

# `extern "lib" { … }` and the `ffi` module

An `extern` block names a shared library and declares functions bound from
it **when the program starts** — no C to compile or link. The program opens
the library and looks the symbols up; a missing library or symbol is a
runtime error naming what was tried. Members are `extern fn`s written
without the `extern` keyword; a string after `=` is the C symbol.

```goose
extern "SDL2" {
    fn SDL_Init(flags: u32) -> i32;
    fn SDL_Quit();
    fn SDL_GetTicks64() -> u64;
    fn delay(ms: u32) = "SDL_Delay";
}
```

Only a member a call reaches is bound; a library none of whose members is
called is never opened — a block may declare an interface wider than any one
program uses. A program whose only C is such libraries runs under the
in-process backend with **no C toolchain installed**.

## Naming

`"SDL2"` opens `SDL2.dll` / `libSDL2.dylib` / `libSDL2.so` by the loader's
search. `"bin/steam_api64"` opens that file relative to `resource_dir()` —
how a program ships a library beside itself. A dotted name
(`"libSDL2-2.0.so.0"`) is taken as written. Per-platform names go in the
sidecar:

```goose
// myprog.build
dylibs_linux = ["SDL2=libSDL2-2.0.so.0"];
dylibs_windows = ["SDL2=bin/SDL2.dll"];
```

A package's sidecar maps its own names as defaults; the program's sidecar
overrides them per platform. Mapping a name no block declares is an error.

## Declaring the C prototypes in Goose terms

- **Scalars** are their C types: `i32` is `int`, `u32` `unsigned`, `i64`
  `int64_t`, `f32` `float`, `f64` `double`, `bool` one byte. A C `enum` is
  an `i32` (not `bool`). C's `long` is `i64` on Linux/macOS, `i32` on
  Windows — per-platform declaration or an adapter.
- **Opaque handles** (`SDL_Window *`) are `u64`: a token passed along, not
  looked inside.
- **A pointer to something Goose owns** is `T&` for flat fixed `T`:
  `fn SDL_GetWindowSize(w: u64, x: i32&, y: i32&)`. `(ptr, count)` → pass
  `xs[0]` and `xs.len`. A NUL-terminated string → append `\0`, pass the
  first byte by reference:

  ```goose
  extern "SDL2" {
      fn SDL_CreateWindow(title: const u8&, x: i32, y: i32, w: i32, h: i32, flags: u32) -> u64;
  }
  let title = "Goose\0";
  let win = SDL_CreateWindow(title[0], 100, 100, 640, 480, 0);
  ```

- **Structs** cross in Goose's packed layout, matching C only when the
  fields already sit where C puts them. A `u8` before an `i32` needs
  `pad 3` where C pads; a struct C rounds up needs `pad 3` at the end. A
  struct with `pad` crosses **by reference only**. A `const Rect&` needs a
  variable (a literal cannot be a reference target).
- **Memory the library owns** comes back as a `u64`; `ffi::cstr(p)` /
  `ffi::bytes(p, n)` copy it into Goose storage:

  ```goose
  import ffi;
  extern "SDL2" { fn SDL_GetError() -> u64; }
  if SDL_Init(0x20) < 0 { abort(str("SDL: ", ffi::cstr(SDL_GetError()))); }
  ```

  `ffi::cstr(p, out) -> bool` / `ffi::bytes(p, n, out) -> bool` append into
  a builder; null pointers append nothing and give `false`. They trust the
  pointer and length, as every extern call trusts C.

- **Still an adapter's job**: callbacks (function pointers) and variadic
  functions cannot be bound — a few lines of C in a native part handle them.

## Related skills

- [extern-fn](extern-fn/skill.md)
- [sidecar-own-c](sidecar-own-c/skill.md)
- [os-module](../stdlib/os-module/skill.md)
