# UC5-TASK-003 — H4: Episode identification by air date

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-005 (IEpisodeHeuristic, EpisodeParseContext, TvEpisodeNumberParser), UC3-TASK-002 (SeriesMetadata, AirDate in EpisodeMetadata)

---

## Context

Some filenames encode the episode's broadcast date rather than a season/episode number. This is common for talk shows, late night programmes, daily news shows, and sports broadcasts — content where episodes are identified by when they aired rather than their position in a season.

Examples of filenames this heuristic targets:

- `breaking.bad.2013.09.29.mkv` — date in YYYY.MM.DD format
- `breaking.bad.20130929.mkv` — date in YYYYMMDD format
- `breaking.bad.2013-09-29.mkv` — date in YYYY-MM-DD format

H4 extracts a date from the filename using an ordered list of date patterns and matches it against `AirDate` in `SeriesMetadata`. If no metadata is available, or no date is found in the filename, H4 returns `null` and the chain continues.

This task implements `IEpisodeHeuristic` and registers it as H4 in the heuristic chain in `TvEpisodeNumberParser`.

---

## Resolution logic

### Step 1 — Guard checks

- If `context.Metadata` is null → return `null`.
- If `context.Filename` contains no recognisable date → return `null`.

### Step 2 — Extract date from filename

Date patterns are tried in order. The first match wins. Patterns are defined as a static ordered list to allow future extensions without structural changes:

| Priority | Format | Example |
|---|---|---|
| 1 | `YYYY.MM.DD` | `2013.09.29` |
| 2 | `YYYY-MM-DD` | `2013-09-29` |
| 3 | `YYYYMMDD` | `20130929` |

The extracted value is parsed into a `DateOnly`. Invalid dates (e.g. month 13, day 32) are rejected and the next pattern is tried.

### Step 3 — Match against metadata

Search all episodes across all seasons in `SeriesMetadata` for episodes where `AirDate == extractedDate`. Collect all matches.

### Step 4 — Resolve

- If exactly one episode matches → proceed to Step 5 (title confirmation).
- If multiple episodes match → attempt title disambiguation (Step 5) across all candidates.
- If no episodes match → return `null`.

### Step 5 — Title confirmation / disambiguation

Extract candidate words from the filename by removing:
- The series title tokens
- The date token
- Common noise tokens (resolution tags, release group suffixes, file extension)

Match the remaining words against the episode title of each candidate using case-insensitive token matching. A match is counted if at least one meaningful word (> 3 characters) from the filename appears in the episode title.

- Single date match + title confirms → return that episode.
- Single date match + title does not match → return that episode anyway (date alone is sufficient).
- Multiple date matches + exactly one title matches → return that episode.
- Multiple date matches + title matches none or more than one → return `null`.

---

## Extensibility

Date patterns are defined as a static `IReadOnlyList<DatePattern>` inside `AirDateHeuristic`. Adding a new format requires only adding a new entry to that list — no interface changes, no configuration needed.

---

## Interface

H4 implements `IEpisodeHeuristic` from UC2-TASK-005:

```
IEpisodeHeuristic

TryParse(EpisodeParseContext context)
    → ParsedEpisodeNumber?
```

`TvEpisodeNumberParser` injects H4 after H3 in its ordered heuristic list.

---

## Notes for Black Widow

- All tests are pure — populate `EpisodeParseContext` directly with mock `SeriesMetadata`. No filesystem or TMDB calls.
- Test null metadata → `null`.
- Test filename with no recognisable date → `null`.
- Test each supported date format (`YYYY.MM.DD`, `YYYY-MM-DD`, `YYYYMMDD`) resolves correctly.
- Test invalid date in filename (e.g. `2013.13.01`) → skipped, treated as no date found.
- Test date found in metadata, single match, no title words → returns that episode.
- Test date found in metadata, single match, title confirms → returns that episode.
- Test date found in metadata, multiple matches, title disambiguates → returns matching episode.
- Test date found in metadata, multiple matches, title does not disambiguate → `null`.
- Test date not found in metadata → `null`.
- Test H4 is registered after H3 in `TvEpisodeNumberParser` — verify ordering in unit tests.

## Notes for Tony Stark

- Implement `AirDateHeuristic` in `Scoutarr.Core`.
- `TryParse` must be synchronous — no async, no I/O.
- Define date patterns as a `static readonly IReadOnlyList<DatePattern>` field. Each `DatePattern` encapsulates a regex and a parse delegate.
- `YYYYMMDD` pattern must include a sanity check to avoid matching bare numbers that happen to be 8 digits but are not valid dates (e.g. `20991399`).
- Title token matching: match if at least one meaningful word (> 3 characters) from the filename appears in the episode title, case-insensitive.
- Register H4 immediately after H3 in the DI registration of `TvEpisodeNumberParser`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: H4 — Episode identification by air date
  As the TvEpisodeNumberParser
  I want to extract a date from a filename and match it against episode air dates
  So that episodes identified by broadcast date are correctly resolved

  Scenario: Date in YYYY.MM.DD format matches exactly one episode
    Given the series has an episode S03E05 with AirDate 2013-09-29
    And the filename is "breaking.bad.2013.09.29.mkv"
    When H4 is applied
    Then the result is S03E05

  Scenario: Date in YYYYMMDD format matches exactly one episode
    Given the series has an episode S03E05 with AirDate 2013-09-29
    And the filename is "breaking.bad.20130929.mkv"
    When H4 is applied
    Then the result is S03E05

  Scenario: Date in YYYY-MM-DD format matches exactly one episode
    Given the series has an episode S03E05 with AirDate 2013-09-29
    And the filename is "breaking.bad.2013-09-29.mkv"
    When H4 is applied
    Then the result is S03E05

  Scenario: Date matches multiple episodes, title disambiguates
    Given the series has S02E01 with AirDate 2013-09-29 and title "Blood Money"
    And the series has S02E02 with AirDate 2013-09-29 and title "Buried"
    And the filename is "breaking.bad.2013.09.29.blood.money.mkv"
    When H4 is applied
    Then the result is S02E01

  Scenario: Date matches multiple episodes, title does not disambiguate
    Given the series has S02E01 with AirDate 2013-09-29 and title "Blood Money"
    And the series has S02E02 with AirDate 2013-09-29 and title "Buried"
    And the filename is "breaking.bad.2013.09.29.mkv"
    When H4 is applied
    Then the result is null

  Scenario: Date not found in metadata
    Given the series has no episode with AirDate 2013-09-29
    And the filename is "breaking.bad.2013.09.29.mkv"
    When H4 is applied
    Then the result is null

  Scenario: Filename contains no recognisable date
    Given the filename is "breaking.bad.mkv"
    When H4 is applied
    Then the result is null

  Scenario: No metadata available
    Given no metadata is available for the series
    And the filename is "breaking.bad.2013.09.29.mkv"
    When H4 is applied
    Then the result is null
```

---

## Subtasks

- [ ] Implement `AirDateHeuristic` in `Scoutarr.Core`
- [ ] Register H4 after H3 in `TvEpisodeNumberParser` DI registration
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `AirDateHeuristic`
- [ ] Hawkeye reviews
