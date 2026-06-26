# UC2-TASK-006 — TV episode lookup service

**Requirements:** [identification.md](identification.md) — "Information extracted from TMDB"
**Dependencies:** UC2-TASK-002 (ITmdbClient), UC2-TASK-005 (ParsedEpisodeNumber)

---

## Context

Given a confirmed TMDB series ID and a parsed season/episode number, retrieve the episode title from TMDB.

This task runs after the series is confirmed (UC2-TASK-004) and the episode number is extracted (UC2-TASK-005). It is the final Core step before formatting — its output feeds directly into the filename formatter (UC2-TASK-007).

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

TvEpisodeLookupError   (reuses shared domain error model)
{
    Reason:   string
    Detail:   string?
}
```

---

## Notes for Black Widow

- Mock `ITmdbClient` for all tests.
- Test successful lookup: valid tvId, season, and episode → `TvEpisodeLookupSuccess` with correct fields.
- Test episode not found: TMDB returns no episode for the given season/episode number → `TvEpisodeLookupError`.
- Test season not found: TMDB returns no season for the given number → `TvEpisodeLookupError`.
- Test TMDB unreachable → `TvEpisodeLookupError`.

## Notes for Tony Stark

- Implement `ITvEpisodeLookupService` and `TvEpisodeLookupService` in `Scoutarr.Core`.
- Inject `ITmdbClient` via constructor.
- Call `GetTvShowDetailsAsync` to verify the season and episode exist before fetching episode details. If not found, return `TvEpisodeLookupError` with reason "Episode not found".
- Add `GetEpisodeDetailsAsync(int tvId, int season, int episode)` to `ITmdbClient` and implement it in `TmdbClient`. This method returns the episode title (and any other fields needed downstream).
- `SeriesName` is fetched from `TvShowDetails.Name` — already retrieved in the season validation step, no extra TMDB call needed.

---

## New method on ITmdbClient

```
GetEpisodeDetailsAsync(int tvId, int season, int episode, string language, CancellationToken ct)
    → TvEpisodeDetails

TvEpisodeDetails
{
    EpisodeTitle:  string
}
```

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV episode lookup service
  As the Core episode identification service
  I want to retrieve the episode title from TMDB
  So that the filename formatter has the correct title to produce the output filename

  # ─────────────────────────────────────────
  # SUCCESSFUL LOOKUP
  # ─────────────────────────────────────────

  Scenario: Valid series, season, and episode — success returned
    Given tvId 1396, season 1, episode 2
    And TMDB confirms season 1 exists with at least 2 episodes
    And TMDB returns episode title "Cat's in the Bag"
    When LookupAsync is called
    Then the result is TvEpisodeLookupSuccess
    And the episode title is "Cat's in the Bag"
    And the season is 1 and the episode is 2

  Scenario: SeriesName is populated from TvShowDetails
    Given tvId 1396, season 1, episode 2
    And TMDB returns series name "Breaking Bad"
    When LookupAsync is called
    Then the result SeriesName is "Breaking Bad"

  # ─────────────────────────────────────────
  # NOT FOUND CASES
  # ─────────────────────────────────────────

  Scenario: Season does not exist — error returned
    Given tvId 1396, season 99, episode 1
    And TMDB confirms season 99 does not exist for this series
    When LookupAsync is called
    Then the result is TvEpisodeLookupError with reason "Episode not found"

  Scenario: Episode number exceeds season episode count — error returned
    Given tvId 1396, season 1, episode 99
    And TMDB confirms season 1 exists but has only 7 episodes
    When LookupAsync is called
    Then the result is TvEpisodeLookupError with reason "Episode not found"

  # ─────────────────────────────────────────
  # SEASON 0 — SPECIALS
  # ─────────────────────────────────────────

  Scenario: Valid season 0 episode — success returned
    Given tvId 1396, season 0, episode 1
    And TMDB confirms season 0 exists with at least 1 episode
    And TMDB returns episode title "Pilot (Unaired)"
    When LookupAsync is called
    Then the result is TvEpisodeLookupSuccess
    And the season is 0 and the episode is 1
    And the episode title is "Pilot (Unaired)"

  Scenario: Season 0 episode number exceeds special count — error returned
    Given tvId 1396, season 0, episode 99
    And TMDB confirms season 0 exists but has only 3 episodes
    When LookupAsync is called
    Then the result is TvEpisodeLookupError with reason "Episode not found"

  Scenario: Series has no season 0 — error returned
    Given tvId 1396, season 0, episode 1
    And TMDB confirms season 0 does not exist for this series
    When LookupAsync is called
    Then the result is TvEpisodeLookupError with reason "Episode not found"

  # ─────────────────────────────────────────
  # ERROR CASES
  # ─────────────────────────────────────────

  Scenario: TMDB is unreachable — error returned
    Given tvId 1396, season 1, episode 2
    And TMDB throws a network exception
    When LookupAsync is called
    Then the result is TvEpisodeLookupError with reason "TMDB API unreachable"
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
