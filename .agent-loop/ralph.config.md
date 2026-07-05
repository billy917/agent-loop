---
id: ralph-config
updated: 2026-07-05
---

# Ralph Loop — Project Config

Single source of truth for the loop. Every agent reads this file. Edit the values here;
you rarely need to touch the agent files themselves.

**Two stores, kept separate:**
- **Memory** — durable **domain / business / technical knowledge only**, **shared across
  every tracked change**. Agents reach it only through the **Memory API**
  (`.agent-loop/memory/memory-api.md`); the concrete backend is swappable and selected by
  `memory_implementation` below (default: `memory-generic-file`). The Curator writes it;
  everyone else `RECALL`s.
- `.agent-loop/handoff/<ISSUE-ID>/` — inter-agent handoff and progress status for **one
  tracked change** (`RESEARCH.md`, `PRD.md`, `PROGRESS.md`, `iterations/`, `scratch.md`),
  plus `.agent-loop/handoff/INDEX.md` as the registry. Plain files — not the Memory API.

## Change tracking (issue ids)

Each functional change is tracked under its own folder keyed by a Jira id or slug, so
concurrent changes never collide. Memory stays shared and accumulates across them.

- **Id convention** — Jira `^[A-Z][A-Z0-9]+-[0-9]+$` (e.g. `PROJ-123`) or slug
  `^[a-z0-9]+(-[a-z0-9]+)*$` (e.g. `add-oauth-login`). No spaces/slashes/quotes. Reserved
  (never an id): `INDEX.md`, `_templates/`.
- **Kickoff** — invoke the Researcher with the id in your message ("Research PROJ-123: ...").
  It creates `.agent-loop/handoff/PROJ-123/` and registers the change.
- **Id flows via conversation** — handoffs preserve context, so each agent inherits the
  active id; every artifact also carries `issue:` frontmatter as a fallback.
- **Registry** — `.agent-loop/handoff/INDEX.md` tracks every change and its status
  (`RESEARCH -> PLANNED -> ACTIVE -> COMPLETE`, or `HALTED — NEEDS HUMAN`).
- **Memory provenance** — distilled notes tag `source_issues` with the contributing id(s);
  there are no per-issue memory silos.

---

## 1. Models per agent

Each agent also declares a default `model:` in its own frontmatter. **This table is the
authoritative override** — set the agent frontmatter to match what you choose here. Model
names must match what your Copilot plan exposes in the picker; the examples below are
placeholders — **replace them with your real model names**. Copilot falls back to the picker
default if a name is unknown, and a list is tried in order.

| Agent | Role | Recommended tier | Example value |
|---|---|---|---|
| RalphResearcher | Research & synthesis | strongest reasoning | `['Claude Opus 4.5', 'GPT-5.2']` |
| RalphPlanner | Task decomposition | strongest reasoning | `['Claude Opus 4.5', 'GPT-5.2']` |
| RalphCoordinator | Orchestration / bookkeeping | fast, capable | `['Claude Sonnet 4.5', 'GPT-5.2']` |
| RalphExecutor | Implementation | strong coding | `['Claude Sonnet 4.5', 'GPT-5.2']` |
| RalphReviewer | Verification | strongest reasoning | `['Claude Opus 4.5', 'GPT-5.2']` |
| RalphCurator | Memory consolidation | fast, capable | `['Claude Sonnet 4.5', 'GPT-5.2']` |

To change a model: edit the `model:` line in the matching `.github/agents/*.agent.md`.
Leaving `model:` off makes an agent use whatever is selected in the picker.

---

## 2. Quality gates (language-agnostic)

Gates are **commands**, not assumptions. Fill in the commands for this project. Prefer a
`Makefile` / project scripts if they exist. Leave a gate blank to skip it.

- **Build:** `<e.g. make build | npm run build | cargo build | go build ./... | mvn -q compile>`
- **Test:** `<e.g. make test | npm test | pytest | cargo test | go test ./...>`
- **Lint:** `<e.g. make lint | npm run lint | ruff check | golangci-lint run>`
- **Format check:** `<e.g. prettier -c . | black --check . | gofmt -l .>`
- **Type check:** `<e.g. tsc --noEmit | mypy . | (n/a)>`
- **Review gate:** RalphReviewer verdict must be `PASS`.

A task passes only when **every non-blank gate passes AND the Review gate is PASS.**

---

## 3. Escalation limits (when to flag a human)

The loop never spins forever. When a bound is hit, the Coordinator **halts** and writes a
`## Needs Human Intervention` block to `.agent-loop/handoff/<ISSUE-ID>/PROGRESS.md` (and marks
the registry row `HALTED — NEEDS HUMAN`), then fires the "Manual intervention required" handoff.

- `max_executor_retries`: **3** — Executor self-fix attempts on a failing gate before it returns `BLOCKED`.
- `max_review_cycles`: **3** — Executor<->Reviewer FAIL round-trips on one task before the Coordinator halts.
- `max_iterations`: **50** — hard ceiling on total loop iterations (safety stop).
- `halt_on_ambiguity`: **true** — if a task needs a product/architecture decision not in the PRD, halt and ask.

---

## 4. Memory (knowledge only, shared)

Agents interact with memory **only** through the Memory API
(`.agent-loop/memory/memory-api.md`): `RECALL · READ · UPDATE · DELETE · DISTILL`. The API is
fixed; the storage behind it is swappable.

- **`memory_implementation`: `memory/memory-generic-file/`** — the active knowledge backend,
  relative to `.agent-loop/` (i.e. `.agent-loop/memory/memory-generic-file/`). This is the
  default markdown-on-disk implementation. To swap in another backend (vectors, SQLite, a
  graph DB, an MCP server), build one that satisfies the conformance checklist in
  `memory-api.md`, point this value at it, and change nothing in the agents. See
  `.agent-loop/memory/README.md` for the full swap guide.
- **Domains:** `<e.g. api, auth, data-model, build-system, deploy>` — top-level knowledge
  buckets. In the default `memory-generic-file` backend these are subfolders under
  `memory/memory-generic-file/knowledge/`.
- **Shared, not per-issue:** all tracked changes `RECALL` and `DISTILL` into the same store;
  notes carry `source_issues` provenance.
- **Distill cadence:** Curator distills a change's `.agent-loop/handoff/<ISSUE-ID>/` learnings
  into memory after every task `PASS` (incremental) + a full pass at loop completion.
- **Prune cadence:** at loop completion (project-wide knowledge maintenance only).
- **Provisional expiry:** `provisional` knowledge idle `<30>` days with `use_count: 0` is
  expired by `DELETE`.

> Iteration logs, status, and scratch are **not** memory and are **not** pruned by the
> Curator — they live in `.agent-loop/handoff/<ISSUE-ID>/` and are yours to archive as you like.

---

## 5. Loop artifacts (per change, created at runtime)

- `.agent-loop/handoff/INDEX.md` — registry of all tracked changes + status.
- `.agent-loop/handoff/<ISSUE-ID>/RESEARCH.md` — Researcher output, human-reviewable.
- `.agent-loop/handoff/<ISSUE-ID>/PRD.md` — Planner output: tasks + acceptance criteria + gate commands.
- `.agent-loop/handoff/<ISSUE-ID>/PROGRESS.md` — live status board + blockers + human-intervention surface.
- `.agent-loop/handoff/<ISSUE-ID>/iterations/<date>-task-0XX.md` — per-iteration log + change manifest + learnings.
- `.agent-loop/handoff/<ISSUE-ID>/scratch.md` — current-task scratch.
- `.agent-loop/handoff/_templates/iteration-log.md` — template the Executor copies per iteration.
