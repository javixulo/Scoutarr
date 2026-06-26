# TASK-003 — Output filename formatter (movie)

**Requirements:** [requirements/configuration.md](../requirements/configuration.md) — "Naming format templates"

---

## Context

Given a TMDB movie result and a naming template, produce a clean output filename (without extension). The template uses `{token}` syntax. Optional tokens (`{resolution}`, `{edition}`) are omitted entirely — including any immediately preceding separator — when the value is not available.

This task covers formatting only. No TMDB calls, no filesystem access, no parsing.

---

## Input

```csharp
MovieFormatterInput
{
    Title:       string   // from TMDB
    Year:        int      // from TMDB
    Resolution:  string?  // from ParsedMovieFilename — null if not detected
    Edition:     string?  // null if not detected
    Template:    string   // e.g. "{title} ({year})" — from configuration
}
```

## Output

```csharp
string  // formatted filename stem, no extension, no leading/trailing whitespace
```

On failure: a domain error when the template is null or empty, or when a required token (`{title}`, `{year}`) is missing from the input.

---

## Token reference

| Token | Source | Optional |
|---|---|---|
| `{title}` | TMDB title | No |
| `{year}` | TMDB release year | No |
| `{resolution}` | Parsed from filename | Yes — omit token + preceding separator if null |
| `{edition}` | Parsed from filename | Yes — omit token + preceding separator if null |

**Preceding separator omission rule:** when an optional token is absent, the formatter removes the token and any immediately preceding separator characters (space, hyphen, underscore, or any combination thereof).

---

## Notes for Black Widow

- Pure logic — no mocks needed.
- Test the omission rule carefully: separator before optional token must be stripped, but separators elsewhere in the template must be preserved.
- Test with the default template and with custom templates.
- Test that the output has no leading or trailing whitespace.

## Notes for Tony Stark

- Implement `IMovieFilenameFormatter` in `Scoutarr.Core`.
- The separator omission rule applies to any characters immediately before the token that are not part of another token or its value — strip greedily leftward until hitting a non-separator character or another token boundary.
- Do not hardcode the default template — it comes from configuration.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Movie output filename formatter
  As the Core identification service
  I want to produce a clean output filename from a TMDB result and a naming template
  So that the renamed file follows the configured naming convention

  # ─────────────────────────────────────────
  # DEFAULT TEMPLATE
  # ─────────────────────────────────────────

  Scenario: Default template with title and year
    Given the template "{title} ({year})"
    And the movie title is "The Batman" and year is 2022
    And resolution is null and edition is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Default template — title with special characters
    Given the template "{title} ({year})"
    And the movie title is "Schindler's List" and year is 1993
    When the filename is formatted
    Then the result is "Schindler's List (1993)"

  Scenario: Default template — non-ASCII title
    Given the template "{title} ({year})"
    And the movie title is "Amélie" and year is 2001
    When the filename is formatted
    Then the result is "Amélie (2001)"

  # ─────────────────────────────────────────
  # OPTIONAL TOKENS PRESENT
  # ─────────────────────────────────────────

  Scenario: Template with resolution — resolution present
    Given the template "{title} ({year}) {resolution}"
    And the movie title is "The Batman" and year is 2022
    And resolution is "1080p"
    When the filename is formatted
    Then the result is "The Batman (2022) 1080p"

  Scenario: Template with edition — edition present
    Given the template "{title} ({year}) {edition}"
    And the movie title is "Blade Runner" and year is 1982
    And edition is "Director's Cut"
    When the filename is formatted
    Then the result is "Blade Runner (1982) Director's Cut"

  Scenario: Template with both optional tokens — both present
    Given the template "{title} ({year}) {edition} {resolution}"
    And the movie title is "Blade Runner" and year is 1982
    And edition is "Director's Cut" and resolution is "4K"
    When the filename is formatted
    Then the result is "Blade Runner (1982) Director's Cut 4K"

  # ─────────────────────────────────────────
  # OPTIONAL TOKENS ABSENT — separator omission
  # ─────────────────────────────────────────

  Scenario: Template with resolution — resolution absent, space separator omitted
    Given the template "{title} ({year}) {resolution}"
    And the movie title is "The Batman" and year is 2022
    And resolution is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template with resolution — resolution absent, hyphen separator omitted
    Given the template "{title} ({year}) - {resolution}"
    And the movie title is "The Batman" and year is 2022
    And resolution is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template with resolution — resolution absent, underscore separator omitted
    Given the template "{title} ({year})_{resolution}"
    And the movie title is "The Batman" and year is 2022
    And resolution is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template with edition — edition absent, separator omitted
    Given the template "{title} ({year}) {edition}"
    And the movie title is "The Batman" and year is 2022
    And edition is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template with both optional tokens — both absent
    Given the template "{title} ({year}) {edition} {resolution}"
    And the movie title is "The Batman" and year is 2022
    And edition is null and resolution is null
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template with both optional tokens — only resolution absent
    Given the template "{title} ({year}) {edition} {resolution}"
    And the movie title is "Blade Runner" and year is 1982
    And edition is "Director's Cut" and resolution is null
    When the filename is formatted
    Then the result is "Blade Runner (1982) Director's Cut"

  Scenario: Template with both optional tokens — only edition absent
    Given the template "{title} ({year}) {edition} {resolution}"
    And the movie title is "The Batman" and year is 2022
    And edition is null and resolution is "1080p"
    When the filename is formatted
    Then the result is "The Batman (2022) 1080p"

  # ─────────────────────────────────────────
  # NEGATIVE CASES
  # ─────────────────────────────────────────

  Scenario: Template is empty — falls back to configuration default
    Given the template ""
    And the configuration default movie template is "{title} ({year})"
    And the movie title is "The Batman" and year is 2022
    When the filename is formatted
    Then the result is "The Batman (2022)"

  Scenario: Template contains unknown token
    Given the template "{title} ({year}) {unknown}"
    And the movie title is "The Batman" and year is 2022
    When the filename is formatted
    Then a format error is returned with reason "Unknown token: {unknown}"
```

---

## Subtasks

- [ ] Define `MovieFormatterInput` record in `Scoutarr.Core`
- [ ] Define `IMovieFilenameFormatter` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `MovieFilenameFormatter`
- [ ] Hawkeye reviews
