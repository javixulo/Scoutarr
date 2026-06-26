# UC2-TASK-007 — Output filename formatter (TV episode)

**Requirements:** [requirements/configuration.md](requirements/configuration.md) — "Naming format templates"
**Dependencies:** UC2-TASK-006 (TvEpisodeLookupSuccess)

---

## Context

Given a confirmed TV episode result and a naming template, produce a clean output filename stem (without extension). The template uses `{token}` syntax. The optional `{resolution}` token is omitted — including any immediately preceding separator — when the value is not available.

This task is the TV equivalent of `IMovieFilenameFormatter` from TASK-003. The separator omission rule and template mechanics are identical — only the tokens differ.

This task covers formatting only. No TMDB calls, no filesystem access, no parsing.

---

## Input

```
TvFormatterInput
{
    SeriesName:    string   // from TMDB
    Season:        int      // from ParsedEpisodeNumber
    Episode:       int      // from ParsedEpisodeNumber
    EpisodeTitle:  string   // from TMDB
    Resolution:    string?  // from ParsedTvSeriesTitle — null if not detected
    Template:      string   // e.g. "{series} - S{season:00}E{episode:00} - {title}" — from configuration
}
```

## Output

```
string  // formatted filename stem, no extension, no leading/trailing whitespace
```

On failure: a domain error when the template is null or empty, or when a required token is missing from the input.

---

## Token reference

| Token | Source | Optional |
|---|---|---|
| `{series}` | TMDB series name | No |
| `{season:00}` | Season number, zero-padded to 2 digits | No |
| `{episode:00}` | Episode number, zero-padded to 2 digits | No |
| `{title}` | TMDB episode title | No |
| `{resolution}` | Parsed from filename | Yes — omit token + preceding separator if null |

**Preceding separator omission rule:** when an optional token is absent, the formatter removes the token and any immediately preceding separator characters (space, hyphen, underscore, or any combination thereof).

**Zero-padding rule:** `{season:00}` and `{episode:00}` are always zero-padded to at least 2 digits. Season 1 → `01`, episode 2 → `02`, season 33 → `33`, episode 10 → `10`. Season 0 → `00`.

**Season/episode co-presence rule:** `{season:00}` and `{episode:00}` must both appear in the template. A template containing one but not the other is invalid and returns a domain error.

---

## Notes for Black Widow

- Pure logic — no mocks needed.
- Test the default template with the standard case.
- Test zero-padding: single-digit season and episode, double-digit, season 0.
- Test the separator omission rule for `{resolution}`: space separator, hyphen separator, absent entirely.
- Test non-ASCII episode titles and series names.
- Test that the output has no leading or trailing whitespace.
- Test the co-presence rule: template with only `{season:00}`, template with only `{episode:00}`, both absent — all must return a domain error.
- Test that an unknown token in the template returns a domain error.

## Notes for Tony Stark

- Implement `ITvFilenameFormatter` and `TvFilenameFormatter` in `Scoutarr.Core`.
- The separator omission rule is identical to `MovieFilenameFormatter` — extract it into a shared `TemplateFormatter` utility if not already done in TASK-003.
- Do not hardcode the default template — it comes from configuration (`NAMING_FORMAT_EPISODE`).
- `{season:00}` and `{episode:00}` are format specifiers, not separate tokens — the formatter must parse the `:00` part and apply `ToString("D2")` (or equivalent minimum-width padding).

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV episode output filename formatter
  As the Core identification service
  I want to produce a clean output filename from a confirmed TV episode and a naming template
  So that the renamed file follows the configured naming convention

  # ─────────────────────────────────────────
  # DEFAULT TEMPLATE
  # ─────────────────────────────────────────

  Scenario: Default template — standard case
    Given the template "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    And resolution is null
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02 - Cat's in the Bag"

  Scenario: Default template — double-digit season and episode
    Given the template "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "The Simpsons", season 33, episode 10, episode title "Treehouse of Horror XXXIII"
    When the filename is formatted
    Then the result is "The Simpsons - S33E10 - Treehouse of Horror XXXIII"

  Scenario: Default template — season 0 special
    Given the template "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "Breaking Bad", season 0, episode 1, episode title "Pilot (Unaired)"
    When the filename is formatted
    Then the result is "Breaking Bad - S00E01 - Pilot (Unaired)"

  Scenario: Default template — non-ASCII series name and episode title
    Given the template "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "La Casa de Papel", season 1, episode 1, episode title "Episodio 1"
    When the filename is formatted
    Then the result is "La Casa de Papel - S01E01 - Episodio 1"

  Scenario: Default template — episode title with special characters
    Given the template "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "Breaking Bad", season 1, episode 1, episode title "Pilot: Walter's Story"
    When the filename is formatted
    Then the result is "Breaking Bad - S01E01 - Pilot: Walter's Story"

  # ─────────────────────────────────────────
  # CUSTOM TEMPLATES
  # ─────────────────────────────────────────

  Scenario: Template without series name
    Given the template "S{season:00}E{episode:00} - {title}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    When the filename is formatted
    Then the result is "S01E02 - Cat's in the Bag"

  Scenario: Template without episode title
    Given the template "{series} - S{season:00}E{episode:00}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02"

  Scenario: Template with episode before series
    Given the template "S{season:00}E{episode:00} - {series} - {title}"
    And the series name is "The Simpsons", season 4, episode 12, episode title "Marge vs. the Monorail"
    When the filename is formatted
    Then the result is "S04E12 - The Simpsons - Marge vs. the Monorail"

  Scenario: Minimal template — only season and episode
    Given the template "S{season:00}E{episode:00}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    When the filename is formatted
    Then the result is "S01E02"

  Scenario: Compact template with dots as separators
    Given the template "{series}.S{season:00}E{episode:00}.{title}"
    And the series name is "Breaking Bad", season 2, episode 3, episode title "Bit by a Dead Bee"
    When the filename is formatted
    Then the result is "Breaking Bad.S02E03.Bit by a Dead Bee"

  # ─────────────────────────────────────────
  # OPTIONAL TOKEN — resolution
  # ─────────────────────────────────────────

  Scenario: Template with resolution — resolution present
    Given the template "{series} - S{season:00}E{episode:00} - {title} {resolution}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    And resolution is "1080p"
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02 - Cat's in the Bag 1080p"

  Scenario: Template with resolution — resolution absent, space separator omitted
    Given the template "{series} - S{season:00}E{episode:00} - {title} {resolution}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    And resolution is null
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02 - Cat's in the Bag"

  Scenario: Template with resolution — resolution absent, hyphen separator omitted
    Given the template "{series} - S{season:00}E{episode:00} - {title} - {resolution}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    And resolution is null
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02 - Cat's in the Bag"

  # ─────────────────────────────────────────
  # NEGATIVE CASES
  # ─────────────────────────────────────────

  Scenario: Template is empty — falls back to configuration default
    Given the template ""
    And the configuration default episode template is "{series} - S{season:00}E{episode:00} - {title}"
    And the series name is "Breaking Bad", season 1, episode 2, episode title "Cat's in the Bag"
    When the filename is formatted
    Then the result is "Breaking Bad - S01E02 - Cat's in the Bag"

  Scenario: Template contains unknown token — error returned
    Given the template "{series} - S{season:00}E{episode:00} - {title} {unknown}"
    When the filename is formatted
    Then a format error is returned with reason "Unknown token: {unknown}"

  Scenario: Template contains season but not episode — error returned
    Given the template "{series} - S{season:00} - {title}"
    When the filename is formatted
    Then a format error is returned with reason "Template must include both {season:00} and {episode:00}"

  Scenario: Template contains episode but not season — error returned
    Given the template "{series} - E{episode:00} - {title}"
    When the filename is formatted
    Then a format error is returned with reason "Template must include both {season:00} and {episode:00}"

  Scenario: Template contains neither season nor episode — error returned
    Given the template "{series} - {title}"
    When the filename is formatted
    Then a format error is returned with reason "Template must include both {season:00} and {episode:00}"
```

---

## Subtasks

- [ ] Define `TvFormatterInput` record in `Scoutarr.Core`
- [ ] Define `ITvFilenameFormatter` interface in `Scoutarr.Core`
- [ ] Evaluate whether the separator omission and template parsing logic can be shared with `MovieFilenameFormatter` via a `TemplateFormatter` utility
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvFilenameFormatter`
- [ ] Hawkeye reviews
