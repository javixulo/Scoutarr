# Scoutarr – Backlog

> Maintained by Nick Fury. Updated at the end of each development cycle.
> Task files live at the root of the repository as `TASK-XXX.md`.

---

## Status legend

| Symbol | Status | Meaning |
|---|---|---|
| ⬜ | Not started | Concept only, no definition yet |
| 🔷 | In refinement | Actively working on definition |
| 🟦 | Refined | Definition complete, pending project owner sign-off |
| 🟢 | Ready to start | Project owner approved — agents may pick up |
| 🔵 | In progress | An agent is actively working on this |
| 🟡 | In review | Agent workflow complete, awaiting project owner review |
| ✅ | Complete | Approved, committed, pushed |
| 🔴 | Blocked | Cannot proceed |

---

## UC-01 — Rename a single movie file

| Task | Title | Status |
|---|---|---|
| TASK-001 | Movie filename parser | 🟦 |
| TASK-002 | TMDB movie search and confidence scoring | 🟦 |
| TASK-003 | Output filename formatter (movie) | 🟦 |
| TASK-004 | Core orchestration — identify movie | 🟦 |
| TASK-005 | REST API — identify and rename movie | 🟦 |
| TASK-006 | MCP — identify and rename movie | 🟦 |

---

## UC-02 — Rename a single TV show episode file

| Task | Title | Status |
|---|---|---|
| TASK-007 | TV episode filename parser | ⬜ |
| TASK-008 | TMDB TV show search and confidence scoring | ⬜ |
| TASK-009 | Core orchestration — identify TV episode | ⬜ |
| TASK-010 | REST API — identify and rename TV episode | ⬜ |
| TASK-011 | MCP — identify and rename TV episode | ⬜ |

---

## UC-03 — Rename all episodes in a folder (full series or single season)

| Task | Title | Status |
|---|---|---|
| TASK-012 | Folder structure builder | ⬜ |
| TASK-013 | Series metadata file — write and read | ⬜ |
| TASK-014 | Core orchestration — identify and rename series folder | ⬜ |
| TASK-015 | REST API — identify and rename series folder | ⬜ |
| TASK-016 | MCP — identify and rename series folder | ⬜ |

---

## UC-04 — Subsequent passes on a known series folder

| Task | Title | Status |
|---|---|---|
| TASK-017 | Series metadata validation — episode range check | ⬜ |
| TASK-018 | Series metadata refresh — airing seasons | ⬜ |
| TASK-019 | Core orchestration — subsequent pass | ⬜ |
| TASK-020 | REST API — subsequent pass | ⬜ |
| TASK-021 | MCP — subsequent pass | ⬜ |

---

## Cross-cutting

| Task | Title | Status |
|---|---|---|
| TASK-000 | Project bootstrap — solution structure, configuration, logging | ⬜ |
| TASK-022 | CI pipeline (GitHub Actions) | ⬜ |
| TASK-023 | Docker image and docker-compose | ⬜ |
