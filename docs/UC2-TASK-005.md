# UC2-TASK-005 — TV episode number parser

**Requirements:** [identification.md](identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-004 (TvShowIdentificationSuccess)

---

## Context

Given a dirty TV episode filename, extract the season number and episode number.

This task runs **after** the series has been identified and accepted (UC2-TASK-004).

The parser is designed as a chain of heuristics. Each heuristic implements a common interface and is tried in priority order — the first one that produces a result wins. This makes adding new heuristics in the future a matter of implementing the interface and registering the new heuristic in the chain, with no changes to the orchestration logic.

**This task implements Heuristic 1 only.** Heuristic 2 (compact numeric with TMDB validation) is a future use case and is out of scope here.

No series identification. No TMDB calls. No confidence scoring. No output formatting. No file renaming.

---

## Output types

```
ParsedEpisodeNumber
{
    Season:   int
    Episode:  int
}
```

On failure: a domain error when no heuristic produces a result.

---

## Interface design

```
IEpisodeHeuristic

TryParse(string filename)
    → ParsedEpisodeNumber?    // null if this heuristic does not match

ITvEpisodeNumberParser

ParseAsync(string filename, CancellationToken ct)
    → ParsedEpisodeNumber | TvEpisodeParseError
```

`TvEpisodeNumberParser` holds an ordered `IReadOnlyList<IEpisodeHeuristic>` injected via constructor. It iterates the list and returns the first non-null result. If no heuristic matches, it returns a domain error.

Adding a new heuristic in the future requires only: implement `IEpisodeHeuristic`, register it in DI in the correct position in the list.

---

## Heuristic 1 — Explicit SxEy pattern

Matches `S01E02`, `S1E2`, `s01e02`, `s1e2`, and similar variants. Case-insensitive. Zero-padded and non-padded season and episode numbers are both valid.

This heuristic is pure — no external dependencies.

---

## Notes for Black Widow

- All tests are pure — no mocks needed for this task.
- Test the full range of SxEy variants: uppercase, lowercase, zero-padded, non-padded.
- Test high season and episode numbers.
- Test season 0.
- Test that a year token (`2005`) is not mistaken for an episode pattern.
- Test that a filename with no matching pattern returns a domain error.
- Test the chain itself: verify that if Heuristic 1 returns null, the chain moves to the next heuristic (use a stub second heuristic that always returns a fixed result).
- Test that if Heuristic 1 returns a result, subsequent heuristics in the chain are never called.

## Notes for Tony Stark

- Define `IEpisodeHeuristic` interface in `Scoutarr.Core`.
- Implement `SxEyHeuristic` as the first implementation of `IEpisodeHeuristic`.
- Implement `TvEpisodeNumberParser` with an injected `IReadOnlyList<IEpisodeHeuristic>`.
- Register `SxEyHeuristic` as the sole heuristic in DI for now.
- `TryParse` on `SxEyHeuristic` is synchronous and pure — no async, no dependencies.
- `ParseAsync` on `TvEpisodeNumberParser` is async to accommodate future heuristics that may need async operations (e.g. TMDB calls in Heuristic 2).

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV episode number parser
  As the Core episode identification service
  I want to extract season and episode number from a dirty filename
  So that the episode can be looked up on TMDB for the confirmed series

  # ─────────────────────────────────────────
  # HEURISTIC 1 — SxEy pattern
  # ─────────────────────────────────────────

  Scenario: Standard zero-padded uppercase pattern
    Given the filename "Breaking.Bad.S01E02.720p.mkv"
    When the episode number is parsed
    Then the season is 1 and the episode is 2

  Scenario: Non-padded season and episode
    Given the filename "Breaking.Bad.S1E2.mkv"
    When the episode number is parsed
    Then the season is 1 and the episode is 2

  Scenario: Lowercase pattern
    Given the filename "breaking.bad.s01e02.mkv"
    When the episode number is parsed
    Then the season is 1 and the episode is 2

  Scenario: Underscores as separators
    Given the filename "Breaking_Bad_S02E05_1080p.mkv"
    When the episode number is parsed
    Then the season is 2 and the episode is 5

  Scenario: High season and episode numbers
    Given the filename "The.Simpsons.S33E10.mkv"
    When the episode number is parsed
    Then the season is 33 and the episode is 10

  Scenario: Season 0 via SxEy pattern
    Given the filename "Breaking.Bad.S00E03.mkv"
    When the episode number is parsed
    Then the season is 0 and the episode is 3

  Scenario: Year token is not mistaken for an episode pattern
    Given the filename "The.Office.2005.S03E01.mkv"
    When the episode number is parsed
    Then the season is 3 and the episode is 1

  # ─────────────────────────────────────────
  # HEURISTIC CHAIN BEHAVIOUR
  # ─────────────────────────────────────────

  Scenario: First heuristic matches — subsequent heuristics are not called
    Given the filename "Breaking.Bad.S01E02.mkv"
    And a second heuristic is registered after Heuristic 1
    When the episode number is parsed
    Then the second heuristic is never called

  Scenario: First heuristic does not match — chain moves to next heuristic
    Given the filename "Some.Show.without.sxey.pattern.mkv"
    And a stub second heuristic that always returns S2E5
    When the episode number is parsed
    Then the season is 2 and the episode is 5

  # ─────────────────────────────────────────
  # NEGATIVE CASES
  # ─────────────────────────────────────────

  Scenario: No heuristic matches — domain error returned
    Given the filename "Some.Random.File.mkv"
    And no heuristic in the chain matches
    When the episode number is parsed
    Then a domain error is returned with reason "No episode pattern found"
```

---

## Subtasks

- [ ] Define `IEpisodeHeuristic` interface in `Scoutarr.Core`
- [ ] Define `ParsedEpisodeNumber` record in `Scoutarr.Core`
- [ ] Define `ITvEpisodeNumberParser` interface in `Scoutarr.Core`
- [ ] Implement `SxEyHeuristic`
- [ ] Implement `TvEpisodeNumberParser` with heuristic chain
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `SxEyHeuristic` and `TvEpisodeNumberParser`
- [ ] Hawkeye reviews
