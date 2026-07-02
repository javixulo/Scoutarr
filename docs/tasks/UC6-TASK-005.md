# UC6-TASK-005 — Rename mode for single-file remap

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC6-TASK-003 (identify mode, including the `force` confirmation flow for conflicts)

---

## Context

Applies a remap that has already been computed by Identify (UC6-TASK-003) to disk: moves and renames the video file and its subtitles to the proposed destination.

A `conflict` entry (title mismatch) cannot be applied directly. It must first be resolved by calling Identify again with the exact same destination and `force: true` (see UC6-TASK-003) — there is no separate "resolve" operation. This task only covers applying entries that are already `resolved`, and rejecting attempts to apply an unresolved `conflict`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Rename mode for single-file remap
  As Scoutarr
  I want to apply a resolved remap to disk
  So that the file physically lives where it now belongs according to TMDB's current structure

  Scenario: Applying a resolved entry
    Given "{Series Name} ({Year}).remapping.json" has an entry for this file marked "resolved"
    When rename is run for this file
    Then the video file and its associated subtitles are moved/renamed to the proposed destination
    And existing no-overwrite/merge behaviour applies at the destination (same as UC3-TASK-004)
    And the series metadata file is left untouched
    And the entry's status is updated to "applied"

  Scenario: Attempting to apply a conflict entry
    Given "{Series Name} ({Year}).remapping.json" has an entry for this file marked "conflict"
    When rename is run for this file
    Then the operation fails
    And the error clearly states the title mismatch and that identify must be called again with the same destination and "force: true" first
    And no file is moved

  Scenario: Rename runs identify first when no working file entry exists yet
    Given no entry exists yet for this file in "{Series Name} ({Year}).remapping.json"
    And the user provides a destination directly to rename
    When rename is run
    Then identify is executed first (as already established for the rest of the system)
    And if the resulting entry is "resolved", the move proceeds
    And if the resulting entry is "conflict", the operation fails per the previous scenario

  Scenario: Destination already occupied at the target path
    Given a resolved entry whose proposed destination file already exists on disk
    When rename is run for this file
    Then the operation is aborted
    And an error is returned
    And nothing is moved
    And the entry's status is left as "resolved" (not "applied")

  Scenario: Working file entries are never deleted
    Given any of the scenarios above
    When the operation completes, regardless of outcome
    Then the entry remains present in "{Series Name} ({Year}).remapping.json", only its status changes
```

---

## Notes for Black Widow

- Cover all five scenarios above.
- Reuse the existing no-overwrite/merge test fixtures from UC3-TASK-004 rather than duplicating them.

## Notes for Tony Stark

- Reuse `TvEpisodeMoveService` / merge logic from UC3-TASK-004 as-is for the actual move.
- The series metadata file is never written or modified by this task.
- Status transition on success is `resolved` → `applied`. A `conflict` entry must go through the `force` confirmation flow in Identify (UC6-TASK-003) first; this task does not resolve conflicts itself.

---

## Subtasks

- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements rename mode
- [ ] Hawkeye reviews
