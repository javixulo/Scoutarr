# UC6-TASK-003 — Identify mode for single-file remap

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC6-TASK-001 (destination validation), UC6-TASK-002 (title cross-check), UC2-TASK-007 (episode filename formatter)

---

## Context

Computes the proposed new name/path for a single-file remap without touching the filesystem, and records the decision in the series' remapping working file so it is preserved and can be reused on a later call.

This is the Identify half of UC-06 (Rename, UC6-TASK-004, applies it to disk).

---

## The remapping working file

Written to the root of the series folder as `{Series Name} ({Year}).remapping.json`, distinct from the series metadata file. Never deleted (kept as an audit trail, per project decision). Example after a couple of identify calls on the same series:

```json
{
  "seriesTmdbId": 1396,
  "seriesName": "Breaking Bad",
  "seriesYear": 2008,
  "entries": [
    {
      "originalPath": "Season 03/Breaking Bad - S03E02 - Caballo Sin Nombre.mkv",
      "status": "resolved",
      "destination": { "season": 1, "episode": 45 },
      "proposedPath": "Season 01/Breaking Bad - S01E45 - Caballo Sin Nombre.mkv",
      "titleCheck": {
        "result": "match",
        "currentTitle": "Caballo Sin Nombre",
        "destinationTitle": "Caballo Sin Nombre"
      },
      "updatedAt": "2026-07-01T10:52:00Z"
    },
    {
      "originalPath": "Season 03/Breaking Bad - S03E05 - Problem Dog.mkv",
      "status": "conflict",
      "destination": { "season": 1, "episode": 48 },
      "proposedPath": "Season 01/Breaking Bad - S01E48 - Problem Dog.mkv",
      "titleCheck": {
        "result": "mismatch",
        "currentTitle": "Problem Dog",
        "destinationTitle": "Hermanos"
      },
      "updatedAt": "2026-07-01T10:53:00Z"
    }
  ]
}
```

Notes:

- `entries` is a list because separate calls against the same series accumulate in the same working file (this structure is also the one UC-07 will reuse for multiple simultaneous files).
- `titleCheck` is the UC6-TASK-002 output embedded as-is.
- No `candidates` field for UC-06 — it's a manual, single-destination operation. That field is introduced by UC-07's `resolve`.
- The file is never deleted, even once the underlying media file has been renamed by UC6-TASK-004.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Identify mode for single-file remap
  As Scoutarr
  I want to compute the proposed new name/path for a remap destination and record it in the working file
  So that the user can preview the result before applying it, and the decision is preserved

  Scenario: Destination is valid and title cross-check matches
    Given a validated remap destination "S01E45" with title "The Beginning"
    And the title cross-check result is a match
    And no "{Series Name} ({Year}).remapping.json" exists yet for this series
    When identify is run
    Then the proposed filename/path is computed for "S01E45"
    And "{Series Name} ({Year}).remapping.json" is created with an entry for this file marked "resolved"

  Scenario: Destination is valid and there is no title to compare
    Given a validated remap destination "S01E45" with title "The Beginning"
    And the title cross-check result is "no title to compare"
    When identify is run
    Then the proposed filename/path is computed for "S01E45"
    And the entry for this file in "{Series Name} ({Year}).remapping.json" is marked "resolved"

  Scenario: Destination is valid but title cross-check is a mismatch
    Given a validated remap destination "S01E45" with title "The Beginning"
    And the title cross-check result is a mismatch, with the file's current title "A New Threat"
    When identify is run
    Then the proposed filename/path is still computed for "S01E45", for preview purposes
    And the entry for this file in "{Series Name} ({Year}).remapping.json" is marked "conflict"
    And the response includes both titles (file's current title and destination's title) so the user can judge the mismatch

  Scenario: Destination validation fails
    Given the remap destination does not exist in the fresh TMDB data
    When identify is run
    Then no filename/path is computed
    And "{Series Name} ({Year}).remapping.json" is not created or modified
    And the result status is "error", with the validation failure detail

  Scenario: Re-running identify on a file already resolved in the working file
    Given "{Series Name} ({Year}).remapping.json" already has an entry for this file marked "resolved"
    When identify is run again for the same file with the same destination
    Then the existing entry is reused rather than recomputed
    And the proposed filename/path from the existing entry is returned

  Scenario: Identify does not touch the media filesystem
    Given any of the scenarios above
    When identify is run
    Then no media file is moved, renamed, or deleted
    And no series metadata file is written or modified
```

---

## Notes for Black Widow

- Cover all six scenarios above.
- Add a case covering a second, different entry being added to an already-existing `remapping.json` for the same series (confirms entries accumulate rather than the file being overwritten wholesale).

## Notes for Tony Stark

- Reuse the episode filename formatter (UC2-TASK-007) to compute `proposedPath`.
- The series metadata file is never written or modified by this task — only by UC6-TASK-004, and only under the conditions already defined for that task.
- Entry matching for the "already resolved" reuse case is by `originalPath` within the working file.

---

## Subtasks

- [ ] Define the remapping working file read/write logic (`{Series Name} ({Year}).remapping.json`)
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements identify mode
- [ ] Hawkeye reviews
