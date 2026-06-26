# UC2-TASK-001 — TV series title parser

**Requirements:** [identification.md](identification.md) — "Information extracted from the filename"

---

## Context

Given a dirty TV episode filename, extract the series title and optionally the year and resolution. This is the first step in the UC-2 pipeline — the result feeds directly into `TvShowSearchService` to identify the series against TMDB.

This task covers:

- Extracting a clean series title from a dirty filename (dots, underscores, release group tags, episode patterns, resolution tokens, etc.)
- Extracting year when present
- Extracting resolution when present
- Stripping everything from the episode pattern onwards (season/episode markers, quality tags, release group)

No episode number extraction. No TMDB calls. No confidence scoring. No output formatting. Pure logic.

---

## Output types

```
ParsedTvSeriesTitle
{
    SeriesTitle:   string   // cleaned, separators replaced, release group stripped
    Year:          int?     // null if not found in filename
    Resolution:    string?  // null if not found in filename
}
```

On failure: a domain error when no recognisable series title can be extracted (e.g. empty filename, filename is only noise tokens).

---

## Title cleaning rules

1. Detect the episode pattern boundary — the first occurrence of an SxEy marker (`S01E02`, `s1e2`, etc.) or a compact numeric token that looks like an episode (`102`, `1102` between separators). Everything from that boundary onwards is discarded before cleaning the title.
2. Replace dots and underscores with spaces.
3. Collapse multiple consecutive spaces into one.
4. Trim leading and trailing whitespace.
5. Strip trailing release group tag: an uppercase word preceded by a hyphen at the very end (e.g. `-YIFY`, `-GROUP`).
6. Year token (4-digit sequence between 1900 and current year) is extracted and removed from the title.
7. Resolution tokens (`720p`, `1080p`, `2160p`, `4K`, case-insensitive) are extracted and removed from the title.

---

## Notes for Black Widow

- All tests are pure — no mocks, no external dependencies.
- Test the full range of SxEy separators: uppercase, lowercase, zero-padded, non-padded, dots, underscores, spaces.
- Test year extraction: present, absent, year-like number that is not a year (e.g. `1899`).
- Test resolution extraction: `720p`, `1080p`, `2160p`, `4K`, absent.
- Test release group stripping: present, absent, hyphen in series title that is not a release group (e.g. `Man-Bat`).
- Test that everything after the episode boundary is discarded before cleaning the title.
- Test that a filename with no recognisable series title returns a domain error.

## Notes for Tony Stark

- Implement `ITvSeriesTitleParser` and `TvSeriesTitleParser` in `Scoutarr.Core`.
- No constructor dependencies — this is a pure parser.
- The episode boundary detection in this parser is only used to know where to truncate; the actual season/episode numbers are not extracted here.
- Reuse title cleaning logic with `MovieFilenameParser` if possible — extract a shared `TitleCleaner` utility if the logic is identical or nearly identical.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV series title parser
  As the Core identification service
  I want to extract a clean series title from a dirty TV episode filename
  So that the result can be used to search TMDB for the correct series

  # ─────────────────────────────────────────
  # TITLE EXTRACTION — basic cases
  # ─────────────────────────────────────────

  Scenario: Dots as separators, SxEy boundary
    Given the filename "Breaking.Bad.S01E02.720p.mkv"
    When the filename is parsed
    Then the series title is "Breaking Bad"
    And the resolution is "720p"

  Scenario: Underscores as separators
    Given the filename "Breaking_Bad_S02E05_1080p.mkv"
    When the filename is parsed
    Then the series title is "Breaking Bad"
    And the resolution is "1080p"

  Scenario: Lowercase SxEy boundary
    Given the filename "breaking.bad.s01e02.mkv"
    When the filename is parsed
    Then the series title is "Breaking Bad"

  Scenario: Non-padded SxEy boundary
    Given the filename "Breaking.Bad.S1E2.mkv"
    When the filename is parsed
    Then the series title is "Breaking Bad"

  Scenario: Multi-word title with spaces
    Given the filename "House of the Dragon S01E03 4K.mkv"
    When the filename is parsed
    Then the series title is "House of the Dragon"
    And the resolution is "4K"

  Scenario: High season number
    Given the filename "The.Simpsons.S33E10.mkv"
    When the filename is parsed
    Then the series title is "The Simpsons"

  # ─────────────────────────────────────────
  # YEAR EXTRACTION
  # ─────────────────────────────────────────

  Scenario: Year present before episode boundary
    Given the filename "The.Office.2005.S03E01.mkv"
    When the filename is parsed
    Then the series title is "The Office"
    And the year is 2005

  Scenario: Year absent
    Given the filename "Breaking.Bad.S01E02.mkv"
    When the filename is parsed
    Then the year is null

  Scenario: Year-like number below 1900 is not extracted as year
    Given the filename "Show.1899.S01E01.mkv"
    When the filename is parsed
    Then the year is null
    And the series title is "Show 1899"

  # ─────────────────────────────────────────
  # RESOLUTION EXTRACTION
  # ─────────────────────────────────────────

  Scenario: 720p resolution
    Given the filename "Breaking.Bad.S01E02.720p.mkv"
    When the filename is parsed
    Then the resolution is "720p"

  Scenario: 2160p resolution
    Given the filename "Breaking.Bad.S01E02.2160p.mkv"
    When the filename is parsed
    Then the resolution is "2160p"

  Scenario: 4K resolution token
    Given the filename "House.of.the.Dragon.S01E03.4K.mkv"
    When the filename is parsed
    Then the resolution is "4K"

  Scenario: No resolution token
    Given the filename "Breaking.Bad.S01E02.mkv"
    When the filename is parsed
    Then the resolution is null

  # ─────────────────────────────────────────
  # RELEASE GROUP STRIPPING
  # ─────────────────────────────────────────

  Scenario: Release group tag after resolution is stripped
    Given the filename "Game.of.Thrones.S08E06.1080p-YIFY.mkv"
    When the filename is parsed
    Then the series title is "Game of Thrones"
    And the resolution is "1080p"

  Scenario: Hyphen in series title is not mistaken for release group
    Given the filename "Man-Bat.S01E01.mkv"
    When the filename is parsed
    Then the series title is "Man-Bat"

  # ─────────────────────────────────────────
  # NEGATIVE CASES
  # ─────────────────────────────────────────

  Scenario: Filename is only noise tokens, no recognisable title
    Given the filename "S01E02.720p-YIFY.mkv"
    When the filename is parsed
    Then a domain error is returned with reason "No series title found"
```

---

## Subtasks

- [ ] Define `ParsedTvSeriesTitle` record in `Scoutarr.Core`
- [ ] Define `ITvSeriesTitleParser` interface in `Scoutarr.Core`
- [ ] Evaluate whether title cleaning logic can be shared with `MovieFilenameParser` via a `TitleCleaner` utility
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvSeriesTitleParser`
- [ ] Hawkeye reviews
