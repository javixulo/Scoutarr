# TASK-004 — Core orchestration — identify movie

**Requirements:**
- [requirements/identification.md](../requirements/identification.md) — "Confidence and automatic acceptance", "Disambiguation flow", "Optional parameters", "No results"
- [requirements/interfaces.md](../requirements/interfaces.md) — "Success responses", "Error responses"

**Dependencies:** TASK-001 (parser), TASK-002 (search + scoring), TASK-003 (formatter)

---

## Context

`MovieIdentificationService` is the central orchestrator for movie identification. It wires together the parser, the TMDB search, and the formatter into a single operation that the REST API and MCP layers call.

The Core does not know about HTTP or MCP — it returns a typed result that each interface translates into its own response format.

---

## Flow

```
1. Parse filename → ParsedMovieFilename (TASK-001)
   └─ on failure → return domain error

2. If candidateIndex is provided:
   └─ select candidate from previous candidates list by index → skip to step 4
   └─ if index out of range → return domain error

3. Search TMDB with parsed title + year + optional parameters → MovieSearchResult (TASK-002)
   └─ on zero results → return domain error
   └─ on TMDB unreachable → return domain error

4. Check confidence of top candidate against threshold:
   └─ if confidence >= threshold → proceed to step 5
   └─ if confidence < threshold → return disambiguation result (top 10 candidates, no filename produced)

5. Format output filename using top candidate + template (TASK-003)
   └─ on failure → return domain error

6. Return identification result
```

---

## Input

```csharp
MovieIdentificationRequest
{
    Filename:         string   // dirty filename including extension
    CandidateIndex:   int?     // 1-based; selects from a previous disambiguation response
    Year:             int?     // optional hint
    OriginalLanguage: string?  // optional hint (e.g. "en", "es")
    Genre:            string?  // optional hint
}
```

## Output (discriminated union)

```csharp
// Success — match found above threshold
MovieIdentificationSuccess
{
    OriginalFilename:  string
    SuggestedFilename: string   // formatted stem + original extension
    Match:
    {
        TmdbId:     int
        Title:      string
        Year:       int
        Confidence: double
    }
}

// Disambiguation needed — no candidate above threshold
MovieDisambiguationNeeded
{
    OriginalFilename: string
    Candidates:       IReadOnlyList<MovieCandidate>   // top 10, ordered by confidence
}

// Failure — domain error
MovieIdentificationError
{
    Reason: string
    Detail: string
}
```

---

## Notes for Black Widow

- Mock `IMovieFilenameParser`, `IMovieSearchService`, and `IMovieFilenameFormatter` — do not test their internals here.
- Test the orchestration logic: correct flow branching, correct output type returned in each scenario.
- Test that `SuggestedFilename` preserves the original file extension.
- Test `CandidateIndex` boundary cases (0, valid, out of range).

## Notes for Tony Stark

- Implement `IMovieIdentificationService` in `Scoutarr.Core`.
- The confidence threshold comes from configuration — inject it via `IConfiguration` or a typed options class.
- `CandidateIndex` requires the previous candidate list to be passed in the request — the caller (REST or MCP layer) is responsible for keeping that list between calls and passing it back. Core does not store state between requests.
- The suggested filename extension must match the original file extension exactly.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Core movie identification orchestration
  As a REST or MCP interface
  I want to call a single service to identify a movie filename
  So that I receive either a match, a disambiguation list, or a clear error

  # ─────────────────────────────────────────
  # HAPPY PATH — automatic acceptance
  # ─────────────────────────────────────────

  Scenario: High confidence match is automatically accepted
    Given the filename "The.Batman.2022.1080p.mkv"
    And the TMDB search returns "The Batman" (2022) with confidence 0.97
    And the confidence threshold is 0.80
    When identification is requested
    Then the result is a success
    And the suggested filename is "The Batman (2022).mkv"
    And the match title is "The Batman" and year is 2022
    And the match confidence is 0.97

  Scenario: Suggested filename preserves original extension
    Given the filename "The.Batman.2022.1080p.mp4"
    And the TMDB search returns "The Batman" (2022) with confidence 0.97
    And the confidence threshold is 0.80
    When identification is requested
    Then the suggested filename ends with ".mp4"

  Scenario: Optional year hint is passed to TMDB search
    Given the filename "Batman.1080p.mkv"
    And the request includes year hint 2022
    And the TMDB search is called with title "Batman" and year 2022
    And returns "The Batman" (2022) with confidence 0.85
    When identification is requested
    Then the result is a success
    And the match title is "The Batman"

  # ─────────────────────────────────────────
  # DISAMBIGUATION — confidence below threshold
  # ─────────────────────────────────────────

  Scenario: Low confidence triggers disambiguation response
    Given the filename "Batman.mkv"
    And the TMDB search returns multiple candidates all below confidence 0.80
    And the confidence threshold is 0.80
    When identification is requested
    Then the result is a disambiguation response
    And the response contains up to 10 candidates ordered by confidence descending

  Scenario: candidateIndex selects a candidate and bypasses threshold
    Given the filename "Batman.mkv"
    And a previous disambiguation returned 7 candidates
    And the request includes candidateIndex 3
    When identification is requested
    Then the third candidate is used directly
    And the result is a success regardless of its confidence score

  Scenario: candidateIndex 1 selects the first candidate
    Given a previous disambiguation returned 5 candidates
    And the request includes candidateIndex 1
    When identification is requested
    Then the first candidate is used

  Scenario: candidateIndex out of range returns error
    Given a previous disambiguation returned 5 candidates
    And the request includes candidateIndex 6
    When identification is requested
    Then the result is an error with reason "Candidate index out of range"

  Scenario: candidateIndex 0 returns error
    Given a previous disambiguation returned 5 candidates
    And the request includes candidateIndex 0
    When identification is requested
    Then the result is an error with reason "Candidate index out of range"

  # ─────────────────────────────────────────
  # ERROR CASES
  # ─────────────────────────────────────────

  Scenario: Filename cannot be parsed
    Given the filename "1080p.mkv"
    When identification is requested
    Then the result is an error with reason "Could not extract title"

  Scenario: TMDB returns zero results
    Given the filename "Zxqfnmvp.2099.mkv"
    And the TMDB search returns zero results
    When identification is requested
    Then the result is an error with reason "No TMDB results found"

  Scenario: TMDB is unreachable
    Given the filename "The.Batman.2022.mkv"
    And the TMDB client is unreachable
    When identification is requested
    Then the result is an error with reason "TMDB API unreachable"
```

---

## Subtasks

- [ ] Define `MovieIdentificationRequest` record in `Scoutarr.Core`
- [ ] Define `MovieIdentificationSuccess`, `MovieDisambiguationNeeded`, `MovieIdentificationError` records in `Scoutarr.Core`
- [ ] Define `IMovieIdentificationService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `MovieIdentificationService`
- [ ] Hawkeye reviews
