# UC2-TASK-003 — TV show search service

**Requirements:** [identification.md](identification.md) — "Candidate scoring", "Confidence threshold"
**Dependencies:** UC2-TASK-001 (ParsedTvSeriesTitle), UC2-TASK-002 (ITmdbClient TV methods), TASK-002 (ConfidenceScorer)

---

## Context

Given a parsed series title (and optional year), search TMDB for TV show candidates and score them using `ConfidenceScorer`. Return an ordered list of candidates with their confidence scores.

This is the direct TV equivalent of `MovieSearchService` from TASK-002. The scoring formula is identical — `ConfidenceScorer` is reused without modification.

No orchestration. No disambiguation logic. No file renaming. Pure search and scoring.

---

## Output types

```
TvShowCandidate
{
    TmdbId:      int
    Name:        string
    Year:        int?
    Overview:    string?
    Confidence:  double    // 0.0 – 1.0
}

TvShowSearchResult
{
    Candidates:  IReadOnlyList<TvShowCandidate>   // ordered by confidence descending
}
```

---

## Interface

```
ITvShowSearchService

SearchAsync(ParsedTvSeriesTitle parsed, CancellationToken ct)
    → TvShowSearchResult
```

---

## Scoring

Uses `ConfidenceScorer` (extracted in TASK-002) with the same formula applied to movies:
- 60% title similarity
- 30% year match (full score if year matches, zero if no year available in either source)
- 10% popularity

The result list is ordered by confidence descending. Candidates below `CONFIDENCE_THRESHOLD` are included in the list — filtering is the responsibility of the identification service (UC2-TASK-004).

---

## Notes for Black Widow

- Mock `ITmdbClient` for all tests — never call the real API.
- Test that candidates are ordered by confidence descending.
- Test the year match component: year present and matching, year present and not matching, year absent in parsed input, year absent in TMDB result.
- Test that candidates below `CONFIDENCE_THRESHOLD` are still returned — this service does not filter.
- Test that TMDB returning zero results produces an empty `TvShowSearchResult`.
- Test that TMDB returning multiple results produces a correctly scored and ordered list.

## Notes for Tony Stark

- Implement `ITvShowSearchService` and `TvShowSearchService` in `Scoutarr.Core`.
- Inject `ITmdbClient` and `ConfidenceScorer` via constructor.
- Call `SearchTvShowAsync` from `ITmdbClient`, then score each result using `ConfidenceScorer`.
- Map `TvShowSearchResult` (TMDB type from UC2-TASK-002) to `TvShowCandidate` after scoring.
- Do not apply the confidence threshold here.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV show search service
  As the Core identification service
  I want to search TMDB for TV show candidates and score them
  So that the identification service can select or present the best match

  # ─────────────────────────────────────────
  # BASIC SEARCH AND SCORING
  # ─────────────────────────────────────────

  Scenario: Single result above threshold
    Given the parsed series title is "Breaking Bad" with year 2008
    And TMDB returns one result: "Breaking Bad" (2008), high popularity
    When the search service is called
    Then the result contains one candidate with Name "Breaking Bad"
    And the candidate has a high confidence score

  Scenario: Multiple results are ordered by confidence descending
    Given the parsed series title is "The Office" with no year
    And TMDB returns two results: "The Office" (2005) with high popularity and "The Office UK" (2001) with low popularity
    When the search service is called
    Then the first candidate is "The Office"
    And the second candidate is "The Office UK"
    And the first candidate has a higher confidence than the second

  Scenario: TMDB returns zero results
    Given the parsed series title is "Nonexistent Show XYZ" with no year
    And TMDB returns no results
    When the search service is called
    Then the result contains an empty candidates list

  # ─────────────────────────────────────────
  # YEAR MATCHING
  # ─────────────────────────────────────────

  Scenario: Year present in parsed input and matching TMDB result — score boosted
    Given the parsed series title is "Breaking Bad" with year 2008
    And TMDB returns "Breaking Bad" with first air year 2008
    When the search service is called
    Then the candidate confidence is higher than when year is absent

  Scenario: Year present in parsed input but not matching TMDB result — score penalised
    Given the parsed series title is "Breaking Bad" with year 2010
    And TMDB returns "Breaking Bad" with first air year 2008
    When the search service is called
    Then the candidate confidence is lower than when year matches

  Scenario: Year absent in parsed input — year component contributes zero
    Given the parsed series title is "Breaking Bad" with no year
    And TMDB returns "Breaking Bad" with first air year 2008
    When the search service is called
    Then a candidate is returned with Name "Breaking Bad"
    And the confidence reflects only title similarity and popularity

  # ─────────────────────────────────────────
  # THRESHOLD BEHAVIOUR
  # ─────────────────────────────────────────

  Scenario: Candidates below confidence threshold are still returned
    Given the parsed series title is "Bad" with no year
    And TMDB returns one result: "Breaking Bad" (2008) with low title similarity
    When the search service is called
    Then the result contains one candidate
    And the candidate confidence is below the configured threshold
```

---

## Subtasks

- [ ] Define `TvShowCandidate` and `TvShowSearchResult` records in `Scoutarr.Core`
- [ ] Define `ITvShowSearchService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvShowSearchService`
- [ ] Hawkeye reviews
