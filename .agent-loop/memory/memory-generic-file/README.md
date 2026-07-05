---
id: memory-generic-file
updated: 2026-07-05
---

# memory-generic-file — reference implementation of the Memory API

The **default** knowledge backend: the two logical stores of
[`../memory-api.md`](../memory-api.md) realized as plain markdown files on disk, one
concept per file. No database, no service, no dependencies — which is exactly why it is the
default (see the project README's goals: drop-in, no-install, closed-environment friendly).

This file is the **binding**: it says how the abstract API maps onto concrete files, fields,
and navigation. The agents never read this file to operate — they read `memory-api.md`. This
spec is for humans, and for whoever maintains or replaces the backend.

> **Swapping this out?** Provide another folder/adapter that satisfies the conformance
> checklist in `../memory-api.md`, then point `memory_implementation` in
> `.agent-loop/ralph.config.md` at it. Nothing in the agents changes.

## API -> files mapping

| API concept (`memory-api.md`) | In this implementation |
|---|---|
| Semantic store | `knowledge/<domain>/<slug>.md` — one concept per file, grouped by domain |
| Procedural store | `procedures/<slug>.md` |
| Corrections audit | `corrections.log.md` (append-only) |
| Store navigation / "resolve the region" | `INDEX.md` (domain map) -> per-store/`<domain>/_index.md` -> hub notes |
| Note identity (the `key` in `READ`) | the note's **slug** (globally unique + stable) |
| Logical attributes | YAML frontmatter fields (legend below) |

## Layout

```
memory-generic-file/
├── INDEX.md                         # domain map + entry point (navigation root)
├── knowledge/
│   ├── _index.md                    # local map/schema for the semantic store
│   └── <domain>/<slug>.md           # 1 concept/file; + _index.md per domain
├── procedures/
│   ├── _index.md
│   └── <slug>.md
└── corrections.log.md               # append-only audit of every UPDATE
```

Rules: knowledge grouped **by domain** (subdivide when a domain exceeds ~15 notes);
**atomic** (one concept per file -> local updates); **slugs globally unique + stable** (a
link is just the slug, so renaming breaks inbound links — avoid).

## Frontmatter legend (the concrete attribute schema)

- `status` — knowledge: `active | provisional | superseded`; procedure: `active | deprecated`
- `confidence` — `high | med | low` (trace strength; gates whether new evidence overwrites)
- `links` — related slugs (this is the retrieval graph)
- `last_used` / `use_count` — recency + frequency (activation; set by `RECALL` on surfaced hits)
- `superseded_by` — replacement slug (the supersession pointer / redirect target)
- `trigger` — when to surface a procedure (retrieval cue)
- `source_issues` — list of tracked-change ids that contributed to this note (provenance);
  appended on each `DISTILL`

## How the operations run against these files

- **READ(slug, touch)** — open `knowledge/<domain>/<slug>.md` or `procedures/<slug>.md`. If
  `status: superseded`, redirect via `superseded_by`. `touch=true` bumps
  `last_used`/`use_count`.
- **RECALL(query)** — resolve domain(s) via `INDEX.md` -> domain `_index.md` -> its hubs;
  spread activation along `links`; rank by relevance then `last_used`/`use_count`; `touch`
  surfaced hits and mismatch-check -> `UPDATE` on conflict.
- **UPDATE(slug, evidence)** — supersede-in-place: old note keeps `status: superseded` +
  `superseded_by`; new/edited note becomes `active`; append one line to `corrections.log.md`.
- **DELETE(target)** — expire idle `provisional` notes (per config threshold), dedupe, trim
  over-linked hubs, repair `links` / `_index.md` backlinks; never hard-delete superseded
  semantic notes.
- **DISTILL(source)** — read the `.agent-loop/handoff/<ISSUE-ID>/` record; fold each
  context-free gist into the relevant `knowledge/<domain>/` note (or a new `procedures/`
  note); append the change id to `source_issues`; update `_index.md`/`INDEX.md`; mark the
  source item distilled.

## Housekeeping files

`INDEX.md`, each store's `_index.md`, and `corrections.log.md` ship pre-seeded and empty so
the store is ready to use on drop-in. Domains are declared in `.agent-loop/ralph.config.md`
§ Memory settings and become subfolders under `knowledge/`.
