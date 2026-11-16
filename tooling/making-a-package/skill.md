---
name: goose-making-a-package
title: Making and publishing a Goose package
category: tooling
tags: [goose, package authoring, publishing, namespace, entry file, examples, testing]
triggers: [make a goose package, publish library, package layout, namespace convention]
description: >
  Package layout (entry module with the package's namespace), the
  requirement rule for a library's own dependencies, examples and tests as
  root files, extern blocks for machine libraries, embed_file for shipped
  data, and the publish workflow (push, tag, tell users the import line).
source: https://github.com/aardappel/goose/blob/main/docs/packages.md#4-making-a-package
---

# Making a package

## Layout

```
mylib/
  mylib.goose        the entry: `namespace mylib;`, its declarations
  mylib.build        the sidecar, when there is C to build (optional)
  extra.goose        another module: `import mylib.extra;`
  src/, include/     the C, if any
  examples/, test/   programs of their own, each a root file
  README.md
```

Inside the package, `import helper;` finds `helper.goose` under the
package's root, `import .helper;` finds it relative to the importing file,
and `import std;` reaches the standard library. **By convention a package's
files declare its name as their namespace** — the namespace (not the import
path) sets the generated C names (`mylib::Pair` → `mylib_Pair_g5`).

## A library's own dependencies

A library that needs another package writes the requirement in its entry
file: `import vec_extras from "github.com/x/vec_extras@v2";`. The compiler
checks it against the program's root: missing → error naming the line to
add; different ref → warning; different source → error. The library's own
examples and tests are **root files**, so in them the line declares the
package, and the examples run on their own.

## Native part

- `extern "name" { ... }` blocks bind machine libraries by name at program
  start, with sidecar `dylibs_*` mappings for platform filenames — a
  program calling only these compiles and links no native code.
- `c_sources`/`c_headers`/`cmake_target` compile the package's own C (see
  sidecar-own-c). The native part builds once per C compiler under
  `~/.goose/build` and links statically.
- `main_thread_only = true;` in the sidecar makes calling the package from
  a `thread_fn` a compile error. Document any non-thread-safe code — by
  convention C is treated as thread-safe unless stated otherwise.
- `exclude = ["tools", "third_party/demo"];` keeps directories out of the
  package. Files below a directory that has its own sidecar (an example
  with its own C) are excluded automatically.

## Files the package ships

- `embed_file("path")` reads a file **when the program is compiled**
  (relative to the file containing the call) into static data — a font, a
  table, a shader. This is how a package ships data without separate
  deployment.
- Files read at run time belong to the program (`resource_dir()`).

## API hygiene

- Take arrays as `T[:]` for reads, `T[>..]&` for builders, `slot_pool<T>&`
  for pools; return fresh arrays as `T[>..]` (built in the caller's
  destination); "maybe an element" is `T?`; errors as trailing `bool` or
  `return E from <your public entry>` / `exception`.
- Write a `format(out: u8[>..]&, v: T)` overload per type so `print` renders
  it well everywhere; a `hash` overload makes a type a dictionary key.
- Namespace privacy does not exist in v1 — encapsulation is by convention;
  prefix internals or put them in an `internal` module.

## Publishing

Push the repository, tag a version, and tell users the line:

```goose
import mylib from "github.com/you/mylib@v1";
```

No registry to register with. Tag can move, branch always does —
reproducible users pin a commit. A package is its directory: two packages
cannot share a root (a program declaring two packages from one directory or
one repo at the same ref is an error).

## Related skills

- [packages](../packages/skill.md)
- [sidecar-own-c](../../interop/sidecar-own-c/skill.md)
- [std-algorithms](../../stdlib/std-algorithms/skill.md)
