# UC5-TASK-004 — H5: Episode identification by title match

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-005 (IEpisodeHeuristic, EpisodeParseContext, TvEpisodeNumberParser), UC3-TASK-002 (SeriesMetadata, EpisodeMetadata)

---

## Context

Some filenames contain the episode title — or words from it — without any numeric identifier. H5 targets this case by matching words extracted from the filename against episode titles in `SeriesMetadata`.

Examples of filenames this heuristic targets:

- `breaking.bad.ozymandias.mkv` — full episode title, single distinctive word
- `breaking.bad.box.cutter.mkv` — full episode title, two words
- `breaking.bad.bit.by.a.dead.bee.mkv` — title with articles/prepositions

H5 applies to all seasons including Season 0.

H5 requires all significant words from the episode title to appear in the filename. This strict requirement reduces false positives on short or common words. If no match is found with all words, the search is retried ignoring articles and short prepositions.

If no metadata is available, or no meaningful words remain after noise removal, H5 returns `null` and the chain continues.

This task implements `IEpisodeHeuristic` and registers it as H5 in the heuristic chain in `TvEpisodeNumberParser`.

---

## Resolution logic

### Step 1 — Guard checks

- If `context.Metadata` is null → return `null`.
- Extract candidate words from `context.Filename` by removing:
  - The series title tokens
  - Common noise tokens (resolution tags, codec tags, release group suffixes, file extension)
- If no meaningful words remain (words of > 3 characters, or any non-noise word) → return `null`.

### Step 2 — Full title match

For each episode across all seasons in `SeriesMetadata`:
- Tokenise the episode title into words.
- Check that every word in the episode title appears in the filename candidate words (case-insensitive).
- Collect all episodes that fully match.

If exactly one episode matches → return it.
If multiple episodes match → return `null`.
If no episodes match → proceed to Step 3.

### Step 3 — Retry without articles and prepositions

Repeat Step 2, but filter out articles and short prepositions from the episode title before matching.

The stop word list is a `static readonly IReadOnlySet<string>` defined in `TitleMatchHeuristic`. It includes at minimum: `a`, `an`, `the`, `of`, `in`, `on`, `at`, `to`, `by`, `as`, `is`, `it`, `or`, `and`, `but`, `for`, `nor`, `so`, `yet`.

If exactly one episode matches → return it.
If multiple episodes match or no episodes match → return `null`.

---

## Interface

H5 implements `IEpisodeHeuristic` from UC2-TASK-005:

```
IEpisodeHeuristic

TryParse(EpisodeParseContext context)
    → ParsedEpisodeNumber?
```

`TvEpisodeNumberParser` injects H5 after H4 in its ordered heuristic list.

---

## Notes for Black Widow

- All tests are pure — populate `EpisodeParseContext` directly with mock `SeriesMetadata`. No filesystem or TMDB calls.
- Test null metadata → `null`.
- Test filename with only noise tokens after series title removal → `null`.
- Test all title words match, single candidate → returns that episode.
- Test all title words match, multiple candidates → `null`.
- Test no full match, retry without stop words finds single candidate → returns that episode.
- Test no full match, retry without stop words finds multiple candidates → `null`.
- Test no words in filename match any title → `null`.
- Test match works across all seasons including Season 0.
- Test H5 is registered after H4 in `TvEpisodeNumberParser` — verify ordering in unit tests.

## Notes for Tony Stark

- Implement `TitleMatchHeuristic` in `Scoutarr.Core`.
- `TryParse` must be synchronous — no async, no I/O.
- Word comparison is case-insensitive and ignores punctuation (strip apostrophes, hyphens, etc. before comparing).
- The stop word list is `static readonly IReadOnlySet<string>` — easy to extend by adding entries.
- Noise token removal (resolution tags, codec tags, release group suffixes) should reuse the same utility used by H2, H3, and H4 to avoid duplication.
- Register H5 immediately after H4 in the DI registration of `TvEpisodeNumberParser`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: H5 — Episode identification by title match
  As the TvEpisodeNumberParser
  I want to match words from the filename against episode titles in metadata
  So that episodes identified by title can be resolved without a numeric marker

  Scenario: All title words match, single candidate
    Given the series has an episode S05E14 with title "Ozymandias"
    And the filename is "breaking.bad.ozymandias.mkv"
    When H5 is applied
    Then the result is S05E14

  Scenario: All title words match including multiple words, single candidate
    Given the series has an episode S03E10 with title "Fly"
    And the series has an episode S04E01 with title "Box Cutter"
    And the filename is "breaking.bad.box.cutter.mkv"
    When H5 is applied
    Then the result is S04E01

  Scenario: Full match finds multiple candidates, returns null
    Given the series has an episode S01E01 with title "Pilot"
    And the series has an episode S00E01 with title "Pilot Extended"
    And the filename is "breaking.bad.pilot.mkv"
    When H5 is applied
    Then the result is null

  Scenario: No full match, retry without articles finds single candidate
    Given the series has an episode S02E03 with title "Bit by a Dead Bee"
    And the filename is "breaking.bad.bit.by.a.dead.bee.mkv"
    When H5 is applied
    Then the result is S02E03

  Scenario: No full match, retry without articles finds multiple candidates, returns null
    Given the series has an episode S01E02 with title "Cat's in the Bag"
    And the series has an episode S02E01 with title "Cat's out of the Bag"
    And the filename is "breaking.bad.cats.bag.mkv"
    When H5 is applied
    Then the result is null

  Scenario: No words in filename match any episode title
    Given the filename is "breaking.bad.xyz.mkv"
    When H5 is applied
    Then the result is null

  Scenario: Filename contains no meaningful words after noise removal
    Given the filename is "breaking.bad.1080p.bluray.mkv"
    When H5 is applied
    Then the result is null

  Scenario: No metadata available
    Given no metadata is available for the series
    And the filename is "breaking.bad.ozymandias.mkv"
    When H5 is applied
    Then the result is null
```

---

## Subtasks

- [ ] Implement `TitleMatchHeuristic` in `Scoutarr.Core`
- [ ] Register H5 after H4 in `TvEpisodeNumberParser` DI registration
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TitleMatchHeuristic`
- [ ] Hawkeye reviews
