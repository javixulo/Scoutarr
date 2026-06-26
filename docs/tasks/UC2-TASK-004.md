# UC2-TASK-004 — TV show identification service (Core)

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow", "Confidence threshold"
**Dependencies:** UC2-TASK-001 (ITvSeriesTitleParser), UC2-TASK-003 (ITvShowSearchService)

---

## Context

Orchestrates the full TV series identification flow: parse the filename, search TMDB, and return either a confirmed match or a disambiguation result for the caller to resolve.

This is the direct TV equivalent of `IMovieIdentificationService` from TASK-004.

Core does not enrich candidates. Enrichment is the responsibility of the MCP layer.

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
    Filename:          string
    TmdbId:            int?
    Year:              int?
    OriginalLanguage:  string?
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
    Candidates:        IReadOnlyList<TvShowCandidate>
}

TvShowIdentificationError
{
    Reason:   string
    Detail:   string?
}
```

---

## Orchestration flow

```
1. If TmdbId is provided → skip parse and search, call GetTvShowDetailsAsync → return TvShowIdentificationSuccess
2. Parse filename via ITvSeriesTitleParser
3. Search TMDB via ITvShowSearchService
4. If no candidates → return TvShowIdentificationError
5. If top candidate confidence ≥ threshold → return TvShowIdentificationSuccess
6. If top candidate confidence < threshold → return TvShowDisambiguationNeeded
```

---

## Subtasks

- [ ] Define `TvShowIdentificationRequest`, `TvShowIdentificationSuccess`, `TvShowDisambiguationNeeded` records in `Scoutarr.Core`
- [ ] Define `ITvShowIdentificationService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvShowIdentificationService`
- [ ] Hawkeye reviews
