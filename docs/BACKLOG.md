# Scoutarr – Backlog

> Maintained by Nick Fury. Updated at the end of each development cycle.
> Task files live under `docs/tasks/` as `TASK-XXX.md` (UC1) or `UC2-TASK-XXX.md` (UC2 onwards).

---

## Status legend

| Symbol | Status         | Meaning                                                |
| ------ | -------------- | ------------------------------------------------------ |
| ⬜      | Not started    | Concept only, no definition yet                        |
| 🔷      | In refinement  | Actively working on definition                         |
| 🟦      | Refined        | Definition complete, pending project owner sign-off    |
| 🟢      | Ready to start | Project owner approved — agents may pick up            |
| 🔵      | In progress    | An agent is actively working on this                   |
| 🟡      | In review      | Agent workflow complete, awaiting project owner review |
| ✅      | Complete       | Approved, committed, pushed                            |
| 🔴      | Blocked        | Cannot proceed                                         |

---

## UC-01 — Rename a single movie file

| Task     | Title                                    | Status |
| -------- | ---------------------------------------- | ------ |
| [TASK-001](tasks/TASK-001.md) | Movie filename parser                    | 🟦      |
| [TASK-002](tasks/TASK-002.md) | TMDB movie search and confidence scoring | 🟦      |
| [TASK-003](tasks/TASK-003.md) | Output filename formatter (movie)        | 🟦      |
| [TASK-004](tasks/TASK-004.md) | Core orchestration — identify movie      | 🟦      |
| [TASK-005](tasks/TASK-005.md) | REST API — identify and rename movie     | 🟦      |
| [TASK-006](tasks/TASK-006.md) | MCP — identify and rename movie          | 🟦      |

---

## UC-02 — Rename a single TV show episode file

| Task           | Title                                                        | Status |
| -------------- | ------------------------------------------------------------ | ------ |
| [UC2-TASK-001](tasks/UC2-TASK-001.md)   | TV series title parser                                       | 🟦      |
| [UC2-TASK-002](tasks/UC2-TASK-002.md)   | Extend ITmdbClient with TV show methods                      | 🟦      |
| [UC2-TASK-003](tasks/UC2-TASK-003.md)   | TV show search service                                       | 🟦      |
| [UC2-TASK-004](tasks/UC2-TASK-004.md)   | Core orchestration — identify TV series                      | 🟦      |
| [UC2-TASK-005](tasks/UC2-TASK-005.md)   | TV episode number parser (heuristic chain, H1 only)          | 🟦      |
| [UC2-TASK-006](tasks/UC2-TASK-006.md)   | TV episode lookup service                                    | 🟦      |
| [UC2-TASK-007](tasks/UC2-TASK-007.md)   | Output filename formatter (TV episode)                       | 🟦      |
| [UC2-TASK-008](tasks/UC2-TASK-008.md)   | TV episode file move service                                 | 🟦      |
| [UC2-TASK-009](tasks/UC2-TASK-009.md)   | REST API — identify series, identify episode, rename episode | 🟦      |
| [UC2-TASK-010](tasks/UC2-TASK-010.md)   | MCP — identify series, identify episode, rename episode      | 🟦      |

---

## UC-03 — Rename all episodes in a folder (full series or single season)

| Task     | Title                                                  | Status |
| -------- | ------------------------------------------------------ | ------ |
| TASK-012 | Folder structure builder                               | ⬜      |
| TASK-013 | Series metadata file — write and read                  | ⬜      |
| TASK-014 | Core orchestration — identify and rename series folder | ⬜      |
| TASK-015 | REST API — identify and rename series folder           | ⬜      |
| TASK-016 | MCP — identify and rename series folder                | ⬜      |

---

## UC-04 — Subsequent passes on a known series folder

| Task     | Title                                            | Status |
| -------- | ------------------------------------------------ | ------ |
| TASK-017 | Series metadata validation — episode range check | ⬜      |
| TASK-018 | Series metadata refresh — airing seasons         | ⬜      |
| TASK-019 | Core orchestration — subsequent pass             | ⬜      |
| TASK-020 | REST API — subsequent pass                       | ⬜      |
| TASK-021 | MCP — subsequent pass                            | ⬜      |

---

## UC-05 — Add new episode number heuristics

| Task | Title | Status |
| ---- | ----- | ------ |
| ⬜    | To be defined | ⬜ |

---

## Cross-cutting

| Task     | Title                                                          | Status |
| -------- | -------------------------------------------------------------- | ------ |
| TASK-000 | Project bootstrap — solution structure, configuration, logging | ⬜      |
| TASK-022 | CI pipeline (GitHub Actions)                                   | ⬜      |
| TASK-023 | Docker image — volume structure, mount points, Dockerfile      | ⬜      |
