# UC5-TASK-001 — H2: Absolute episode number heuristic

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-005 (IEpisodeHeuristic, EpisodeParseContext, TvEpisodeNumberParser), UC3-TASK-002 (SeriesMetadata, ISeriesMetadataService)

---

## Context

H1 (`SxEyHeuristic`) handles the most common and unambiguous filename format. H2 handles a different class of filenames where the episode is identified by a bare number — either its absolute position across the whole series, or a compact `SxxEyy`-style encoding without separators.

Examples of filenames this heuristic targets:

- `breaking.bad.13.mkv` — episode 13 in absolute terms
- `breaking.bad.13.dead.freight.mkv` — absolute number plus words from the episode title

H2 relies on `context.Metadata` — the series metadata file written by UC3-TASK-002, which contains `AbsoluteEpisodeNumber` precalculated for every episode. If no metadata is available, the heuristic returns `null` and the chain continues.

This task implements `IEpisodeHeuristic` and registers it as H2 in the heuristic chain in `TvEpisodeNumberParser`.

---

## Resolution logic

### Step 1 — Extract a bare number from the filename

A bare number is a token that:
- Consists entirely of digits
- Is not preceded or followed by letters (i.e. it is not part of a word or a year token)
- Does not match the `SxxEyy` pattern already handled by H1

If no bare number is found, return `null`.

### Step 2 — Resolve against metadata

Two interpretations are considered in parallel:

**Interpretation A — absolute episode number:**
Look up the bare number in `AbsoluteEpisodeNumber` across all episodes in the metadata. If a match is found, that is the absolute candidate.

Discard interpretation A if: the bare number exceeds the total episode count of the series.

**Interpretation B — compact SxEy encoding:**
Treat the bare number as `{season}{episode}` concatenated (e.g. `101` → S01E01, `213` → S02E13). Valid only if the season and episode numbers both exist in the metadata.

Discard interpretation B if: the episode portion of the number exceeds the episode count of the candidate season.

### Step 3 — Resolve ambiguity

- If only one interpretation survives → return that result.
- If both interpretations survive → attempt title disambiguation (Step 4).
- If neither interpretation survives → return `null`.

### Step 4 — Title disambiguation

Extract candidate words from the filename by removing:
- The series title tokens
- The bare number
- Common noise tokens (resolution tags, release group suffixes, file extension)

Match the remaining words against the episode titles of each surviving interpretation using a case-insensitive substring or token match. The interpretation whose episode title produces the better match wins.

- If one interpretation matches and the other does not → return the matching interpretation.
- If both match or neither matches → return `null`.

---

## Interface

H2 implements `IEpisodeHeuristic` from UC2-TASK-005:

```
IEpisodeHeuristic

TryParse(EpisodeParseContext context)
    → ParsedEpisodeNumber?
```

H2 only uses `context.Filename` and `context.Metadata` from the context (it does not read `context.FileDuration`). When `context.Metadata` is `null`, H2 returns `null` immediately.

`TvEpisodeNumberParser` injects H2 after H1 in its ordered heuristic list.

---

## Notes for Black Widow

- All tests are pure — populate `EpisodeParseContext` directly with mock `SeriesMetadata`, no filesystem or TMDB calls.
- Test no bare number in filename → `null`.
- Test no metadata available → `null`.
- Test bare number exceeds total episode count — interpretation A discarded, interpretation B resolves → result is B.
- Test episode portion of compact number exceeds season episode count — interpretation B discarded, interpretation A resolves → result is A.
- Test both interpretations survive, no title words in filename → `null`.
- Test both interpretations survive, title words match interpretation A only → result is A.
- Test both interpretations survive, title words match interpretation B only → result is B.
- Test both interpretations survive, title words match both → `null`.
- Test H2 is registered after H1 in `TvEpisodeNumberParser` — verify ordering in unit tests.

## Notes for Tony Stark

- Implement `AbsoluteEpisodeNumberHeuristic` in `Scoutarr.Core`.
- `TryParse` must be synchronous — no async, no I/O.
- Bare number extraction must not match year-like tokens (4-digit numbers in the range 1900–2100) to avoid false positives on filenames like `breaking.bad.2008.mkv`.
- Title token matching should be lenient: match if at least one meaningful word (>3 characters) from the filename appears in the episode title.
- Register H2 immediately after H1 in the DI registration of `TvEpisodeNumberParser`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: H2 — Absolute episode number heuristic
  As the TvEpisodeNumberParser
  I want to resolve a bare number in a filename to a season and episode
  Using the series metadata and optionally the episode title

  Scenario: Bare number resolves unambiguously as absolute — season 1 too short
    Given the series has 80 episodes in total
    And season 1 has 10 episodes
    And the episode with AbsoluteEpisodeNumber 13 is S02E03
    And the filename is "breaking.bad.13.mkv"
    When H2 is applied
    Then the result is S02E03

  Scenario: Bare number is ambiguous — season 1 has enough episodes to absorb it
    Given the series has 80 episodes in total
    And season 1 has 13 episodes
    And the episode with AbsoluteEpisodeNumber 13 is S01E13
    And the filename is "breaking.bad.13.mkv"
    When H2 is applied
    Then the result is null

  Scenario: Bare number resolves unambiguously as absolute, title confirms
    Given the series has 80 episodes in total
    And season 1 has 10 episodes
    And the episode with AbsoluteEpisodeNumber 13 is S02E03 with title "Dead Freight"
    And the filename is "breaking.bad.13.dead.freight.mkv"
    When H2 is applied
    Then the result is S02E03

  Scenario: Bare number is ambiguous, title disambiguates toward absolute
    Given the series has 120 episodes in total
    And season 1 has 13 episodes
    And the episode with AbsoluteEpisodeNumber 101 is S08E05 with title "Ozymandias"
    And S01E01 has title "Pilot"
    And the filename is "breaking.bad.101.ozymandias.mkv"
    When H2 is applied
    Then the result is S08E05

  Scenario: Bare number is ambiguous, title does not disambiguate
    Given the series has 120 episodes in total
    And season 1 has 13 episodes
    And the filename is "breaking.bad.101.mkv"
    When H2 is applied
    Then the result is null

  Scenario: Bare number exceeds total episodes — resolves as compact SxEy
    Given the series has 80 episodes in total
    And season 1 has 13 episodes
    And S01E01 exists in metadata with title "Pilot"
    And the filename is "breaking.bad.101.mkv"
    When H2 is applied
    Then the result is S01E01

  Scenario: No metadata available
    Given no metadata is available for the series
    And the filename is "breaking.bad.13.mkv"
    When H2 is applied
    Then the result is null

  Scenario: Filename contains no bare number
    Given the filename is "breaking.bad.mkv"
    When H2 is applied
    Then the result is null
```

---

## Subtasks

- [ ] Implement `AbsoluteEpisodeNumberHeuristic` in `Scoutarr.Core`
- [ ] Register H2 after H1 in `TvEpisodeNumberParser` DI registration
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `AbsoluteEpisodeNumberHeuristic`
- [ ] Hawkeye reviews
