# UC6-TASK-001 — Validate remap destination

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC6-TASK-000 (series resolution)

---

## Context

The user provides an explicit destination season/episode for a file to be remapped. Before anything is computed or moved, Scoutarr must confirm that destination actually exists in the series' current TMDB data — never trusting a possibly-stale local metadata file for this, since the entire premise of remapping is that TMDB's structure has changed since the file was last organised.

This task assumes the series has already been resolved (UC6-TASK-000) and works against a fresh TMDB listing of seasons and episodes for that series.

No episode title comparison here — that is UC6-TASK-002. This task only confirms the destination season/episode exists.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Validate remap destination
  As Scoutarr
  I want to confirm that the season/episode a user specifies as a remap destination actually exists
  So that files are never moved to a destination that doesn't correspond to real TMDB data

  Scenario: Destination season and episode exist
    Given a resolved series with a fresh TMDB listing showing Season 1 with 20 episodes
    And the user specifies destination "S01E10" for a file
    When the destination is validated
    Then the validation succeeds

  Scenario: Destination season does not exist
    Given a resolved series with a fresh TMDB listing showing seasons 1 through 3 only
    And the user specifies destination "S05E01" for a file
    When the destination is validated
    Then the validation fails
    And the error clearly states that season 5 does not exist for this series

  Scenario: Destination season exists but episode number does not
    Given a resolved series with a fresh TMDB listing showing Season 1 with 12 episodes
    And the user specifies destination "S01E20" for a file
    When the destination is validated
    Then the validation fails
    And the error clearly states that episode 20 does not exist in season 1, and mentions the actual episode count for that season

  Scenario: Destination is a specials episode that exists
    Given a resolved series with a fresh TMDB listing showing Season 0 with 3 special episodes
    And the user specifies destination "S00E02" for a file
    When the destination is validated
    Then the validation succeeds
```

---

## Notes for Black Widow

- Mock the fresh TMDB season/episode listing — no real TMDB access.
- Cover all four scenarios above, plus at least one additional case with Season 0 absent entirely (destination "S00E01" requested on a series with no specials) to confirm it fails the same way as any other non-existent season.

## Notes for Tony Stark

- Validation must always use a freshly-fetched TMDB listing, never a cached/local metadata file — that's the whole point of this use case.
- Error messages should mention the actual episode count of the requested season when the season exists but the episode doesn't, per the acceptance criteria.

---

## Subtasks

- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements the validation
- [ ] Hawkeye reviews
