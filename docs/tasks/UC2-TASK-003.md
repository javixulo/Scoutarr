# UC2-TASK-003 — TV show search service

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Candidate scoring", "Confidence threshold"
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
    Confidence:  double
}

TvShowSearchResult
{
    Candidates:  IReadOnlyList<TvShowCandidate>
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

Uses `ConfidenceScorer` (extracted in TASK-002) with the same formula:
- 60% title similarity
- 30% year match
- 10% popularity

Candidates below `CONFIDENCE_THRESHOLD` are included — filtering is the responsibility of the identification service.

---

## Notes for Black Widow

- Mock `ITmdbClient` for all tests.
- Test that candidates are ordered by confidence descending.
- Test year match component: year matching, not matching, absent in parsed input, absent in TMDB result.
- Test that candidates below threshold are still returned.
- Test that TMDB zero results produces an empty `TvShowSearchResult`.

## Notes for Tony Stark

- Implement `ITvShowSearchService` and `TvShowSearchService` in `Scoutarr.Core`.
- Inject `ITmdbClient` and `ConfidenceScorer` via constructor.
- Do not apply the confidence threshold here.

---

## Subtasks

- [ ] Define `TvShowCandidate` and `TvShowSearchResult` records in `Scoutarr.Core`
- [ ] Define `ITvShowSearchService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvShowSearchService`
- [ ] Hawkeye reviews
