---
id: memory-readme
updated: 2026-07-05
---

# Memory — a swappable knowledge store

Memory is split into two layers on purpose:

| Layer | File / folder | Role | Changes when you swap backends? |
|---|---|---|---|
| **Contract** | [`memory-api.md`](./memory-api.md) | The stable API the agents depend on: the operations (`RECALL · READ · UPDATE · DELETE · DISTILL`), the four invariants, and the scope boundary. | **No** — fixed |
| **Implementation** | [`memory-generic-file/`](./memory-generic-file/README.md) | The concrete backend that realizes the contract. The default is plain markdown-on-disk. | **Yes** — this is the pluggable part |

## The decoupling decision (why this exists)

The agents were originally written against a specific file-based memory layout — they knew
about `knowledge/<domain>/`, `procedures/`, `corrections.log.md`, and so on. That coupled
the *loop* to one *storage choice*: you couldn't move to vectors, SQLite, a graph DB, or an
MCP-backed store without rewriting every agent.

So the two are now separated:

- **Agents depend only on `memory-api.md`** — the verbs and invariants. They never name a
  folder inside the store or a frontmatter field. When an agent needs knowledge it
  `RECALL`s; when the Curator promotes a learning it `DISTILL`s. That's the whole surface.
- **The concrete store is a module** selected in `.agent-loop/ralph.config.md` via
  `memory_implementation`. `memory-generic-file/` is **one** implementation — the reference
  one, chosen because it needs zero infrastructure. It is not privileged; it is replaceable.

The result: the knowledge backend is an implementation choice, not an architectural one.
Replace it as you see fit.

## What's fixed vs. what's swappable

**Fixed (the contract, in `memory-api.md`):** the five operations and their semantics; the
four invariants (knowledge-only; earned via `DISTILL`, never captured; correction is
retrieval-triggered; supersede, don't overwrite); the two logical stores (semantic +
procedural) and the corrections audit; the per-note logical attributes; the scope boundary
vs. `.agent-loop/handoff/`.

**Swappable (the implementation):** everything about *how* those are stored — file layout,
frontmatter schema, indexing/navigation, the storage engine itself.

## How to swap in your own backend

1. **Build an implementation** — a store (folder, service, adapter, MCP server) that honors
   the **conformance checklist** at the bottom of [`memory-api.md`](./memory-api.md): the
   four invariants, the five operations, the two logical stores + audit, and the note
   attributes. Ship it with a short README describing how the abstract API maps onto it
   (use `memory-generic-file/README.md` as the template).
2. **Point the config at it** — set `memory_implementation` in `.agent-loop/ralph.config.md`
   to your implementation (e.g. `memory/memory-vector/` or an adapter the agents can reach).
3. **Change nothing in the agents** — they already speak only `memory-api.md`. If your
   backend honors the contract, the loop runs unchanged.

Keep the **scope boundary**: whatever you build stores durable knowledge only. Handoff,
progress, and scratch always stay in `.agent-loop/handoff/` and are never part of a memory
implementation.

## Files here

- `memory-api.md` — the contract (do not fork per-implementation; it is shared).
- `memory-generic-file/` — the default reference implementation (markdown-on-disk).
