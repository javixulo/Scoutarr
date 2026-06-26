# UC2-TASK-002 — Extend ITmdbClient with TV show methods

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Metadata source"
**Skill:** [skills/tmdb-api/SKILL.md](skills/tmdb-api/SKILL.md)
**Dependencies:** TASK-002 (ITmdbClient definition)

---

## Context

`ITmdbClient` was defined in TASK-002 with a single method: `SearchMovieAsync`. This task extends the interface and its real implementation (`TmdbClient`) with two new methods needed by UC-2:

- `SearchTvShowAsync` — to find TV show candidates by title
- `GetTvShowDetailsAsync` — to retrieve season and episode counts for a confirmed series

No search service logic. No confidence scoring. No orchestration. This task is purely about the TMDB client contract and its real HTTP implementation.

---

## New methods on ITmdbClient

```
SearchTvShowAsync(string title, int? year, string language, CancellationToken ct)
    → IReadOnlyList<TvShowSearchResult>

GetTvShowDetailsAsync(int tvId, string language, CancellationToken ct)
    → TvShowDetails

GetTvShowEnrichmentAsync(int tvId, string language, CancellationToken ct)
    → TvShowEnrichment
```

---

## Output types

```
TvShowSearchResult
{
    TmdbId:            int
    Name:              string
    OriginalName:      string
    FirstAirYear:      int?
    Overview:          string?
    OriginalLanguage:  string
    Popularity:        double
}

TvShowDetails
{
    TmdbId:       int
    Name:         string
    Seasons:      IReadOnlyList<TvSeasonInfo>
}

TvSeasonInfo
{
    SeasonNumber:  int
    EpisodeCount:  int
}

TvShowEnrichment
{
    Creator:   string?
    Cast:      IReadOnlyList<string>
    Network:   string?
    Genres:    IReadOnlyList<string>
    Status:    string
}
```

---

## Notes for Black Widow

- Mock `ITmdbClient` in all downstream tests.
- Unit tests for this task focus on the mapping layer: given a raw TMDB JSON response, verify that `TvShowSearchResult` and `TvShowDetails` are correctly mapped.
- Test `FirstAirYear` extraction: valid date, null date, malformed date.
- Test `TvSeasonInfo` mapping: multiple seasons, season 0 (specials) included.
- Mock realistic TMDB response structures as defined in `skills/tmdb-api/SKILL.md`.

## Notes for Tony Stark

- Add `SearchTvShowAsync` and `GetTvShowDetailsAsync` to the `ITmdbClient` interface in `Scoutarr.Core`.
- Implement both methods in `TmdbClient` using TMDbLib.
- `FirstAirYear`: parse from `first_air_date` string (format `YYYY-MM-DD`). Return null if absent or unparseable.
- Do not filter out season 0 — return it as-is.

---

## Subtasks

- [ ] Add `TvShowSearchResult`, `TvShowDetails`, `TvSeasonInfo`, and `TvShowEnrichment` records to `Scoutarr.Core`
- [ ] Add `SearchTvShowAsync`, `GetTvShowDetailsAsync`, and `GetTvShowEnrichmentAsync` to `ITmdbClient`
- [ ] Black Widow writes tests in red (mapping scenarios)
- [ ] Tony Stark implements all three methods in `TmdbClient`
- [ ] Hawkeye reviews
