# UC3-TASK-002 — Series Metadata File

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Series state", "Metadata file", "Validation on subsequent passes"

---

## Context

After a series folder is successfully identified, Scoutarr writes a JSON metadata file to the root series folder. On subsequent passes, that file is read instead of re-identifying the series, and its data is used to validate incoming episodes against known season and episode counts.

This task covers:

- Writing the metadata file after a successful series identification
- Reading and deserialising the metadata file on subsequent passes
- Refreshing season and episode data from TMDB for any season still marked as airing
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
    AbsoluteEpisodeNumber: int?    // null for Season 0 specials
    Title:                 string
    RuntimeMinutes:        int?    // null if not provided by TMDB
}

SeriesMetadataReadResult =
    | SeriesMetadataFound(SeriesMetadata metadata)
    | SeriesMetadataNotFound                        // expected on first pass
    | SeriesMetadataError(string reason)            // file exists but is malformed
```

---

## Refresh behaviour

On each pass where a metadata file already exists, Scoutarr refreshes data from TMDB for any season where `IsAiring` is true. This updates episode counts, episode titles, and runtimes for ongoing seasons. Seasons marked as complete are not refreshed.

`AbsoluteEpisodeNumber` is recalculated after each refresh, since episode counts in earlier airing seasons may have changed.

---

## TMDB data source

Episode names and runtimes are fetched from the `/3/tv/{series_id}/season/{season_number}` endpoint, which returns the full episode list for a season including `name` and `runtime` per episode. One call per season is required.

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
        { "episodeNumber": 1, "absoluteEpisodeNumber": null, "title": "Special 1", "runtimeMinutes": 45 },
        { "episodeNumber": 2, "absoluteEpisodeNumber": null, "title": "Special 2", "runtimeMinutes": 42 }
      ]
    },
    {
      "seasonNumber": 1,
      "episodeCount": 2,
      "isAiring": false,
      "isComplete": true,
      "episodes": [
        { "episodeNumber": 1, "absoluteEpisodeNumber": 1, "title": "Pilot", "runtimeMinutes": 58 },
        { "episodeNumber": 2, "absoluteEpisodeNumber": 2, "title": "Cat's in the Bag", "runtimeMinutes": 48 }
      ]
    },
    {
      "seasonNumber": 2,
      "episodeCount": 2,
      "isAiring": false,
      "isComplete": true,
      "episodes": [
        { "episodeNumber": 1, "absoluteEpisodeNumber": 3, "title": "Seven Thirty-Seven", "runtimeMinutes": 47 },
        { "episodeNumber": 2, "absoluteEpisodeNumber": 4, "title": "Grilled", "runtimeMinutes": 47 }
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
- Test refresh: seasons with `IsAiring: true` are refreshed from TMDB; complete seasons are not.
- Test `AbsoluteEpisodeNumber` calculation: correct across multiple seasons, excluding Season 0.
- Test `AbsoluteEpisodeNumber` is recalculated after refresh when episode counts change.
- Test validation: episode within range, episode out of range, season not found.

## Notes for Tony Stark

- Implement `ISeriesMetadataService` and `SeriesMetadataService` in `Scoutarr.Core`.
- Filename format: `{Series Title} ({Year}).json`, written to the root series folder.
- Depends on `IFileSystem` and `ITmdbClient`.
- Use `System.Text.Json` for serialisation — consistent with the rest of the project.
- `AbsoluteEpisodeNumber` is stored in the JSON file and only recalculated after a refresh, not on every read.

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
    And each episode has a title, runtime, and absolute episode number where applicable

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

  Scenario: Airing season is refreshed from TMDB
    Given a metadata file where Season 1 has IsAiring true and EpisodeCount 10
    And TMDB now reports Season 1 has 12 episodes
    When the metadata is refreshed
    Then Season 1 EpisodeCount is updated to 12
    And the two new episodes have titles and runtimes from TMDB

  Scenario: Complete season is not refreshed
    Given a metadata file where Season 1 has IsComplete true and EpisodeCount 10
    When the metadata is refreshed
    Then TMDB is not called for Season 1

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
