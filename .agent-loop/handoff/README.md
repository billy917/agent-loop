# Handoff & Progress (NOT memory) — per tracked change

Everything the agents pass between each other, and all live status, lives here — kept
separate from **memory**, which holds only durable project knowledge and is reached solely
through the Memory API (`.agent-loop/memory/memory-api.md`).

**One folder per tracked change**, keyed by a Jira id or slug (e.g. `PROJ-123/` or
`add-oauth-login/`). This isolates concurrent changes: `PROJ-123`'s working state never
collides with `PROJ-456`'s. See `INDEX.md` for the registry, status values, and the id
convention.

```
.agent-loop/handoff/
├── INDEX.md                       # registry / dashboard of all tracked changes
├── _templates/iteration-log.md    # copied per iteration (reserved name)
└── <ISSUE-ID>/                    # created at runtime by the Researcher
    ├── RESEARCH.md   PRD.md   PROGRESS.md   scratch.md
    └── iterations/<date>-task-0XX.md
```

| File | Written by | Purpose |
|---|---|---|
| `INDEX.md` | Researcher/Planner/Coordinator | Cross-change registry + status |
| `<ID>/RESEARCH.md` | Researcher | Research summary handed to the Planner |
| `<ID>/PRD.md` | Planner | Plan: tasks + acceptance criteria + gates |
| `<ID>/PROGRESS.md` | Planner (seed), Executor, Coordinator | Live status, blockers, human-intervention surface |
| `<ID>/iterations/<date>-task-0XX.md` | Executor | Per-iteration log + change manifest + learnings |
| `<ID>/scratch.md` | any agent | Current-task scratch; overwrite/clear freely |

**Memory is shared, not per-issue.** These files are transient per-change state. The
**durable** knowledge inside them (domain/business/technical learnings) is promoted into
shared memory by the Curator via the Memory API's `DISTILL`, tagged with `source_issues`
provenance so any change benefits from what earlier changes learned. How that knowledge is
physically stored is the memory implementation's concern, not this folder's.

Every file carries `issue: <ISSUE-ID>` in its frontmatter, so it is self-identifying.
