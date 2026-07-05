---
id: memory-index
updated: 2026-07-05
---

# Memory Index (memory-generic-file)

Navigation root for the **memory-generic-file** backend — the default markdown-on-disk
implementation of the Memory API. **Memory holds only durable domain, business, and
technical knowledge of this project, shared across every tracked change.** Transient,
per-change handoff and progress state is NOT here — it lives in `.agent-loop/handoff/<ISSUE-ID>/`.
Notes carry `source_issues` provenance so what one change learns benefits the next.

See [`../memory-api.md`](../memory-api.md) for the operation contract
(`RECALL · READ · UPDATE · DELETE · DISTILL`) and the invariants — that is what the agents
depend on. This file and the folders beside it are this backend's concrete realization; see
[`./README.md`](./README.md) for the API-to-files mapping.

## Domains
_None yet._ Fill from `.agent-loop/ralph.config.md` § Memory settings, e.g.:
- `api/` — knowledge/api/_index.md
- `auth/` — knowledge/auth/_index.md
- `data-model/` — knowledge/data-model/_index.md

## Stores
- Semantic (durable facts): `knowledge/`
- Procedural (reusable technical how-to): `procedures/`
- Audit of corrections: `corrections.log.md`

## Not here (see .agent-loop/handoff/<ISSUE-ID>/)
- Registry of all tracked changes: `.agent-loop/handoff/INDEX.md`
- Inter-agent handoff: `RESEARCH.md`, `PRD.md`
- Progress status: `PROGRESS.md`
- Iteration logs + change manifests: `iterations/`
- Scratch: `scratch.md`
