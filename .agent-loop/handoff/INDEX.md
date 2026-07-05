---
id: handoff-index
updated: 2026-07-04
---

# Tracked Changes Registry

One row per functional change (Jira id or slug). This is the cross-change dashboard.
Each change owns a folder `.agent-loop/handoff/<ISSUE-ID>/`; **memory is shared project-wide** and
is not per-issue.

| Issue | Title | Status | Path | Updated |
|---|---|---|---|---|
| _(none yet)_ | | | | |

## Status values
`RESEARCH` → `PLANNED` → `ACTIVE` → `COMPLETE`  (or `HALTED — NEEDS HUMAN` at any loop point)

Set by: Researcher (adds row, `RESEARCH`) · Planner (`PLANNED`) · Coordinator
(`ACTIVE` on start; `HALTED — NEEDS HUMAN` or `COMPLETE` at end).

## Issue id convention (filesystem-safe folder key)
- **Jira:** `^[A-Z][A-Z0-9]+-[0-9]+$` → e.g. `PROJ-123`
- **Slug:** `^[a-z0-9]+(-[a-z0-9]+)*$` → e.g. `add-oauth-login`
- No spaces, slashes, or quotes. If given only a title, derive a slug.
- Reserved names (never an issue id): `INDEX.md`, `_templates/`.

## Per-change folder contents
```
.agent-loop/handoff/<ISSUE-ID>/
├── RESEARCH.md          # Researcher → Planner
├── PRD.md               # the plan: tasks + acceptance criteria + gates
├── PROGRESS.md          # live status board + blockers + human-intervention surface
├── scratch.md           # current-task scratch
└── iterations/<YYYY-MM-DD>-task-0XX.md
```
Every file carries `issue: <ISSUE-ID>` in its frontmatter, so it is self-identifying.
