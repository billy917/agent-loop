---
id: memory-api
v: 3.0
updated: 2026-07-05
---

# Memory API (the contract) — knowledge only

This file is the **stable contract** between the agents and the knowledge store. The
agents depend on *this* — the operations, the invariants, and the scope boundary — and on
nothing beneath it. **How** the store is realized (markdown files, vectors, SQLite, a
graph DB, an MCP server) is an **implementation detail** that lives behind this API and
can be swapped without touching a single agent.

> **The active implementation is chosen in `.agent-loop/ralph.config.md` -> `memory_implementation`.**
> The default is [`memory-generic-file/`](./memory-generic-file/README.md) — a plain
> markdown-on-disk backend. See [`./README.md`](./README.md) for the decoupling rationale
> and how to swap it.

**Memory holds one thing: the durable domain, business, and technical knowledge of this
coding project.** Nothing transient lives here. The operations below are the contract; any
conforming implementation must preserve their *semantics* and the invariants — not any
particular file layout, schema, or storage engine.

## Scope boundary (important)

Inter-agent handoff and progress status do **not** use this API. They live in per-change
folders under `.agent-loop/handoff/<ISSUE-ID>/` (`RESEARCH.md`, `PRD.md`, `PROGRESS.md`,
`iterations/`, `scratch.md`). Memory is *only* for what should still be true and useful
after this feature — and the next — is done.

**Memory is shared project-wide**, not per-issue. Every tracked change retrieves from the
same store and distills back into it; notes carry **provenance** (which change(s)
contributed them), so what `PROJ-123` learned is available to `PROJ-456`.

## Core invariants (an implementation MUST preserve all four)

1. **Knowledge only** — memory stores durable facts and reusable how-to, never iteration
   logs, status, scratch, or handoff. Those belong in `.agent-loop/handoff/`.
2. **Earned, never captured** — knowledge enters memory only through `DISTILL` (slow,
   deliberate) from the transient `.agent-loop/handoff/` record. Nothing writes raw
   experience straight into memory.
3. **Correction is retrieval-triggered** — a fact changes only when retrieving it surfaces
   a conflict (`UPDATE`).
4. **Supersede, don't overwrite** — a replaced fact is retained with a supersession pointer
   to its replacement; history is never silently lost.

## The two logical stores (abstract)

Every implementation exposes two logical kinds of knowledge plus a correction audit. The
names below are *roles*, not paths — an implementation maps them onto its own storage.

| Logical store | Holds | Written by | Mutability |
|---|---|---|---|
| **Semantic** | Durable domain / business / technical facts about the project | `DISTILL` / `UPDATE` | supersede-in-place |
| **Procedural** | Reusable technical how-to (build, test, migrate, deploy, recover) | `DISTILL` / revise | revise / deprecate |
| **Corrections audit** | Append-only record of every `UPDATE` | `UPDATE` | append-only |

Each note carries, at minimum, these **logical attributes** (an implementation may name or
store them however it likes, but must support them):

- **status** — semantic: `active | provisional | superseded`; procedural: `active | deprecated`
- **confidence** — trace strength; gates whether new evidence overwrites (`high | med | low`)
- **links** — related notes (the retrieval graph)
- **recency / frequency** — last-used + use-count, for activation ranking (bumped on retrieval)
- **supersession pointer** — replacement note for a superseded fact (redirect target)
- **trigger** — when to surface a procedure (retrieval cue)
- **provenance** — the tracked-change id(s) that contributed the note; appended on each `DISTILL`

## Operations (the API)

Agent-facing verbs: **`RECALL · UPDATE · DELETE · DISTILL`**. `READ` is an internal
**primitive** the others call; agents rarely invoke it directly. Activation bumps and
mismatch-checks live in `RECALL`, never in `READ`.

> There is intentionally **no `WRITE`** in this API. Raw capture happens in
> `.agent-loop/handoff/`; only `DISTILL` promotes a distilled subset into memory.

### READ(key, touch=false) — primitive: fetch by identity
Pure fetch; returns one note or nothing. If the note is superseded, redirect via its
supersession pointer. If `touch=true`, bump recency/frequency. No mismatch-check, no other
side effects. Internal callers (`UPDATE`, `DELETE`) use `touch=false`.

### RECALL(query) — associative retrieval (the verb agents call)
1. **Resolve** the relevant region of the store for the query.
2. **Spread activation**: follow `links` outward, reading candidates; stop when answered or
   activation is weak (don't walk the whole graph).
3. **Rank**: relevance, tie-break recency then frequency. Prefer `status: active`.
4. **On surfaced hits only**: `touch` them, and run the mismatch check -> `UPDATE` on conflict.

Return top-k (may legitimately be empty — the caller judges).

### UPDATE(key, evidence) — retrieval-triggered correction
Trigger only on a mismatch surfaced by `RECALL` / `DISTILL` (user correction, new evidence,
contradiction). Weigh new evidence against the note's `confidence` + age.
**Supersede-in-place**: the old note is retained as `superseded` with a pointer to the
replacement; the new/edited note becomes `active`. Append one line to the corrections audit.

### DELETE(target) — knowledge maintenance
Expire stale `provisional` knowledge (past the config's idle / zero-use threshold). Dedupe
near-identical notes; trim over-linked hubs; repair broken `links` and index backlinks.
**Never hard-delete superseded semantic notes** — history is kept via the supersession
pointer.

### DISTILL(source) — transient -> durable (the only way in)
Input is the transient `.agent-loop/handoff/` record (research notes, iteration
`## Learnings`, review findings), **not** an in-memory store. For each undistilled source
item:
- Extract the **context-free gist** — the durable domain/business/technical fact,
  independent of task wording (e.g. "auth middleware lives in `middleware/auth.*` and
  expects a `roles` claim").
- **Interleave, don't overwrite**: fold the gist into the relevant semantic note. On
  conflict, run the mismatch check -> `UPDATE` (supersede-in-place, log it).
- Promote recurring how-to into the procedural store.
- Promote `provisional` knowledge to `active` once implementation confirms it.
- Append the contributing change id to the note's **provenance**.
- Mark the source item distilled in `.agent-loop/handoff/<ISSUE-ID>/` (so it isn't
  distilled twice).

Keep notes **atomic** and **linked**; update the store's indexes for new hubs.

## When each fires

| Signal | Operation |
|---|---|
| Need project knowledge / "what do we know about..." | `RECALL` |
| A recalled fact conflicts with new evidence / user correction | `UPDATE` |
| Knowledge stale, duplicate, or expired-provisional | `DELETE` |
| Learnings in `.agent-loop/handoff/` should become durable knowledge (checkpoint / loop end) | `DISTILL` |
| Exact note identity already known (rare) | `READ` |

## Implementing a backend (conformance checklist)

To provide an alternative implementation (and point `memory_implementation` at it), honor:

- [ ] The **four invariants** above, without exception.
- [ ] The **five operations** with the semantics above (internal names may differ, but the
      agent-facing verbs `RECALL · UPDATE · DELETE · DISTILL` and the `READ` primitive must
      be realizable).
- [ ] The **two logical stores** (semantic + procedural) and the **corrections audit**.
- [ ] The **logical attributes** each note carries (status, confidence, links,
      recency/frequency, supersession pointer, trigger, provenance).
- [ ] The **scope boundary**: never absorb handoff/progress/scratch; those stay in
      `.agent-loop/handoff/`.
- [ ] A short **implementation README** describing how these map onto the concrete store, so
      agents (and humans) can operate it. See `memory-generic-file/README.md` as the
      reference example.

Nothing in the agent files references your internal layout — only these verbs and
invariants — so a conforming backend is a drop-in replacement.
