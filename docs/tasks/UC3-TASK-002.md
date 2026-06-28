# UC3-TASK-002 — Series Metadata File

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Series state", "Metadata file", "Validation on subsequent passes"

---

## Context

After a series folder is successfully identified, Scoutarr writes a JSON metadata file to the root series folder. On subsequent passes, that file is read instead of re-identifying the series, and its data is used to validate incoming episodes against known season and episode counts.

This task covers:

- Writing the metadata file after a successful series identification
- Reading and deserialising the metadata file on subsequent passes
- Refreshing season and episode data from TMDB for any season still marked as airing
- Refreshing the series and season airing status on every pass, regardless of whether any season is still airing
- Validating an episode number against the known series data

No folder scanning. No renaming. No orchestration. Pure metadata read/write and validation logic.

---

## Output types

```
SeriesMetadata
{
    TmdbId:        int
    Title:         string
    Year:          int
    TotalSeasons:  int
    IsAiring:      bool
    Seasons:       IReadOnlyList<SeasonMetadata>
}

SeasonMetadata
{
    SeasonNumber:  int
    EpisodeCount:  int
    IsAiring:      bool
    IsComplete:    bool
    Episodes:      IReadOnlyList<EpisodeMetadata>
}

EpisodeMetadata
{
    EpisodeNumber:         int
    AbsoluteEpisodeNumber: int?        // null for Season 0 specials
    Title:                 string
    RuntimeMinutes:        int?        // null if not provided by TMDB
    AirDate:               DateOnly?   // null if not provided by TMDB
}

SeriesMetadataReadResult =
    | SeriesMetadataFound(SeriesMetadata metadata)
    | SeriesMetadataNotFound                        // expected on first pass
    | SeriesMetadataError(string reason)            // file exists but is malformed
```

---

## Refresh behaviour

On every pass where a metadata file already exists, Scoutarr:

1. **Always** fetches the current series status from TMDB and updates `IsAiring` on the series. This handles reactivation of cancelled or ended series, as well as series that have ended since the last pass.
2. **Always** fetches the current status of each season from TMDB and updates `IsAiring` and `IsComplete` accordingly. A season that was airing may now be complete; a season that was complete is checked for status changes only, not for new episodes.
3. Refreshes the full episode list (counts, titles, runtimes, air dates) only for seasons where `IsAiring` is still `true` after the status update.

`AbsoluteEpisodeNumber` is recalculated after each refresh, since episode counts in earlier airing seasons may have changed.

---

## TMDB data source

Episode names, runtimes, and air dates are fetched from the `/3/tv/{series_id}/season/{season_number}` endpoint, which returns the full episode list for a season including `name`, `runtime`, and `air_date` per episode. One call per season is required.

Series and season status is fetched from the `/3/tv/{series_id}` endpoint, which returns the overall series status and the list of seasons with their current state.

`AbsoluteEpisodeNumber` is not provided by TMDB — it is calculated by Scoutarr as the sum of all episodes in preceding seasons (ordered by season number, excluding Season 0) plus the episode's own number within its season. Season 0 (specials) is excluded from the absolute count and stored as `null`.

---

## Example JSON

```json
{
  "tmdbId": 1396,
  "title": "Breaking Bad",
  "year": 2008,
  "totalSeasons": 2,
  "isAiring": false,
  "seasons": [
    {
      "seasonNumber": 0,
      "episodeCount": 2,
      "isAiring": false,
      "isComplete": true,
      "episodes": [
        { "episodeNumber": 1, "absoluteEpisodeNumber": null, "title": "Special 1", "runtimeMinutes": 45, "airDate": "2008-01-20" },
        { "episodeNumber": 2, "absoluteEpisodeNumber": null, "title": "Special 2", "runtimeMinutes": 42, "airDate": "2008-02-10" }
      ]
    },
    {
      "seasonNumber": 1,
      "episodeCount": 2,
      "isAiring": false,
      "isComplete": true,
      "episodes": [
        { "episodeNumber": 1, "absoluteEpisodeNumber": 1, "title": "Pilot", "runtimeMinutes": 58, "airDate": "2008-01-20" },
        { "episodeNumber": 2, "absoluteEpisodeNumber": 2, "title": "Cat's in the Bag", "runtimeMinutes": 48, "airDate": "2008-01-27" }
      ]
    },
    {
      "seasonNumber": 2,
      "episodeCount": 2,
      "isAiring": false,
      "isComplete": true,
      "episodes": [
        { "episodeNumber": 1, "absoluteEpisodeNumber": 3, "title": "Seven Thirty-Seven", "runtimeMinutes": 47, "airDate": "2009-03-08" },
        { "episodeNumber": 2, "absoluteEpisodeNumber": 4, "title": "Grilled", "runtimeMinutes": 47, "airDate": "2009-03-15" }
      ]
    }
  ]
}
```

---

## Notes for Black Widow

- All tests are pure — mock `IFileSystem` and `ITmdbClient`, no real I/O.
- Test write: correct filename, correct JSON structure, written to the correct path.
- Test read: valid file returns `SeriesMetadataFound`; missing file returns `SeriesMetadataNotFound`; malformed file returns `SeriesMetadataError`.
- Test series status refresh: always called, regardless of airing state.
- Test season status refresh: always called; `IsAiring` and `IsComplete` are updated correctly.
- Test episode refresh: only called for seasons that are still `IsAiring` after status update; `AirDate` is included in the refreshed episode data.
- Test series reactivation: series was ended/cancelled, TMDB now marks it as airing — `IsAiring` updated to true.
- Test season completion: season was `IsAiring: true`, TMDB now marks it complete — `IsAiring` set to false, `IsComplete` set to true, episode list not re-fetched.
- Test `AbsoluteEpisodeNumber` calculation: correct across multiple seasons, excluding Season 0.
- Test `AbsoluteEpisodeNumber` is recalculated after refresh when episode counts change.
- Test validation: episode within range, episode out of range, season not found.

## Notes for Tony Stark

- Implement `ISeriesMetadataService` and `SeriesMetadataService` in `Scoutarr.Core`.
- Filename format: `{Series Title} ({Year}).json`, written to the root series folder.
- Depends on `IFileSystem` and `ITmdbClient`.
- Use `System.Text.Json` for serialisation — consistent with the rest of the project.
- `AirDate` is serialised as `"yyyy-MM-dd"` string in JSON using `DateOnly` with a custom converter if needed.
- `AbsoluteEpisodeNumber` is stored in the JSON file and only recalculated after a refresh, not on every read.
- Series and season status refresh always happens on every pass — do not skip it even when no seasons are airing.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Series metadata file
  As the Folder Rename Orchestrator
  I want to persist and retrieve series metadata
  So that subsequent passes skip re-identification and validate episodes correctly

  Scenario: Metadata file is written after successful identification
    Given a confirmed series "Breaking Bad" (2008) with TMDB ID 1396
    When the metadata file is written to "/media/tv/Breaking Bad (2008)/"
    Then a file named "Breaking Bad (2008).json" exists at that path
    And each episode has a title, runtime, air date, and absolute episode number where applicable

  Scenario: Metadata file is read on subsequent pass
    Given a valid "Breaking Bad (2008).json" exists in the series folder
    When the metadata file is read
    Then SeriesMetadataFound is returned with the correct values

  Scenario: Metadata file does not exist
    Given no metadata file exists in the series folder
    When the metadata file is read
    Then SeriesMetadataNotFound is returned

  Scenario: Metadata file is malformed
    Given a "Breaking Bad (2008).json" that contains invalid JSON
    When the metadata file is read
    Then SeriesMetadataError is returned with reason "Metadata file malformed"

  Scenario: Series status is always refreshed from TMDB
    Given a metadata file where the series has IsAiring false
    When the metadata is refreshed
    Then the series status is fetched from TMDB regardless

  Scenario: Ended series is reactivated
    Given a metadata file where the series has IsAiring false
    And TMDB now reports the series as airing
    When the metadata is refreshed
    Then the series IsAiring is updated to true

  Scenario: Airing season completes
    Given a metadata file where Season 1 has IsAiring true and IsComplete false
    And TMDB now reports Season 1 as complete
    When the metadata is refreshed
    Then Season 1 IsAiring is set to false
    And Season 1 IsComplete is set to true
    And the episode list for Season 1 is not re-fetched

  Scenario: Airing season is refreshed from TMDB
    Given a metadata file where Season 1 has IsAiring true and EpisodeCount 10
    And TMDB now reports Season 1 has 12 episodes
    When the metadata is refreshed
    Then Season 1 EpisodeCount is updated to 12
    And the two new episodes have titles, runtimes, and air dates from TMDB

  Scenario: Complete season episode list is not re-fetched
    Given a metadata file where Season 1 has IsComplete true and EpisodeCount 10
    When the metadata is refreshed
    Then TMDB is not called to fetch the episode list for Season 1

  Scenario: Absolute episode number is calculated correctly across seasons
    Given a series with Season 1 (2 episodes) and Season 2 (2 episodes)
    When the metadata file is written
    Then Season 2 Episode 1 has AbsoluteEpisodeNumber 3
    And Season 2 Episode 2 has AbsoluteEpisodeNumber 4

  Scenario: Season 0 is excluded from absolute episode count
    Given a series with Season 0 (2 specials), Season 1 (2 episodes), and Season 2 (2 episodes)
    When the metadata file is written
    Then Season 0 episodes have AbsoluteEpisodeNumber null
    And Season 2 Episode 1 has AbsoluteEpisodeNumber 3

  Scenario: Absolute episode number is recalculated after refresh
    Given a metadata file where Season 1 has IsAiring true and EpisodeCount 2
    And Season 2 Episode 1 has AbsoluteEpisodeNumber 3
    And TMDB now reports Season 1 has 4 episodes
    When the metadata is refreshed
    Then Season 2 Episode 1 has AbsoluteEpisodeNumber 5

  Scenario: Episode within range is valid
    Given a metadata file where Season 1 has EpisodeCount 13
    When episode S01E07 is validated
    Then no error is returned

  Scenario: Episode out of range
    Given a metadata file where Season 1 has EpisodeCount 13
    When episode S01E22 is validated
    Then a domain error is returned with reason "Episode out of range"

  Scenario: Season not found
    Given a metadata file with no Season 5
    When episode S05E01 is validated
    Then a domain error is returned with reason "Season not found"
```

---

## Subtasks

- [ ] Define `SeriesMetadata`, `SeasonMetadata`, and `EpisodeMetadata` records in `Scoutarr.Core`
- [ ] Define `SeriesMetadataReadResult` discriminated union in `Scoutarr.Core`
- [ ] Define `ISeriesMetadataService` interface in `Scoutarr.Core`
- [ ] Extend `ITmdbClient` with season-level episode list fetching if not already present from UC-02
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `SeriesMetadataService`
- [ ] Hawkeye reviews
