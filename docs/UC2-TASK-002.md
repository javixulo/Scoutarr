# UC2-TASK-002 — Extend ITmdbClient with TV show methods

**Requirements:** [identification.md](identification.md) — "Metadata source"
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
    Name:              string       // TMDB "name" field
    OriginalName:      string       // TMDB "original_name" field
    FirstAirYear:      int?         // extracted from TMDB "first_air_date"
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
    Cast:      IReadOnlyList<string>    // top 5 cast names
    Network:   string?
    Genres:    IReadOnlyList<string>
    Status:    string                   // as returned by TMDB: "Returning Series", "Ended", "Canceled", etc.
}
```

`TvShowDetails` is used downstream by the episode parser (UC2-TASK-005) to validate season/episode numbers from compact numeric filenames. Only season number and episode count are needed — no episode titles here.

---

## Notes for Black Widow

- Mock `ITmdbClient` in all downstream tests — this task's own tests are integration tests against the real `TmdbClient`, run only in E2E.
- Unit tests for this task focus on the mapping layer: given a raw TMDB JSON response, verify that `TvShowSearchResult` and `TvShowDetails` are correctly mapped.
- Test `FirstAirYear` extraction: valid date, null date, malformed date.
- Test `TvSeasonInfo` mapping: multiple seasons, season 0 (specials) — verify season 0 is included in the list (downstream callers decide whether to use it).
- Mock realistic TMDB response structures as defined in `skills/tmdb-api/SKILL.md`.

## Notes for Tony Stark

- Add `SearchTvShowAsync` and `GetTvShowDetailsAsync` to the `ITmdbClient` interface in `Scoutarr.Core`.
- Implement both methods in `TmdbClient` using TMDbLib.
- `SearchTvShowAsync` maps to TMDbLib `SearchTvShowAsync` — pass `language` and `firstAirDateYear` if year is provided.
- `GetTvShowDetailsAsync` maps to TMDbLib `GetTvShowAsync` — map the `Seasons` list to `TvSeasonInfo`.
- `GetTvShowEnrichmentAsync` maps to TMDbLib `GetTvShowAsync` — extract `created_by[0].name` for Creator, top 5 from `credits.cast` for Cast, `networks[0].name` for Network, `genres` for Genres, and `status` for Status.
- `FirstAirYear`: parse from `first_air_date` string (format `YYYY-MM-DD`). Return null if absent or unparseable.
- Do not filter out season 0 — return it as-is.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: ITmdbClient TV show methods
  As the Core search and parser services
  I want to search for TV shows and retrieve their season structure from TMDB
  So that series candidates and episode validation are based on real TMDB data

  # ─────────────────────────────────────────
  # SearchTvShowAsync — response mapping
  # ─────────────────────────────────────────

  Scenario: TMDB response is correctly mapped to TvShowSearchResult list
    Given the TMDB search response contains a show with name "Breaking Bad", original_name "Breaking Bad", first_air_date "2008-01-20", popularity 150.0
    When SearchTvShowAsync is called
    Then the result contains a TvShowSearchResult with Name "Breaking Bad"
    And FirstAirYear is 2008
    And Popularity is 150.0

  Scenario: first_air_date is null — FirstAirYear is null
    Given the TMDB search response contains a show with first_air_date null
    When SearchTvShowAsync is called
    Then the result contains a TvShowSearchResult with FirstAirYear null

  Scenario: first_air_date is malformed — FirstAirYear is null
    Given the TMDB search response contains a show with first_air_date "unknown"
    When SearchTvShowAsync is called
    Then the result contains a TvShowSearchResult with FirstAirYear null

  Scenario: Multiple results are returned in the order TMDB provides them
    Given the TMDB search response contains three shows
    When SearchTvShowAsync is called
    Then the result contains all three shows in the same order

  Scenario: TMDB returns zero results
    Given the TMDB search response contains no shows
    When SearchTvShowAsync is called
    Then the result is an empty list

  # ─────────────────────────────────────────
  # GetTvShowDetailsAsync — response mapping
  # ─────────────────────────────────────────

  Scenario: Season list is correctly mapped to TvSeasonInfo
    Given the TMDB show details response contains seasons: season 1 with 7 episodes, season 2 with 13 episodes
    When GetTvShowDetailsAsync is called
    Then the result contains TvSeasonInfo for season 1 with EpisodeCount 7
    And TvSeasonInfo for season 2 with EpisodeCount 13

  Scenario: Season 0 (specials) is included in the result
    Given the TMDB show details response contains season 0 with 5 episodes and season 1 with 10 episodes
    When GetTvShowDetailsAsync is called
    Then the result contains TvSeasonInfo for season 0 with EpisodeCount 5
    And TvSeasonInfo for season 1 with EpisodeCount 10

  Scenario: Show with a single season is correctly mapped
    Given the TMDB show details response contains only season 1 with 6 episodes
    When GetTvShowDetailsAsync is called
    Then the result contains exactly one TvSeasonInfo for season 1 with EpisodeCount 6

  # ─────────────────────────────────────────
  # GetTvShowEnrichmentAsync — response mapping
  # ─────────────────────────────────────────

  Scenario: Enrichment fields are correctly mapped
    Given the TMDB show details response contains creator "Vince Gilligan", network "AMC", status "Ended", genres ["Drama", "Crime"], and cast with 7 members
    When GetTvShowEnrichmentAsync is called
    Then the result contains Creator "Vince Gilligan"
    And Network "AMC"
    And Status "Ended"
    And Genres ["Drama", "Crime"]
    And Cast contains exactly 5 members (top 5 only)

  Scenario: Creator absent — Creator is null
    Given the TMDB show details response contains no created_by entries
    When GetTvShowEnrichmentAsync is called
    Then the result contains Creator null

  Scenario: Cast has fewer than 5 members — all are returned
    Given the TMDB show details response contains cast with 3 members
    When GetTvShowEnrichmentAsync is called
    Then the result contains Cast with exactly 3 members

  # ─────────────────────────────────────────
  # ERROR CASES
  # ─────────────────────────────────────────

  Scenario: TMDB is unreachable during search
    Given the TMDB client throws a network exception on SearchTvShowAsync
    When SearchTvShowAsync is called
    Then a domain error is returned with reason "TMDB API unreachable"

  Scenario: TMDB is unreachable during details fetch
    Given the TMDB client throws a network exception on GetTvShowDetailsAsync
    When GetTvShowDetailsAsync is called
    Then a domain error is returned with reason "TMDB API unreachable"
```

---

## Subtasks

- [ ] Add `TvShowSearchResult`, `TvShowDetails`, `TvSeasonInfo`, and `TvShowEnrichment` records to `Scoutarr.Core`
- [ ] Add `SearchTvShowAsync`, `GetTvShowDetailsAsync`, and `GetTvShowEnrichmentAsync` to `ITmdbClient`
- [ ] Black Widow writes tests in red (mapping scenarios above)
- [ ] Tony Stark implements all three methods in `TmdbClient`
- [ ] Hawkeye reviews
