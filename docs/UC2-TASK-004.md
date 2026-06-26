# UC2-TASK-004 — TV show identification service (Core)

**Requirements:** [identification.md](identification.md) — "Disambiguation flow", "Confidence threshold"
**Dependencies:** UC2-TASK-001 (ITvSeriesTitleParser), UC2-TASK-003 (ITvShowSearchService)

---

## Context

Orchestrates the full TV series identification flow: parse the filename, search TMDB, and return either a confirmed match or a disambiguation result for the caller to resolve.

This is the direct TV equivalent of `IMovieIdentificationService` from TASK-004. The logic is identical — the only differences are the types involved (TV show vs movie) and that the result carries series-specific metadata.

Core does not enrich candidates with additional TMDB data. Enrichment (creator, cast, network, seasons, status) is the responsibility of the MCP layer (UC2-TASK-008).

No episode identification. No file renaming. No REST or MCP concerns.

---

## Interface

```
ITvShowIdentificationService

IdentifyAsync(TvShowIdentificationRequest request, CancellationToken ct)
    → TvShowIdentificationResult
```

---

## Input type

```
TvShowIdentificationRequest
{
    Filename:          string       // dirty filename
    TmdbId:            int?         // optional; bypasses search entirely when provided
    Year:              int?         // optional hint
    OriginalLanguage:  string?      // optional hint
}
```

---

## Output types

```
TvShowIdentificationResult  (discriminated union)
    | TvShowIdentificationSuccess
    | TvShowDisambiguationNeeded
    | TvShowIdentificationError

TvShowIdentificationSuccess
{
    OriginalFilename:  string
    TmdbId:            int
    Name:              string
    Year:              int?
    Confidence:        double
}

TvShowDisambiguationNeeded
{
    OriginalFilename:  string
    Candidates:        IReadOnlyList<TvShowCandidate>   // ordered by confidence descending
}

TvShowIdentificationError   (reuses shared domain error model)
{
    Reason:   string
    Detail:   string?
}
```

---

## Orchestration flow

```
1. If TmdbId is provided → skip parse and search, call GetTvShowDetailsAsync directly → return TvShowIdentificationSuccess
2. Parse filename via ITvSeriesTitleParser → extract ParsedTvSeriesTitle
3. Search TMDB via ITvShowSearchService → get ordered TvShowCandidate list
4. If no candidates → return TvShowIdentificationError (no results)
5. If top candidate confidence ≥ CONFIDENCE_THRESHOLD → return TvShowIdentificationSuccess
6. If top candidate confidence < CONFIDENCE_THRESHOLD → return TvShowDisambiguationNeeded with full candidate list
```

---

## Notes for Black Widow

- Mock `ITvSeriesTitleParser`, `ITvShowSearchService`, and `ITmdbClient`.
- Test the `TmdbId` bypass: when provided, parser and search service are never called.
- Test auto-accept: top candidate at or above threshold → `TvShowIdentificationSuccess`.
- Test disambiguation: top candidate below threshold → `TvShowDisambiguationNeeded` with all candidates.
- Test no candidates from search → `TvShowIdentificationError`.
- Test TMDB unreachable → `TvShowIdentificationError`.
- Test with optional hints: verify they are passed through to the search service.

## Notes for Tony Stark

- Implement `ITvShowIdentificationService` and `TvShowIdentificationService` in `Scoutarr.Core`.
- Inject `ITvSeriesTitleParser`, `ITvShowSearchService`, and `ITmdbClient` via constructor.
- `CONFIDENCE_THRESHOLD` is read from configuration — same mechanism as UC1.
- When `TmdbId` is provided, call `GetTvShowDetailsAsync` to confirm the series exists and populate `TvShowIdentificationSuccess`. Return error if the ID is not found on TMDB.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV show identification service
  As the REST API and MCP server
  I want to identify the TV series from a dirty filename
  So that the caller can confirm the match or resolve disambiguation before proceeding to episode identification

  # ─────────────────────────────────────────
  # AUTO-ACCEPT
  # ─────────────────────────────────────────

  Scenario: Top candidate above threshold — success returned
    Given the filename "Breaking.Bad.S01E02.mkv"
    And the search service returns one candidate "Breaking Bad" (2008) with confidence 0.95
    And 0.95 is above the configured threshold
    When IdentifyAsync is called
    Then the result is TvShowIdentificationSuccess
    And the result name is "Breaking Bad"
    And the result year is 2008
    And the result confidence is 0.95

  Scenario: Top candidate exactly at threshold — success returned
    Given the filename "Breaking.Bad.S01E02.mkv"
    And the search service returns one candidate with confidence equal to the configured threshold
    When IdentifyAsync is called
    Then the result is TvShowIdentificationSuccess

  # ─────────────────────────────────────────
  # DISAMBIGUATION
  # ─────────────────────────────────────────

  Scenario: Top candidate below threshold — disambiguation returned
    Given the filename "The.Office.S01E01.mkv"
    And the search service returns two candidates, both below the configured threshold
    When IdentifyAsync is called
    Then the result is TvShowDisambiguationNeeded
    And the result contains both candidates ordered by confidence descending

  Scenario: Disambiguation result contains all candidates, not just the top one
    Given the filename "The.Office.S01E01.mkv"
    And the search service returns three candidates all below threshold
    When IdentifyAsync is called
    Then the result is TvShowDisambiguationNeeded
    And the result contains all three candidates

  # ─────────────────────────────────────────
  # TMDB ID BYPASS
  # ─────────────────────────────────────────

  Scenario: TmdbId provided — parser and search are skipped
    Given the request includes TmdbId 1396
    When IdentifyAsync is called
    Then ITvSeriesTitleParser is never called
    And ITvShowSearchService is never called
    And the result is TvShowIdentificationSuccess with TmdbId 1396

  Scenario: TmdbId provided but not found on TMDB — error returned
    Given the request includes TmdbId 999999
    And TMDB returns no details for that ID
    When IdentifyAsync is called
    Then the result is TvShowIdentificationError with reason "TMDB ID not found"

  # ─────────────────────────────────────────
  # OPTIONAL HINTS
  # ─────────────────────────────────────────

  Scenario: Year hint is passed through to search service
    Given the request includes filename "The.Office.S01E01.mkv" and year hint 2005
    When IdentifyAsync is called
    Then the search service is called with year 2005

  Scenario: OriginalLanguage hint is passed through to search service
    Given the request includes filename "La.Casa.De.Papel.S01E01.mkv" and originalLanguage hint "es"
    When IdentifyAsync is called
    Then the search service is called with originalLanguage "es"

  # ─────────────────────────────────────────
  # ERROR CASES
  # ─────────────────────────────────────────

  Scenario: Search service returns no candidates — error returned
    Given the filename "Xzqfnmvp.S01E01.mkv"
    And the search service returns an empty candidate list
    When IdentifyAsync is called
    Then the result is TvShowIdentificationError with reason "No TMDB results found"

  Scenario: TMDB is unreachable — error returned
    Given the filename "Breaking.Bad.S01E02.mkv"
    And the search service throws a TMDB unreachable error
    When IdentifyAsync is called
    Then the result is TvShowIdentificationError with reason "TMDB API unreachable"

  Scenario: Filename parser fails — error returned
    Given the filename is an empty string
    When IdentifyAsync is called
    Then the result is TvShowIdentificationError with reason "No series title found"
```

---

## Subtasks

- [ ] Define `TvShowIdentificationRequest`, `TvShowIdentificationSuccess`, `TvShowDisambiguationNeeded` records in `Scoutarr.Core`
- [ ] Define `ITvShowIdentificationService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvShowIdentificationService`
- [ ] Hawkeye reviews
