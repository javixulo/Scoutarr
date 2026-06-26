# UC1-TASK-004 — Core orchestration — identify movie

**Requirements:**
- [requirements/identification.md](../requirements/identification.md) — "Confidence and automatic acceptance", "Disambiguation flow", "Optional parameters", "No results"
- [requirements/interfaces.md](../requirements/interfaces.md) — "Success responses", "Error responses"

**Dependencies:** UC1-TASK-001 (parser), UC1-TASK-002 (search + scoring), UC1-TASK-003 (formatter)

---

## Context

`MovieIdentificationService` is the central orchestrator for movie identification. It wires together the parser, the TMDB search, and the formatter into a single operation that the REST API and MCP layers call.

The Core does not know about HTTP or MCP — it returns a typed result that each interface translates into its own response format.

---

## Flow

```
1. Parse filename → ParsedMovieFilename (UC1-TASK-001)
   └─ on failure → return domain error

2. If candidateIndex is provided:
   └─ select candidate from previous candidates list by index → skip to step 4
   └─ if index out of range → return domain error

3. Search TMDB with parsed title + year + optional parameters → MovieSearchResult (UC1-TASK-002)
   └─ on zero results → return domain error
   └─ on TMDB unreachable → return domain error

4. Check confidence of top candidate against threshold:
   └─ if confidence >= threshold → proceed to step 5
   └─ if confidence < threshold → return disambiguation result (top 10 candidates, no filename produced)

5. Format output filename using top candidate + template (UC1-TASK-003)
   └─ on failure → return domain error

6. Return identification result
```

---

## Input

```csharp
MovieIdentificationRequest
{
    Filename:         string
    CandidateIndex:   int?
    Year:             int?
    OriginalLanguage: string?
    Genre:            string?
}
```

## Output (discriminated union)

```csharp
MovieIdentificationSuccess
{
    OriginalFilename:  string
    SuggestedFilename: string
    Match: { TmdbId, Title, Year, Confidence }
}

MovieDisambiguationNeeded
{
    OriginalFilename: string
    Candidates:       IReadOnlyList<MovieCandidate>   // top 10
}

MovieIdentificationError
{
    Reason: string
    Detail: string
}
```

---

## Notes for Black Widow

- Mock `IMovieFilenameParser`, `IMovieSearchService`, and `IMovieFilenameFormatter`.
- Test the orchestration logic: correct flow branching, correct output type returned in each scenario.
- Test that `SuggestedFilename` preserves the original file extension.
- Test `CandidateIndex` boundary cases (0, valid, out of range).

## Notes for Tony Stark

- Implement `IMovieIdentificationService` in `Scoutarr.Core`.
- The confidence threshold comes from configuration.
- `CandidateIndex` requires the previous candidate list to be passed in the request — Core does not store state between requests.
- The suggested filename extension must match the original file extension exactly.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Core movie identification orchestration
  As a REST or MCP interface
  I want to call a single service to identify a movie filename
  So that I receive either a match, a disambiguation list, or a clear error

  Scenario: High confidence match is automatically accepted
    Given the filename "The.Batman.2022.1080p.mkv"
    And the TMDB search returns "The Batman" (2022) with confidence 0.97
    And the confidence threshold is 0.80
    When identification is requested
    Then the result is a success
    And the suggested filename is "The Batman (2022).mkv"

  Scenario: Low confidence triggers disambiguation response
    Given the filename "Batman.mkv"
    And the TMDB search returns multiple candidates all below confidence 0.80
    When identification is requested
    Then the result is a disambiguation response
    And the response contains up to 10 candidates ordered by confidence descending

  Scenario: candidateIndex selects a candidate and bypasses threshold
    Given a previous disambiguation returned 7 candidates
    And the request includes candidateIndex 3
    When identification is requested
    Then the third candidate is used directly

  Scenario: candidateIndex out of range returns error
    Given a previous disambiguation returned 5 candidates
    And the request includes candidateIndex 6
    When identification is requested
    Then the result is an error with reason "Candidate index out of range"

  Scenario: TMDB is unreachable
    Given the filename "The.Batman.2022.mkv"
    And the TMDB client is unreachable
    When identification is requested
    Then the result is an error with reason "TMDB API unreachable"
```

---

## Subtasks

- [ ] Define `MovieIdentificationRequest`, `MovieIdentificationSuccess`, `MovieDisambiguationNeeded`, `MovieIdentificationError` records in `Scoutarr.Core`
- [ ] Define `IMovieIdentificationService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `MovieIdentificationService`
- [ ] Hawkeye reviews
