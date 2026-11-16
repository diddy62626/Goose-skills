---
name: goose-sidecar-own-c
title: Sidecar foo.build — a program or package with C of its own
category: interop
tags: [goose, sidecar, foo.build, c_sources, c_headers, cmake, dylibs, own C]
triggers: [goose sidecar, c_sources, cmake package, c_headers, foo.build, link library]
description: >
  C that belongs to the program goes in a sidecar foo.build beside foo.goose:
  c_sources/c_include/c_defines/c_flags/c_libs (each with platform suffixes),
  c_headers adapters using the generated types, dylibs mappings, and output
  naming. Packages use the same sidecar with cmake_target for CMake projects.
source: https://github.com/aardappel/goose/blob/main/docs/packages.md#3-a-program-with-c-of-its-own
---

# The sidecar: a program (or package) with C of its own

C that belongs to the program goes in a sidecar **`foo.build`** beside
`foo.goose`. The file looks like Goose (`//` comments, `key = value;`,
strings, `true`/`false`, `[lists]`), and every `c_*` key takes a
`_windows`/`_linux`/`_macos` suffix for a platform-only part.

```goose
// foo.build
c_sources = ["impl.c", "vendor/stb_impl.c"];   // compiled with the program's build
c_include = ["vendor"];
c_defines = ["IMPL_FAST=1"];
c_flags_cc = ["-mavx2"];                       // gcc/clang flags; c_flags_msvc for MSVC
c_libs = ["m"];                                // system libraries, by name
c_libs_windows = ["user32"];
dylibs_linux = ["SDL2=libSDL2-2.0.so.0"];      // what extern "SDL2" opens here
output = "foo";                                // executable name if not the file's
```

`goose build` compiles and links that C; `goose run` builds it into a
library the in-process run loads. Either way, `extern fn`s in the program
call it:

```goose
extern fn impl_sum(a: i32, b: i32) -> i32;
extern fn impl_count(s: const u8[:], c: u8) -> i64;   // a slice arrives as { data, len }
```

A single-header library needs one `impl.c` defining its implementation macro
and including the header.

## Headers that see the generated types

`c_headers = ["include/adapters.h"];` includes a header in the **program's
generated C**, so the C compiler can inline small adapters into Goose
functions — no separate object file needed. Headers follow the generated
types and precede extern prototypes and function bodies; they are included
in list order (command-line `--include` entries, the program's sidecar, then
packages in load order). They must compile with the selected backend,
including TinyCC — a header-only package needs no external C toolchain for
`goose run`.

The generated names follow spec §7.10:

| Goose declaration | C spelling |
|---|---|
| `mylib::Pair` | `mylib_Pair_g5` (namespace, type, `_g`, namespace length) |
| `Pair` without a namespace | `Pair_g` |
| Fields `lo`, `hi` | `lo_g`, `hi_g` |

Inspect the actual declarations with `goose -o app.c app.goose`. Use the
generated type, not a second C definition with similar fields: Goose
structs are **packed**, not naturally aligned — matching names is not ABI
compatibility. Separately compiled `c_sources` do not see the generated
declarations.

## Packages: the same sidecar

A package is a git repository (or directory):

```
mylib/
  mylib.goose        the entry: `namespace mylib;`, its declarations
  mylib.build        the sidecar, when there is C to build
  extra.goose        another module: `import mylib.extra;`
  src/, include/     the C, if any
  examples/, test/   programs of their own, each a root file
  README.md
```

Two ways to name the native C:

```goose
// plain sources
c_sources = ["src/mylib.c"];
c_include = ["include"];
c_libs_linux = ["m"];

// or a CMake project at the package root, named by its static library target
cmake_target = "mylib";
cmake_options = ["MYLIB_SIMD=ON"];
```

The CMake route configures and builds a wrapper around the package's own
CMakeLists and uses the target's complete link line — a thin Goose layer over
an existing C library needs no Goose-specific CMake code. Submodules are
cloned with it. Either way the native part is built once per C compiler
under `~/.goose/build` and linked statically. `main_thread_only = true;`
makes calling the package from a `thread_fn` a compile error;
`exclude = [...]` keeps directories out of the package.

## C appending to a Goose builder

```c
#include "goose.h"
void mylib_name(gs_rref out, int64_t id) {   /* extern fn mylib_name(out: u8[>..]&, id: i64); */
    gs_bld_append(out, "item", 4);
}
```

`goose.h` is on the include path of every native part (program's own, CMake
targets) and changes rebuild them all.

## Rules to remember

- A package's C is **self-contained**: two packages that must share C share
  a repository.
- Two packages cannot share a root: a program declaring two packages from
  one directory/repo at the same ref is an error.
- `embed_file("path")` ships compile-time data files; run-time files belong
  to the program via `resource_dir()`.

## Related skills

- [packages](../../tooling/packages/skill.md)
- [making-a-package](../../tooling/making-a-package/skill.md)
- [extern-fn](../extern-fn/skill.md)
