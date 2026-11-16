---
name: goose-editors-lsp-format
title: Editors, the language server, and goose format
category: tooling
tags: [goose, lsp, vscode, neovim, helix, emacs, formatter, goose format]
triggers: [goose editor, goose lsp, goose vscode, goose neovim, goose format, format on save]
description: >
  The compiler is its own language server (goose lsp over stdio): diagnostics
  as you type, hover, go-to-def, references, rename, semantic tokens,
  inlay hints, code lenses with instantiation counts and Run. VS Code
  extension; Neovim/Helix/Emacs configs; entryFiles initialization option;
  goose format rules and format-on-save per editor.
source: https://github.com/aardappel/goose/blob/main/docs/editors.md
---

# Editors, LSP, and the formatter

## The compiler is the language server

```bash
goose lsp        # LSP on stdin/stdout
goose lsp --probe   # prints "goose lsp <version>" and exits
```

Features: diagnostics as files change (over every file of the program),
hover (declarations, types, instantiations, doc comments), go to definition
(across imports, on import lines), references and rename, document/workspace
symbols, semantic highlighting of resolved names, signature help, inlay
hints (types of `let`/`var`, parameter names at calls), folding, code lenses
(above each generic function: its instantiation count; **Run** above `main`).

It analyzes the programs whose root files the editor names in
`initializationOptions` (`entryFiles`, absolute paths), plus every open file
none of them imports. **It never fetches a package** — one not fetched yet
is an error at its import; run `goose fetch` first.

## Editor setup

**VS Code**: the official extension (`vscode/` in the repo) starts the
server; `goose.entryFile` setting; syntax highlighting including nested
block comments, raw strings, hex floats; Markdown fenced-code highlighting;
bracket matching, snippets (`main`, `fn`, `struct`, `enum`, `for`, `guard`,
`match`); commands Check Program / Run with JIT / Build / Generate C.

**Neovim 0.11+** (`init.lua`):

```lua
vim.filetype.add({ extension = { goose = 'goose' } })
vim.lsp.config('goose', {
  cmd = { 'goose', 'lsp' },
  filetypes = { 'goose' },
  root_markers = { '.git' },
  -- init_options = { entryFiles = { '/path/to/project/main.goose' } },
})
vim.lsp.enable('goose')
```

**Helix** (`languages.toml`):

```toml
[language-server.goose]
command = "goose"
args = ["lsp"]

[[language]]
name = "goose"
scope = "source.goose"
file-types = ["goose"]
roots = [".git"]
comment-token = "//"
indent = { tab-width = 4, unit = "    " }
language-servers = ["goose"]
```

**Emacs (Eglot)**:

```elisp
(define-derived-mode goose-mode prog-mode "Goose"
  (setq-local comment-start "// "))
(add-to-list 'auto-mode-alist '("\\.goose\\'" . goose-mode))
(with-eval-after-load 'eglot
  (add-to-list 'eglot-server-programs '(goose-mode "goose" "lsp")))
(add-hook 'goose-mode-hook #'eglot-ensure)
```

For a compiler outside its source tree: `goose lsp --stdlib /path/to/goose/stdlib`.

## `goose format`

```bash
goose format src/                # rewrite every .goose file under src/
goose format --check .           # list files that are not formatted (exit 1)
goose format < in.goose > out.goose
goose format --verify file.goose # in-memory check, writes nothing
```

Rewrites in place in the standard layout: indentation, spaces between
tokens, blank lines, trailing commas (added where a list's closing bracket
starts a line). Line breaks stay where they are. The types and `=`s of a
run of declarations line up in columns. A file that does not parse is left
as-is (exit 2). Directories stand for their `.goose` files (no
dot-directories, no build trees).

## Format on save

- Neovim: `vim.api.nvim_create_autocmd('BufWritePre', { pattern = '*.goose',
  callback = function() vim.lsp.buf.format() end })` — or conform.nvim with
  `goose format -`.
- Helix: `auto-format = true` in the `[[language]]` entry.
- Emacs: `before-save-hook` → `eglot-format-buffer`.

The language server formats the editor's text with edits touching only the
lines that change.

## Related skills

- [cli-reference](../cli-reference/skill.md)
- [diagnostics](../diagnostics/skill.md)
