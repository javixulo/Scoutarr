# Scoutarr – Backlog

> Maintained by Nick Fury. Updated at the end of each development cycle.
> Task files live under `docs/tasks/` as `UC1-TASK-XXX.md`, `UC2-TASK-XXX.md`, etc.

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
| -------- | ----------------------------------------- | ------ |
| [UC1-TASK-001](tasks/UC1-TASK-001.md) | Movie filename parser                    | 🟦      |
| [UC1-TASK-002](tasks/UC1-TASK-002.md) | TMDB movie search and confidence scoring | 🟦      |
| [UC1-TASK-003](tasks/UC1-TASK-003.md) | Output filename formatter (movie)        | 🟦      |
| [UC1-TASK-004](tasks/UC1-TASK-004.md) | Core orchestration — identify movie      | 🟦      |
| [UC1-TASK-005](tasks/UC1-TASK-005.md) | REST API — identify and rename movie     | 🟦      |
| [UC1-TASK-006](tasks/UC1-TASK-006.md) | MCP — identify and rename movie          | 🟦      |

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
| [UC3-TASK-001](tasks/UC3-TASK-001.md) | Folder scanner                                         | 🟦      |
| [UC3-TASK-002](tasks/UC3-TASK-002.md) | Series metadata file — write, read, refresh, validate  | 🟦      |
| [UC3-TASK-003](tasks/UC3-TASK-003.md) | Folder rename orchestrator                             | 🟦      |
| [UC3-TASK-004](tasks/UC3-TASK-004.md) | Folder merge logic                                     | 🟦      |
| [UC3-TASK-005](tasks/UC3-TASK-005.md) | REST API — identify_folder, rename_folder              | 🟦      |
| [UC3-TASK-006](tasks/UC3-TASK-006.md) | MCP — identify_folder, rename_folder                   | 🟦      |

---

## UC-04 — Subsequent passes on a known series folder

| Task     | Title                                            | Status |
| -------- | ------------------------------------------------ | ------ |
| [UC4-TASK-001](tasks/UC4-TASK-001.md) | Known series locator                             | 🟦      |
| [UC4-TASK-002](tasks/UC4-TASK-002.md) | Subsequent pass orchestration for single episode | 🟦      |

---

## UC-05 — Add a set of heuristics to improve episode number recognition

| Task     | Title                                                      | Status |
| -------- | ---------------------------------------------------------- | ------ |
| [UC5-TASK-001](tasks/UC5-TASK-001.md) | H2 — Absolute episode number                              | 🟦      |
| [UC5-TASK-002](tasks/UC5-TASK-002.md) | H3 — Special episode identification by file duration      | 🟦      |
| [UC5-TASK-003](tasks/UC5-TASK-003.md) | H4 — Episode identification by air date                  | 🟦      |
| [UC5-TASK-004](tasks/UC5-TASK-004.md) | H5 — Episode identification by title match               | 🟦      |

---

## UC-06 — Remap a single already-organised file to a different season/episode

| Task     | Title                                                      | Status |
| -------- | ------------------------------------------------------------ | ------ |
| [UC6-TASK-000](tasks/UC6-TASK-000.md) | Resolve the series for a remap operation                    | 🟦      |
| [UC6-TASK-001](tasks/UC6-TASK-001.md) | Validate remap destination                                   | 🟦      |
| [UC6-TASK-002](tasks/UC6-TASK-002.md) | Compare current filename title against destination episode title | 🟦      |
| [UC6-TASK-003](tasks/UC6-TASK-003.md) | Identify mode — compute proposed remap filename/path, including `force` confirmation for conflicts | 🟦      |
| [UC6-TASK-005](tasks/UC6-TASK-005.md) | Rename mode — apply the remap on disk                        | 🟦      |
| UC6-TASK-006 | REST API — identify and rename remap                          | ⬜      |
| UC6-TASK-007 | MCP — identify and rename remap                                | ⬜      |

> Task number 004 is intentionally unused: conflict resolution was folded into the `force` confirmation flow within UC6-TASK-003 (Identify) rather than kept as a separate operation.
>
> UC-07 (remap a series folder after a TMDB restructuring) is deliberately parked until UC-06 is complete.

---

## Cross-cutting

| Task     | Title                                                          | Status |
| -------- | -------------------------------------------------------------- | ------ |
| [TASK-000](tasks/TASK-000.md) | Project bootstrap — solution structure, configuration, logging, Docker | 🟦      |
| [TASK-001](tasks/TASK-001.md) | CI/CD pipeline (GitHub Actions + ghcr.io)                     | 🟦      |
