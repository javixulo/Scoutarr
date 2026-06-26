# UC2-TASK-006 — TV episode lookup service

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Information extracted from TMDB"
**Dependencies:** UC2-TASK-002 (ITmdbClient), UC2-TASK-005 (ParsedEpisodeNumber)

---

## Context

Given a confirmed TMDB series ID and a parsed season/episode number, retrieve the episode title from TMDB.

No series identification. No filename parsing. No confidence scoring. No output formatting. No file renaming.

---

## Interface

```
ITvEpisodeLookupService

LookupAsync(int tvId, int season, int episode, CancellationToken ct)
    → TvEpisodeLookupResult
```

---

## Output types

```
TvEpisodeLookupResult  (discriminated union)
    | TvEpisodeLookupSuccess
    | TvEpisodeLookupError

TvEpisodeLookupSuccess
{
    TmdbId:        int
    SeriesName:    string
    Season:        int
    Episode:       int
    EpisodeTitle:  string
}

TvEpisodeLookupError
{
    Reason:   string
    Detail:   string?
}
```

---

## Subtasks

- [ ] Add `GetEpisodeDetailsAsync` to `ITmdbClient` and implement in `TmdbClient`
- [ ] Define `TvEpisodeDetails` record in `Scoutarr.Core`
- [ ] Define `TvEpisodeLookupSuccess` and `TvEpisodeLookupError` records in `Scoutarr.Core`
- [ ] Define `ITvEpisodeLookupService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvEpisodeLookupService`
- [ ] Hawkeye reviews
