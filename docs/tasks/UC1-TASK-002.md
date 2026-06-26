# UC1-TASK-002 — TMDB movie search and confidence scoring

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Metadata source", "Confidence and automatic acceptance", "No results"
**Skill:** [skills/tmdb-api/SKILL.md](skills/tmdb-api/SKILL.md)

---

## Context

Given a `ParsedMovieFilename` from UC1-TASK-001, Scoutarr queries TMDB for movie candidates and assigns a confidence score to each result. The caller receives a ranked list of `MovieCandidate` objects.

This task covers:
- Calling the TMDB `/search/movie` endpoint via `ITmdbClient`
- Computing a confidence score per candidate
- Returning a ranked list

No filesystem access. No disambiguation logic. No output formatting.

---

## Output types

```csharp
MovieCandidate
{
    TmdbId:            int
    Title:             string
    Year:              int?
    Overview:          string?
    OriginalLanguage:  string
    Confidence:        double   // 0.0–1.0
}

MovieSearchResult
{
    Candidates: IReadOnlyList<MovieCandidate>   // ordered by confidence descending
}
```

On failure: a domain error (TMDB unreachable, zero results from TMDB).

---

## Confidence score formula

TMDB does not return a confidence score. Scoutarr computes it as a weighted combination:

| Signal | Weight |
|---|---|
| Title similarity (normalised Levenshtein or equivalent) | 60% |
| Year match (exact match = full points, absent = neutral, mismatch = penalty) | 30% |
| TMDB popularity (normalised, used as tiebreaker) | 10% |

**Year scoring detail:**
- Parsed year matches TMDB release year → full 30 points
- Parsed year is null (not found in filename) → 15 points (neutral, no bonus no penalty)
- Parsed year does not match TMDB release year → 0 points

**Title similarity:**
- Compare the parsed title (lowercased, punctuation stripped) against the TMDB `title` and `original_title` (same normalisation). Take the best of the two.

The final score is clamped to [0.0, 1.0].

---

## Notes for Black Widow

- `ITmdbClient` must be mocked in all tests — never call the real API.
- Mock realistic TMDB response structures as defined in `skills/tmdb-api/SKILL.md`.
- Test confidence scoring by constructing scenarios where the expected winner is unambiguous (e.g. exact title + year match scores higher than partial title match).
- Test ranking: candidates must be ordered by confidence descending.
- Test error cases: TMDB unreachable, zero results.
- Do not test the internal scoring formula directly — test observable outcomes (which candidate wins, what score range it falls in).

## Notes for Tony Stark

- Define `ITmdbClient` interface in `Scoutarr.Core` with a `SearchMovieAsync` method.
- Implement `TmdbClient` as the real HTTP client — unit tests will mock the interface, E2E will use the real implementation.
- Implement `IMovieSearchService` in `Scoutarr.Core` — this is what takes a `ParsedMovieFilename` and returns `MovieSearchResult`.
- Title normalisation: lowercase, strip punctuation, collapse whitespace.
- Popularity normalisation: use `Math.Log` or a similar dampening function — raw popularity scores from TMDB vary wildly (0.6 to 5000+) and must not dominate the score.
- **Implement the scoring logic in a standalone `ConfidenceScorer` class in `Scoutarr.Core`, not inline in `MovieSearchService`.** `MovieSearchService` calls `ConfidenceScorer` — it does not own the formula. This allows UC-02's `TvShowSearchService` to reuse `ConfidenceScorer` directly without duplication.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TMDB movie search and confidence scoring
  As the Core identification service
  I want to search TMDB for movie candidates and rank them by confidence
  So that the caller can select the best match or trigger disambiguation

  Scenario: Exact title and year match scores highest
    Given a parsed filename with title "The Batman" and year 2022
    And TMDB returns candidates including "The Batman" (2022) and "Batman" (1989)
    When the search is executed
    Then the top candidate is "The Batman" (2022)
    And its confidence score is above 0.90

  Scenario: Exact title match without year scores reasonably high
    Given a parsed filename with title "Inception" and no year
    And TMDB returns a single candidate "Inception" (2010)
    When the search is executed
    Then the top candidate is "Inception" (2010)
    And its confidence score is above 0.60

  Scenario: TMDB returns zero results
    Given a parsed filename with title "Zxqfnmvp" and no year
    And TMDB returns zero candidates
    When the search is executed
    Then a domain error is returned with reason "No TMDB results found"

  Scenario: TMDB is unreachable
    Given a parsed filename with title "The Batman" and year 2022
    And the TMDB client throws a network exception
    When the search is executed
    Then a domain error is returned with reason "TMDB API unreachable"
```

---

## Subtasks

- [ ] Define `MovieCandidate` and `MovieSearchResult` records in `Scoutarr.Core`
- [ ] Define `ITmdbClient` interface in `Scoutarr.Core`
- [ ] Define `IMovieSearchService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TmdbClient` (real HTTP) and `MovieSearchService` (scoring logic)
- [ ] Hawkeye reviews
