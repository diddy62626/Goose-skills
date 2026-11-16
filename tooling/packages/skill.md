---
name: goose-packages
title: Packages — declaring, fetching, pinning dependencies
category: tooling
tags: [goose, packages, dependencies, git, import from, fetch, root file]
triggers: [goose package, import from github, goose fetch, dependency pinning, pkg cache]
description: >
  import name from "owner/repo@v1"; (or a directory) declares a package in
  the program's root file. Only the root declares; other files' declarations
  are requirements the root must satisfy. Tags/commits pin; branches need
  --update. goose fetch pre-fetches; --offline forbids fetching. The
  ~/.goose cache layout.
source: https://github.com/aardappel/goose/blob/main/docs/packages.md#2-using-a-package
---

# Packages

A **package** is a library in a directory of its own: a module named after
it (`name.goose`), further modules beside it, and possibly C.

## Declaring

The *blessed* packages (`gfx`, `audio`, `physics`, `ui`, …) are imported by
name like any module — fetched from their repositories at a pinned tag into
`~/.goose/pkg` the first time. A compiler configured with
`-DGOOSE_BUILTIN_LIBS=all` has them compiled in (`goose toolchain` lists
them).

Any other package is declared in the program's **root file** (the one named
on the command line):

```goose
import mylib from "joe/mylib@v2";                  // GitHub, at tag v2
import mylib from "github.com/joe/mylib@3f2c1a9";  // any host, at a commit
import mylib from "https://git.example.com/x/mylib.git@main";   // any git URL, at a branch
import mylib from "../mylib";                      // a directory, used in place
```

`import name from "source";` declares the package `name` **and** imports its
entry module `name.goose`; its other modules are imported with
`import name.extra;`. A git source needs a ref after `@`.

## The flatness rule

**Only the root file declares packages.** Every other file — including the
library's own — imports by name alone. A library needing another package
writes the same `import other from "…";` line in its own files; the compiler
reads that as a **requirement**: if the root does not declare `other`, the
error says exactly what line to add to the root. Same package from a
different ref → warning; from a different source → error. So a program's
dependencies are flat, pinned, and in one place, with one version of each.

## Fetching

```bash
goose fetch app.goose            # fetch what the root declares; compile nothing
goose fetch --update app.goose   # re-fetch tag/branch pins
goose run --offline app.goose    # fetch nothing; unfetched package = error at its import
```

A package pinned to a **commit** is never fetched again; a **tag or branch**
only with `--update`. The language server never fetches. Missing tools are
named in the error: `git` to fetch, `cmake` for CMake packages, a C
compiler for C code. A program with no C and no such packages needs none of
them.

## The cache

```
~/.goose/
  config                          cc = "clang";
  pkg/<host>/<owner>/<repo>/<ref>/   fetched packages, submodules included
  build/pkg/.../<toolchain>/      their native parts, per C compiler
  build/runtime/<hash>/...        the runtime object, per C compiler and configuration
  build/prog/<name>-<hash>/       each program's C, objects and logs
  toolchain/                      what finding the C compiler took (vcvars on Windows)
```

Deleting `GOOSE_HOME` loses nothing a build cannot recreate.

## Publishing (see making-a-package)

Push the repository, tag a version, and tell users the line:

```goose
import mylib from "github.com/you/mylib@v1";
```

There is no registry to register with. A tag can move and a branch always
does — reproducible builds pin a commit. Fetched package code runs on the
user's machine when its C is built (as with any build script); pinning a
version expresses trust in that code.

## Related skills

- [making-a-package](../making-a-package/skill.md)
- [sidecar-own-c](../../interop/sidecar-own-c/skill.md)
- [cli-reference](../cli-reference/skill.md)
