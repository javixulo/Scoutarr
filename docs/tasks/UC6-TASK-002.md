# UC6-TASK-002 — Title cross-check for remap destination

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC6-TASK-001 (destination validation), UC5-TASK-004 (H5 title normalisation)

---

## Context

The user provides an explicit destination for a remap, but a manually-entered destination can still be wrong. If the current filename encodes an episode title, comparing it against the title of the destination episode in fresh TMDB data is a cheap, strong signal that the requested destination doesn't correspond to the episode actually contained in the file.

This task only produces the comparison result. It does not decide what happens with a mismatch — that is the responsibility of Identify/Rename (UC6-TASK-003/004).

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Title cross-check for remap destination
  As Scoutarr
  I want to compare the episode title encoded in the current filename against the title of the destination episode
  So that an obviously wrong manual destination gets flagged before anything is moved

  Scenario: Current filename encodes a title that matches the destination episode
    Given a file named "Series Name - S03E02 - The Beginning.mkv"
    And the destination episode "S01E45" has title "The Beginning" in the fresh TMDB data
    When the title cross-check is performed
    Then the result is a match

  Scenario: Current filename encodes a title that does not match the destination episode
    Given a file named "Series Name - S03E02 - The Beginning.mkv"
    And the destination episode "S01E45" has title "A New Threat" in the fresh TMDB data
    When the title cross-check is performed
    Then the result is a mismatch

  Scenario: Title match is case-insensitive and tolerant of punctuation differences
    Given a file named "Series Name - S03E02 - the beginning!.mkv"
    And the destination episode "S01E45" has title "The Beginning" in the fresh TMDB data
    When the title cross-check is performed
    Then the result is a match

  Scenario: Current filename does not encode a title at all
    Given a file named "Series Name - S03E02.mkv"
    And the destination episode "S01E45" has title "The Beginning" in the fresh TMDB data
    When the title cross-check is performed
    Then the result is "no title to compare"

  Scenario: Destination episode has no title in TMDB data
    Given a file named "Series Name - S03E02 - The Beginning.mkv"
    And the destination episode "S01E45" has no title in the fresh TMDB data
    When the title cross-check is performed
    Then the result is "no title to compare"
```

---

## Notes for Black Widow

- Cover all five scenarios above.
- Add a couple of additional normalisation edge cases already covered by the H5 test suite (accents, extra whitespace) to confirm the same normalisation logic is genuinely reused, not reimplemented.

## Notes for Tony Stark

- Reuse the title normalisation/comparison logic already implemented for H5 (UC5-TASK-004) — do not implement a second normalisation routine.
- Output is a simple three-way result: `Match`, `Mismatch`, `NoTitleToCompare`. No confidence score needed here (unlike H5's own use in identification).

---

## Subtasks

- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements the cross-check, reusing H5 normalisation
- [ ] Hawkeye reviews
