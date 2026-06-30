# UC1-TASK-001 — Movie filename parser

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Information extracted from the filename"
**File types:** [requirements/file-handling.md](../requirements/file-handling.md) — "Recognised file extensions"

---

## Context

Before querying TMDB, Scoutarr must parse a dirty movie filename and extract four pieces of structured data:

- **Title** — required; used as the primary TMDB search query.
- **Year** — optional; used to narrow the TMDB search when present.
- **Resolution** — optional; stored for use in the output filename template.
- **Edition** — optional; e.g. `Director's Cut`, `Extended`, `Unrated`, `Theatrical`, `Remastered`. TMDB does not expose edition/alternate-version data for movies, so this is detected purely from a fixed list of filename keywords (no TMDB call involved).

The release group tag (e.g. `-YIFY`, `[YIFY]`) must be identified and discarded.

This task covers parsing only. No TMDB calls, no filesystem access, no output formatting.

---

## Output type

```csharp
ParsedMovieFilename
{
    Title:        string        // never null or empty on success
    Year:         int?          // null if not found or not parseable
    Resolution:   string?       // null if not found; normalised (e.g. "4K" not "2160p")
    Edition:      string?       // null if no recognised edition keyword is found; normalised display form (e.g. "Director's Cut")
    ReleaseGroup: string?       // null if not found
}
```

On failure: a domain error describing why parsing failed (empty filename, no extension, unrecognised extension, no extractable title).

---

## Notes for Black Widow

- Test all Gherkin scenarios below as individual xUnit `[Fact]` or `[Theory]` tests.
- The parser is pure logic — no mocks needed.
- Test the output type fields directly; do not test internal regex or implementation details.
- Recognised video extensions are defined in `file-handling.md`; for tests, use `.mkv`, `.mp4`, and `.avi` to cover the main cases.

## Notes for Tony Stark

- Implement `IMovieFilenameParser` in `Scoutarr.Core`.
- Normalise `2160p` → `"4K"` in the output.
- Year must be a plausible 4-digit number but no range validation — if TMDB can't find it, that's TMDB's problem, not the parser's.
- Release group detection: strip the `-GROUP` suffix and `[GROUP]` bracket patterns at the end of the stem, after all other tokens are consumed.
- Edition detection: match against a fixed keyword table (case-insensitive, separators `.`/`_`/` ` normalised to spaces), e.g. `DIRECTORS CUT` / `DIRECTOR'S CUT` → `"Director's Cut"`, `EXTENDED` / `EXTENDED EDITION` / `EXTENDED CUT` → `"Extended"`, `UNRATED` → `"Unrated"`, `THEATRICAL` / `THEATRICAL CUT` → `"Theatrical"`, `REMASTERED` → `"Remastered"`, `ULTIMATE EDITION` → `"Ultimate Edition"`, `SPECIAL EDITION` → `"Special Edition"`. Keep the table easy to extend — new edition keywords will surface over time, the same way release-group conventions do.
- Edition tokens must be consumed like any other noise token (similar to `PROPER`/`REPACK`) and must never leak into the extracted title.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Movie filename parser
  As the Core identification service
  I want to extract structured metadata from a dirty movie filename
  So that I can query TMDB with clean, reliable data

  # ─────────────────────────────────────────
  # HAPPY PATH — standard patterns
  # ─────────────────────────────────────────

  Scenario: Dot-separated filename with title, year and resolution
    Given the filename "The.Batman.2022.1080p.BluRay.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"
    And the release group is null

  Scenario: Space-separated filename
    Given the filename "The Batman 2022 1080p BluRay.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"

  Scenario: Underscore-separated filename
    Given the filename "The_Batman_2022_1080p.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"

  Scenario: Mixed separators
    Given the filename "The.Batman.2022_1080p BluRay.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"

  Scenario: 4K resolution token (2160p normalised to 4K)
    Given the filename "Dune.2021.2160p.BluRay.mkv"
    When the filename is parsed
    Then the title is "Dune"
    And the year is 2021
    And the resolution is "4K"

  Scenario: 720p resolution
    Given the filename "Inception.2010.720p.mkv"
    When the filename is parsed
    Then the title is "Inception"
    And the year is 2010
    And the resolution is "720p"

  Scenario: Release group tag with hyphen stripped
    Given the filename "The.Batman.2022.1080p.BluRay-YIFY.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the release group is "YIFY"

  Scenario: Release group tag in brackets stripped
    Given the filename "The.Batman.2022.1080p.[YIFY].mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the release group is "YIFY"

  # ─────────────────────────────────────────
  # OPTIONAL FIELDS — year or resolution absent
  # ─────────────────────────────────────────

  Scenario: No year in filename
    Given the filename "Inception.BluRay.1080p.mkv"
    When the filename is parsed
    Then the title is "Inception"
    And the year is null
    And the resolution is "1080p"

  Scenario: No resolution in filename
    Given the filename "The.Batman.2022.BluRay.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is null

  Scenario: No year and no resolution
    Given the filename "Inception.mkv"
    When the filename is parsed
    Then the title is "Inception"
    And the year is null
    And the resolution is null

  # ─────────────────────────────────────────
  # EDITION DETECTION
  # ─────────────────────────────────────────

  Scenario: Director's Cut edition detected
    Given the filename "Blade.Runner.1982.Directors.Cut.1080p.mkv"
    When the filename is parsed
    Then the title is "Blade Runner"
    And the year is 1982
    And the edition is "Director's Cut"

  Scenario: Extended edition detected
    Given the filename "The.Lord.of.the.Rings.2001.EXTENDED.1080p.BluRay.mkv"
    When the filename is parsed
    Then the title is "The Lord of the Rings"
    And the year is 2001
    And the edition is "Extended"

  Scenario: Unrated edition detected
    Given the filename "Superbad.2007.UNRATED.720p.mkv"
    When the filename is parsed
    Then the title is "Superbad"
    And the edition is "Unrated"

  Scenario: No edition keyword present
    Given the filename "The.Batman.2022.1080p.BluRay.mkv"
    When the filename is parsed
    Then the edition is null

  Scenario: Edition keyword does not leak into the title
    Given the filename "Close.Encounters.of.the.Third.Kind.1977.Directors.Cut.1080p.mkv"
    When the filename is parsed
    Then the title is "Close Encounters of the Third Kind"
    And the edition is "Director's Cut"

  # ─────────────────────────────────────────
  # UNUSUAL TOKEN ORDER
  # ─────────────────────────────────────────

  Scenario: Resolution appears before year
    Given the filename "The.Batman.1080p.2022.BluRay.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"

  Scenario: Year appears at the beginning
    Given the filename "2022.The.Batman.1080p.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"

  Scenario: Extra noise tokens between title and year
    Given the filename "The.Batman.EXTENDED.2022.1080p.mkv"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"
    And the edition is "Extended"

  Scenario: Multiple noise tokens scattered through filename
    Given the filename "The.Batman.2022.PROPER.REPACK.1080p.BluRay.x264-GROUP.mp4"
    When the filename is parsed
    Then the title is "The Batman"
    And the year is 2022
    And the resolution is "1080p"
    And the release group is "GROUP"

  # ─────────────────────────────────────────
  # TRICKY TITLES
  # ─────────────────────────────────────────

  Scenario: Title contains a number that is not the year
    Given the filename "2001.A.Space.Odyssey.1968.1080p.mkv"
    When the filename is parsed
    Then the title is "2001 A Space Odyssey"
    And the year is 1968
    And the resolution is "1080p"

  Scenario: Title is a 4-digit number followed by a release year
    Given the filename "1917.2019.1080p.mkv"
    When the filename is parsed
    Then the title is "1917"
    And the year is 2019
    And the resolution is "1080p"

  Scenario: Title is a single word
    Given the filename "Dune.2021.1080p.mkv"
    When the filename is parsed
    Then the title is "Dune"
    And the year is 2021

  Scenario: Title contains an apostrophe
    Given the filename "Schindler's.List.1993.1080p.mkv"
    When the filename is parsed
    Then the title is "Schindler's List"
    And the year is 1993

  Scenario: Title in non-ASCII characters
    Given the filename "Amélie.2001.1080p.mkv"
    When the filename is parsed
    Then the title is "Amélie"
    And the year is 2001

  # ─────────────────────────────────────────
  # NEGATIVE CASES
  # ─────────────────────────────────────────

  Scenario: Filename is empty
    Given the filename ""
    When the filename is parsed
    Then a parse error is returned with reason "Filename is empty"

  Scenario: Filename has no extension
    Given the filename "The.Batman.2022.1080p"
    When the filename is parsed
    Then a parse error is returned with reason "No file extension found"

  Scenario: Filename extension is not a recognised video format
    Given the filename "The.Batman.2022.1080p.pdf"
    When the filename is parsed
    Then a parse error is returned with reason "Unrecognised file extension"

  Scenario: No parseable title can be extracted
    Given the filename "1080p.BluRay.mkv"
    When the filename is parsed
    Then a parse error is returned with reason "Could not extract title"
```

---

## Subtasks

- [ ] Define `ParsedMovieFilename` record in `Scoutarr.Core` (including `Edition`)
- [ ] Define `IMovieFilenameParser` interface in `Scoutarr.Core`
- [ ] Define the edition keyword table (Director's Cut, Extended, Unrated, Theatrical, Remastered, Ultimate Edition, Special Edition — extensible)
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `MovieFilenameParser`
- [ ] Hawkeye reviews
