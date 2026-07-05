# Ralph Loop (extended) — research → plan → implement → review

A drop-in **GitHub Copilot** custom-agent framework that runs an autonomous loop over
a coding project: **research → plan → implement → review**, with a clean split between
durable knowledge and transient state, bounded quality gates that **flag a human**
when they can't pass, and a per-agent model you can customize.

Inspired by [ralph-copilot](https://github.com/giocaizzi/ralph-copilot); extended with
a first-class **Researcher**, a memory-keeping **Curator**, a knowledge-only **memory
API**, **per-tracked-change handoff folders** (keyed by Jira id / slug), **human
escalation** on gate failure, and **per-agent model config**.

## The two-store split (the core idea)

| | `.agent-loop/memory/` | `.agent-loop/handoff/<ISSUE-ID>/` |
|---|---|---|
| Holds | Durable **domain / business / technical knowledge** | Inter-agent handoff + **progress status** |
| Scope | **Shared** across every tracked change | **One tracked change** (Jira id / slug) |
| Examples | "auth uses OAuth2", "orders table uses optimistic locking", "how to run migrations" | `RESEARCH.md`, `PRD.md`, `PROGRESS.md`, iteration logs, scratch |
| Lifetime | Survives across tasks, features, and changes | Transient, per-change working state |
| API | `RECALL · READ · UPDATE · DELETE · DISTILL` (`.agent-loop/memory/memory-framework.md`) | Plain files — **not** the memory framework |
| Who writes | Researcher (own research) + Curator, via `DISTILL` | Whichever agent owns the file |

Every agent `RECALL`s knowledge from the shared `.agent-loop/memory/` for context, but records its
work in that change's `.agent-loop/handoff/<ISSUE-ID>/`. Durable learnings surface in an iteration
log's `## Learnings`; the Curator `DISTILL`s those into `.agent-loop/memory/`, tagged with
`source_issues` provenance so what one change learns benefits the next. The two never blur.

### Working multiple changes at once
Each functional change gets its own folder under `.agent-loop/handoff/`, keyed by its issue id, so
concurrent changes stay isolated. `.agent-loop/handoff/INDEX.md` is the registry/dashboard across
all of them (id, title, status, path). Kick a change off by invoking the Researcher
with its id — e.g. *"Research PROJ-123: add OAuth login"* — and the id flows through the
handoffs; every artifact also carries `issue:` frontmatter as a fallback.

## Install (drop-in)

Two things get copied into your project: the **agent definitions** (which Copilot
auto-discovers) and the **framework base** the agents read and write. They go in two
fixed places:

```
your-project/                          ← your existing project root
├── .github/
│   └── agents/                        ← Copilot only discovers agents HERE (fixed path)
│       ├── researcher.agent.md
│       ├── planner.agent.md
│       ├── coordinator.agent.md
│       ├── executor.agent.md
│       ├── reviewer.agent.md
│       └── curator.agent.md
└── .agent-loop/                        ← the framework base (agents reference this name)
    ├── ralph.config.md                ← models, gates, escalation limits, domains — EDIT THIS
    ├── memory/                        ← durable, shared project knowledge
    │   ├── memory-framework.md        ← the memory API: RECALL·READ·UPDATE·DELETE·DISTILL
    │   ├── INDEX.md                   ← domain map / entry point
    │   ├── knowledge/                 ← semantic facts, grouped by domain (+ _index.md)
    │   ├── procedures/                ← reusable technical how-to (+ _index.md)
    │   └── corrections.log.md         ← audit log of knowledge corrections
    └── handoff/                       ← per-change working state (mostly created at runtime)
        ├── INDEX.md                   ← registry of every tracked change + status
        ├── README.md                  ← what this folder is
        ├── _templates/
        │   └── iteration-log.md       ← copied per iteration by the Executor
        └── <ISSUE-ID>/                ← one folder per Jira id / slug (auto-created)
            ├── RESEARCH.md  PRD.md  PROGRESS.md  scratch.md
            └── iterations/<date>-task-0XX.md
```

**Where each piece goes:**

| Copy this | To here | Why |
|---|---|---|
| the six `*.agent.md` files | `.github/agents/` | Copilot only loads custom agents from `.github/agents/` — this path is fixed and cannot move |
| the whole `.agent-loop/` folder | your project root | the framework base; every agent references `.agent-loop/…` paths, so **keep the folder name `.agent-loop`** |

Nothing else is required — the `.agent-loop/memory/` and `.agent-loop/handoff/` scaffolds
ship ready to use, and the per-change `.agent-loop/handoff/<ISSUE-ID>/` folders are
created for you at runtime. If you want the base somewhere other than the project root,
you'd have to update the `.agent-loop/` paths inside the six agent files to match.

**Then:**

1. Edit **`.agent-loop/ralph.config.md`** — set the model per agent, the gate commands for your stack, the escalation limits, and your memory domains.
2. Confirm each agent's `model:` line (or delete it to use the picker default).
3. In Copilot Chat / CLI, select **RalphResearcher** and describe the change **with its issue id** — e.g. *"Research PROJ-123: add OAuth login"*.
4. Follow the handoff buttons through the loop; watch progress across changes in `.agent-loop/handoff/INDEX.md`.

## The six agents

All paths below are scoped to the active change's `.agent-loop/handoff/<ISSUE-ID>/`.

| Agent | Phase | Reads | Writes |
|---|---|---|---|
| **RalphResearcher** | research | `RECALL` shared memory | `DISTILL` → memory; creates `<ID>/RESEARCH.md` + registers the change |
| **RalphPlanner** | plan | `RECALL` memory; `<ID>/RESEARCH.md` | `<ID>/PRD.md`, `PROGRESS.md`; registry → PLANNED |
| **RalphCoordinator** | orchestrate | `RECALL` memory; `<ID>/` | `<ID>/PROGRESS.md` + registry (ACTIVE/HALTED/COMPLETE) |
| **RalphExecutor** | implement | `RECALL` memory; `<ID>/` | `<ID>/PROGRESS.md` + `<ID>/iterations/` |
| **RalphReviewer** | review | `RECALL` memory; `<ID>/` + files | review report (to Coordinator) |
| **RalphCurator** | memory | `<ID>/` learnings | `DISTILL`/`UPDATE`/`DELETE` → **shared memory** (`source_issues`) |

## State machine

`H/` = `.agent-loop/handoff/<ISSUE-ID>/` · memory is shared across all issues.

```
"Research PROJ-123: …"
   │
   ▼
Researcher ── RECALL + DISTILL ─▶ shared .agent-loop/memory/   ── creates H/RESEARCH.md + registers in .agent-loop/handoff/INDEX.md
   │  [you review]
   ▼
Planner ── RECALL memory ───────────────────────── writes ─▶ H/PRD.md + H/PROGRESS.md  (registry → PLANNED)
   │  [you review + approve]
   ▼
Coordinator ◀───────────────────────────────┐ (loop)   RECALL memory · read/write H/ · registry → ACTIVE
   │ spawn with <ISSUE-ID> (fresh context)   │
   ▼                                         │
Executor ─ implement 1 task, run GATES ─────▶ H/PROGRESS.md + H/iterations/<log>
   │       (## Learnings captured for the Curator)
   ├─ gate fails > max_executor_retries ─▶ BLOCKED ─┐
   ▼ COMPLETE                                        │
Reviewer ─ read-only verify ─ PASS/FAIL             │
   ├─ FAIL > max_review_cycles ──────────▶ BLOCKED ─┤
   ▼ PASS                                            ▼
Curator ─ DISTILL learnings ─▶ shared .agent-loop/memory/  HALT → H/PROGRESS.md "Needs Human
   │        (tag source_issues)               Intervention" + registry HALTED
   ▼                                          + Manual-Intervention handoff
more tasks? ─ yes ─▶ Coordinator
   │ no
   ▼
full-project gates ─ fail ─▶ HALT → human
   │ pass
   ▼
Curator final DISTILL + prune ─▶ registry → COMPLETE ─▶ <promise>COMPLETE</promise>
```

## Gates & human escalation

Gates are commands you declare in `.agent-loop/ralph.config.md` (build / test / lint / format /
type) plus the Reviewer's PASS. The loop is **bounded**: the Executor self-fixes up to
`max_executor_retries`; Reviewer FAILs re-run up to `max_review_cycles`. When a bound
is hit — or a decision isn't in the PRD, or the final full-project gates fail — the
Coordinator **stops**, writes a `## Needs Human Intervention` block to
`.agent-loop/handoff/<ISSUE-ID>/PROGRESS.md`, marks the registry row `HALTED — NEEDS HUMAN`, and
fires the **Manual Intervention Required** handoff. It never loops forever.

## Customizing models per agent

Set them in `.agent-loop/ralph.config.md` § Models and mirror the value in each agent's `model:`
frontmatter (a string, or a list tried in order). Omit `model:` to inherit the picker
default. Suggested split: strongest reasoning for Researcher/Planner/Reviewer, strong
coding for Executor, faster models for Coordinator/Curator.

## What's new vs ralph-copilot

- **Researcher** and **Curator** agents added (research phase + memory keeping).
- **Knowledge-only memory API**, **shared** across changes with `source_issues` provenance.
- **Per-tracked-change handoff folders** (`.agent-loop/handoff/<ISSUE-ID>/`) + a registry dashboard, keyed by Jira id / slug — concurrent changes stay isolated while sharing one memory.
- **Bounded gates with real human escalation** instead of "never give up."
- **Per-agent model configuration.**
