# UC6-TASK-003 — Identify mode for single-file remap

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC6-TASK-001 (destination validation), UC6-TASK-002 (title cross-check), UC2-TASK-005 (episode number parser, used to derive `origin` from the file's current path/filename), UC2-TASK-007 (episode filename formatter)

---

## Context

Computes the proposed new name/path for a single-file remap without touching the filesystem, and records the decision in the series' remapping working file so it is preserved and can be reused on a later call.

This is the Identify half of UC-06 (Rename, UC6-TASK-005, applies it to disk).

A `remapping.json` entry is always per file, not per (file, destination) — each file has at most one active entry. Calling identify again with a different destination overwrites the existing entry.

A `conflict` (title mismatch) is never resolved implicitly by repeating the same call. It requires an explicit `force: true` on a call using the exact same destination that produced the conflict. `force` has no effect on a different destination — a different destination is always evaluated fresh, and can itself produce a new `conflict` if its own title cross-check fails. `force` with no matching prior `conflict` entry to confirm is a usage error.

If the requested destination is identical to the file's current (origin) season/episode, there is nothing to remap — this is a `noop`, resolved before any title cross-check.

---

## Deriving `origin`

`origin` (the season/episode the file currently has) is derived once per file, by applying the existing episode number parser (UC2-TASK-005, heuristic H1) to the file's current path/filename. This is not new parsing logic — it is the same parser already used elsewhere in the system, applied here to a filename that is presumed already well-formed (the file was correctly organised at some point in the past).

If no season/episode can be parsed from the current path/filename, the file's origin cannot be established and the operation fails outright — no entry is created.

---

## The remapping working file

Written to the root of the series folder as `{Series Name} ({Year}).remapping.json`, distinct from the series metadata file. Never deleted (kept as an audit trail, per project decision). Example after an identify call:

```json
{
  "seriesTmdbId": 1396,
  "seriesName": "Breaking Bad",
  "seriesYear": 2008,
  "entries": [
    {
      "originalPath": "Season 03/Breaking Bad - S03E10 - The Beginning.mkv",
      "origin": { "season": 3, "episode": 10 },
      "status": "resolved",
      "destination": { "season": 1, "episode": 45 },
      "proposedPath": "Season 01/Breaking Bad - S01E45 - The Beginning.mkv",
      "titleCheck": {
        "result": "match",
        "currentTitle": "The Beginning",
        "destinationTitle": "The Beginning"
      },
      "updatedAt": "2026-07-01T10:52:00Z"
    }
  ]
}
```

Notes:

- `origin` records the season/episode the file actually had before any remap — parsed once from its current path/filename — and never changes for a given file, even if the entry is later overwritten with a different destination.
- `entries` is a list because separate calls against the same series accumulate in the same working file (this structure is also the one UC-07 will reuse for multiple simultaneous files).
- `titleCheck` is the UC6-TASK-002 output embedded as-is. Absent (`null`) for `noop` entries, since no cross-check is performed.
- No `candidates` field for UC-06 — it's a manual, single-destination operation. That field is introduced by UC-07's `resolve`.
- The file is never deleted, even once the underlying media file has been renamed by UC6-TASK-005.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Identify mode for single-file remap
  As Scoutarr
  I want to compute the proposed new name/path for a remap destination and record it in the working file
  So that the user can preview the result before applying it, with explicit confirmation required for conflicts

  Scenario: Destination is valid and title cross-check matches
    Given remap is called for the file "Season 03/Breaking Bad - S03E10 - The Beginning.mkv"
    And the destination provided is "S01E45"
    And "S01E45" is validated against the fresh TMDB listing and exists, with title "The Beginning"
    And the title cross-check between "The Beginning" (current) and "The Beginning" (destination) is a match
    And no "Breaking Bad (2008).remapping.json" exists yet
    When identify is run
    Then the proposed path is computed as "Season 01/Breaking Bad - S01E45 - The Beginning.mkv"
    And "Breaking Bad (2008).remapping.json" is created with an entry for this file
    And the entry's status is "resolved"
    And the entry's originalPath is "Season 03/Breaking Bad - S03E10 - The Beginning.mkv"
    And the entry's origin is season 3, episode 10
    And the entry's destination is season 1, episode 45

  Scenario: Destination is valid and there is no title to compare
    Given remap is called for the file "Season 03/Breaking Bad - S03E10.mkv"
    And the destination provided is "S01E45"
    And "S01E45" is validated against the fresh TMDB listing and exists, with title "The Beginning"
    And the current filename has no title to extract
    When identify is run
    Then the title cross-check result is "no title to compare"
    And the entry's status is "resolved"
    And the entry's origin is season 3, episode 10
    And the entry's destination is season 1, episode 45

  Scenario: Destination is valid but title cross-check is a mismatch (first attempt)
    Given remap is called for the file "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv"
    And the destination provided is "S01E45"
    And "S01E45" is validated against the fresh TMDB listing and exists, with title "The Beginning"
    And no prior entry exists for this file in "Breaking Bad (2008).remapping.json"
    When identify is run
    Then the title cross-check between "Problem Dog" (current) and "The Beginning" (destination) is a mismatch
    And the proposed path "Season 01/Breaking Bad - S01E45 - The Beginning.mkv" is still computed, for preview purposes
    And the entry's status is "conflict"
    And the entry's destination is season 1, episode 45
    And the response includes both "Problem Dog" and "The Beginning" so the user can judge the mismatch

  Scenario: Re-running identify with the same conflicted destination, without confirming, does not resolve it
    Given "Breaking Bad (2008).remapping.json" has an entry for "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv"
    And that entry's status is "conflict" with destination season 1, episode 45
    When identify is run again for the same file with the same destination "S01E45" and no confirmation flag
    Then the entry's status remains "conflict"
    And the entry's destination remains season 1, episode 45

  Scenario: Re-running identify with the same conflicted destination and force=true confirms it
    Given "Breaking Bad (2008).remapping.json" has an entry for "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv"
    And that entry's status is "conflict" with destination season 1, episode 45
    When identify is run again for the same file with the same destination "S01E45" and "force" set to true
    Then the entry's status changes to "resolved"
    And the entry's destination remains season 1, episode 45
    And the entry's origin remains season 3, episode 10

  Scenario: Force is ignored when the destination differs from the existing conflict
    Given "Breaking Bad (2008).remapping.json" has an entry for "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv"
    And that entry's status is "conflict" with destination season 1, episode 45
    When identify is run again for the same file with a different destination "S01E12" and "force" set to true
    Then "force" has no effect, since it does not match the existing conflicted destination
    And the entry is evaluated fresh against "S01E12", exactly as in an unforced call
    And if the title cross-check for "S01E12" is also a mismatch, the resulting status is "conflict" (not "resolved")

  Scenario: Force with no prior conflict to confirm is a usage error
    Given no entry exists yet for "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv" in "Breaking Bad (2008).remapping.json"
    When identify is run for the first time for this file with destination "S01E45" and "force" set to true
    Then the operation fails
    And the error explains that identify must be called without "force" first, before a conflict can be confirmed
    And no entry is created

  Scenario: Re-running identify with a different destination overrides a conflicted entry
    Given "Breaking Bad (2008).remapping.json" has an entry for "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv"
    And that entry's status is "conflict" with destination season 1, episode 45
    When identify is run again for the same file with a different destination "S01E12"
    Then the entry for this file is overwritten
    And the title cross-check and status are recomputed from scratch against "S01E12"
    And the entry's origin remains season 3, episode 10 (unchanged — the file's real origin never changes)

  Scenario: Destination equals the file's current origin (no-op)
    Given remap is called for the file "Season 01/Breaking Bad - S01E45 - The Beginning.mkv"
    And the destination provided is "S01E45", the same as the file's current origin
    When identify is run
    Then no title cross-check is performed
    And the entry's status is "noop"
    And the entry's proposedPath equals its originalPath

  Scenario: Destination validation fails
    Given remap is called for the file "Season 03/Breaking Bad - S03E10.mkv"
    And the destination provided is "S09E01"
    And season 9 does not exist in the fresh TMDB listing for this series
    When identify is run
    Then no proposed path is computed
    And "Breaking Bad (2008).remapping.json" is not created or modified
    And the result status is "error", with the validation failure detail

  Scenario: Origin cannot be derived from the current filename
    Given remap is called for the file "Season 03/13.mkv"
    And the episode number parser cannot extract a season/episode from this path
    When identify is run
    Then the operation fails
    And the error explains that the file's current season/episode could not be determined
    And "Breaking Bad (2008).remapping.json" is not created or modified

  Scenario: Re-running identify on a file already resolved with the same destination
    Given "Breaking Bad (2008).remapping.json" has an entry for "Season 03/Breaking Bad - S03E10 - The Beginning.mkv"
    And that entry's status is "resolved" with destination season 1, episode 45
    When identify is run again for the same file with the same destination "S01E45"
    Then the existing entry is reused rather than recomputed
    And the proposed path from the existing entry is returned unchanged

  Scenario: Identify does not touch the media filesystem
    Given any of the scenarios above
    When identify is run
    Then no media file is moved, renamed, or deleted
    And no series metadata file is written or modified
```

---

## Notes for Black Widow

- Cover all thirteen scenarios above.
- Add a case covering a second, different entry being added to an already-existing `remapping.json` for the same series (confirms entries accumulate rather than the file being overwritten wholesale).

## Notes for Tony Stark

- Reuse the episode number parser (UC2-TASK-005) to derive `origin` — do not implement a second parser.
- Reuse the episode filename formatter (UC2-TASK-007) to compute `proposedPath`.
- The series metadata file is never written or modified by this task — only by UC6-TASK-005, and only under the conditions already defined for that task.
- Entry matching for reuse/overwrite/force logic is by `originalPath` within the working file.
- `force` is a same-destination confirmation, not a general override — validate that the provided destination matches the existing conflicted entry's destination before honouring it, and reject it outright if no such entry exists yet.
- `noop` is checked first, before destination validation against title or any TMDB title cross-check — it only needs `origin == destination`, which is cheap to compute.

---

## Subtasks

- [ ] Define the remapping working file read/write logic (`{Series Name} ({Year}).remapping.json`)
- [ ] Wire in the episode number parser (UC2-TASK-005) to derive `origin`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements identify mode
- [ ] Hawkeye reviews
