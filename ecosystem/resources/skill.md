---
name: goose-resources
title: Resources — official docs, samples, community
category: ecosystem
tags: [goose, resources, docs, samples, community, discord, releases]
triggers: [goose documentation, goose samples, goose community, goose discord, goose links]
description: >
  The authoritative links: the language docs (tutorial/reference/spec), the
  standard library reference, CLI and packages docs, the 31 samples in
  reading order, benchmarks, design notes, releases, and the Discord. What
  to read in what order when an agent needs to verify a detail.
source: https://github.com/aardappel/goose
---

# Resources

## Official (source of truth — these skills are grounded in these)

| Resource | URL |
|---|---|
| Repository | https://github.com/aardappel/goose |
| Docs index | https://aardappel.github.io/goose |
| Tutorial (an afternoon) | https://github.com/aardappel/goose/blob/main/docs/tutorial.md |
| Language reference | https://github.com/aardappel/goose/blob/main/docs/reference.md |
| Specification (exact rules) | https://github.com/aardappel/goose/blob/main/docs/goose_spec.md |
| Standard library | https://github.com/aardappel/goose/blob/main/docs/stdlib.md |
| Command line | https://github.com/aardappel/goose/blob/main/docs/command_line.md |
| Packages | https://github.com/aardappel/goose/blob/main/docs/packages.md |
| Samples (31 programs) | https://github.com/aardappel/goose/tree/main/samples |
| Benchmarks | https://github.com/aardappel/goose/tree/main/bench |
| Implementation notes | https://github.com/aardappel/goose/blob/main/docs/implementation.md |
| Design notes | https://github.com/aardappel/goose/tree/main/docs/design |
| Releases | https://github.com/aardappel/goose/releases |
| Discord | https://discord.gg/397eZdbnsR |
| Companion packages | https://github.com/aardappel/goose_gfx , https://github.com/aardappel/goose_physics |

## The samples, in reading order

`01_tour`, `02_memory`, `03_strings`, `04_errors`, `05_shapes`,
`06_functions`, `07_sieve`, `08_bignum`, `09_sorting`, `10_sudoku`,
`11_maze`, `12_huffman`, `13_linked_list`, `14_bst`, `15_word_freq`,
`16_records`, `17_calc`, `18_json`, `19_vm`, `20_life`, `21_raytrace`,
`22_image_filters`, `23_mandelbrot_threads`, `24_call_c`, `25_serialize`,
`26_file_tree`, `27_gfx_cube`, `28_physics_boxes`, `29_ui_todo`,
`30_mini_minecraft`, `31_mini_doom`, `32_pool_worker_snapshots`.

Start with `01_tour` and `02_memory`; `13_linked_list`, `14_bst` and
`18_json` show the data structures in detail; `26_file_tree` uses both pool
kinds together; `24_call_c` and `25_serialize` cover the boundary and
serialization.

## When an agent should verify against the source

These skills condense the docs as of November 2026. Goose is young and
moves fast; for anything this corpus does not answer (exact spec wording,
new stdlib functions, changed flags), fetch the corresponding doc from the
repository above. The spec (`goose_spec.md`) has the final say on every
rule; each skill's `source:` field points at the doc section it came from.

## Design notes worth reading

`relative_references` (why two link forms), `serialization` (the verified
loader), `index_types`, `const_for`, `namespaces`, `owning_dynamic_data`
(slice pools worked example), `jit_backend` (TinyCC limits), `prior_art`
(the memory model's lineage).

## Related skills

- [what-is-goose](../../getting-started/what-is-goose/skill.md)
- [limitations](../limitations/skill.md)
