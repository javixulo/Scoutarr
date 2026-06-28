# UC4-TASK-002 — Subsequent pass orchestration for single TV episode

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Rename mode", "TV episode move behaviour", "Series state", "Validation on subsequent passes"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Confidence and automatic acceptance", "Disambiguation flow"
**Dependencies:** UC4-TASK-001 (known series matching), UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService), UC3-TASK-002 (ISeriesMetadataService)

---

## Context

This task extends the existing `rename_episode` orchestration from UC-02 with a new branch at the start of the flow: before going to TMDB, check whether the series already exists in `/media/tv` using the matching service from UC4-TASK-001.

Everything that happens after the series is confirmed — episode number parsing, episode title lookup, filename formatting, file move, subtitle move, empty directory cleanup — is unchanged and reuses the UC-02 services directly. REST and MCP interfaces require no changes: disambiguation, when needed, follows exactly the same flow already implemented in UC-02.

---

## Flow

```
rename_episode called
    │
    ▼
UC4-TASK-001: match series in /media/tv
    │
    ├── KnownSeriesNotFound → delegate to existing UC-02 flow (no changes)
    │
    ├── KnownSeriesAmbiguous → go to TMDB, disambiguate (same flow as UC-02)
    │                          cross tmdbId with KnownSeriesAmbiguous.Candidates
    │                          if match found → continue as KnownSeriesFound
    │                          if no match found → delegate to existing UC-02 flow
    │
    └── KnownSeriesFound
            │
            ▼
        Read metadata file (already loaded by UC4-TASK-001)
            │
            ▼
        Parse episode number from filename (ITvEpisodeNumberParser — reused)
            │
            ▼
        Is the episode's season marked as airing in metadata?
            │
            ├── No → validate episode against metadata as-is
            │
            └── Yes → refresh that season from TMDB via ISeriesMetadataService
                       update and persist metadata file
                       validate episode against refreshed metadata
            │
            ▼
        Episode valid?
            │
            ├── No → domain error: episode out of range or season not found
            │
            └── Yes → get episode title from metadata (no TMDB call needed)
                       format output filename (ITvFilenameFormatter — reused)
                       move file and subtitles (ITvEpisodeMoveService — reused)
                       return success
```

---

## What is new in this task

- The decision branch at the start of `rename_episode` that calls the matching service and routes accordingly.
- The refresh-or-skip logic based on season airing status.
- Getting the episode title from the local metadata instead of calling `ITvEpisodeLookupService` — when the series is known and the season is closed, no TMDB call is needed at all.

Everything else is a direct reuse of existing UC-02 and UC-03 services.

---

## Episode title resolution

When the series is known (`KnownSeriesFound`):
- If the season is **not airing**: the episode title is read directly from `SeriesMetadata` — no TMDB call.
- If the season **is airing**: after refreshing the season via `ISeriesMetadataService`, the episode title is again read from the updated `SeriesMetadata` — still no separate `ITvEpisodeLookupService` call.

`ITvEpisodeLookupService` is only used in the fallback path (UC-02 flow).

---

## Notes for Black Widow

- Mock `IKnownSeriesLocator` (or whatever Tony Stark names the UC4-TASK-001 service), `ISeriesMetadataService`, `ITvEpisodeNumberParser`, `ITvFilenameFormatter`, `ITvEpisodeMoveService`, and `ITmdbClient`.
- Test `KnownSeriesNotFound` → UC-02 flow is triggered, no metadata is read.
- Test `KnownSeriesAmbiguous` → TMDB disambiguation is triggered; if tmdbId matches a candidate, continue as known series; if not, UC-02 flow.
- Test `KnownSeriesFound`, season not airing → no TMDB refresh, episode title from metadata, move succeeds.
- Test `KnownSeriesFound`, season airing → `ISeriesMetadataService` refresh is called, metadata is updated, episode title from refreshed metadata, move succeeds.
- Test `KnownSeriesFound`, episode out of range (e.g. S03E99 when season 3 has 13 episodes) → domain error, no files moved.
- Test `KnownSeriesFound`, season not found in metadata (e.g. S05E01 when metadata only has seasons 1–4) → domain error, no files moved.
- Test `KnownSeriesFound`, metadata file malformed → domain error propagated from UC4-TASK-001, no files moved.
- Test that `ITvEpisodeLookupService` is never called when the series is known.
- E2E: full happy path — known series, closed season, episode valid → file moved to correct destination.
- E2E: known series, airing season → TMDB refresh called, file moved.
- E2E: known series, episode out of range → error returned, no files on disk are modified.

## Notes for Tony Stark

- Extend the existing `rename_episode` orchestrator in `Scoutarr.Core` — do not create a separate orchestrator.
- The new branch must be inserted before any TMDB call in the current flow.
- When `KnownSeriesAmbiguous` resolves via TMDB and the `tmdbId` does not match any candidate folder, fall through to the standard UC-02 flow — treat it as a new series.
- When reading the episode title from metadata, use `SeriesMetadata.Seasons[seasonNumber].Episodes[episodeNumber].Title`. If the episode entry has no title (null or empty), fall back to `ITvEpisodeLookupService` for that episode only.
- No changes to `Scoutarr.Api`, `Scoutarr.Mcp`, or any interface layer.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Subsequent pass orchestration for single TV episode
  As the rename_episode orchestrator
  I want to detect when a series already exists in /media/tv
  So that I can skip TMDB identification and use local metadata instead

  Scenario: Known series, closed season — no TMDB calls
    Given the file "breaking.bad.s03e05.mkv"
    And "Breaking Bad (2008)" exists in /media/tv with a valid metadata file
    And season 3 is marked as not airing in the metadata
    And S03E05 exists in the metadata with title "Dead Freight"
    When rename_episode is called
    Then no TMDB call is made
    And the file is moved to "/media/tv/Breaking Bad (2008)/Season 03/"
    And the output filename is "Breaking Bad - S03E05 - Dead Freight.mkv"

  Scenario: Known series, airing season — TMDB refresh before processing
    Given the file "breaking.bad.s03e05.mkv"
    And "Breaking Bad (2008)" exists in /media/tv with a valid metadata file
    And season 3 is marked as airing in the metadata
    When rename_episode is called
    Then ISeriesMetadataService.RefreshAsync is called for season 3
    And the metadata file is updated
    And the file is moved successfully

  Scenario: Known series, episode out of range
    Given the file "breaking.bad.s03e99.mkv"
    And "Breaking Bad (2008)" exists in /media/tv with a valid metadata file
    And season 3 has 13 episodes in the metadata
    When rename_episode is called
    Then a domain error is returned with reason "Episode out of range"
    And no files are moved

  Scenario: Known series, season not in metadata
    Given the file "breaking.bad.s05e01.mkv"
    And "Breaking Bad (2008)" exists in /media/tv with a valid metadata file
    And the metadata only contains seasons 1 to 4
    When rename_episode is called
    Then a domain error is returned with reason "Season not found in metadata"
    And no files are moved

  Scenario: Series not known — falls through to UC-02 flow
    Given the file "severance.s02e01.mkv"
    And no folder in /media/tv matches "Severance"
    When rename_episode is called
    Then the UC-02 identification flow is triggered
    And TMDB is queried for the series

  Scenario: Ambiguous match resolved via TMDB — tmdbId matches a candidate
    Given the file "the.office.s03e05.mkv"
    And both "The Office (2001)" and "The Office (2005)" exist in /media/tv
    And TMDB disambiguation resolves to tvId 2316 (The Office US, 2005)
    And tvId 2316 matches "The Office (2005)" in /media/tv
    When rename_episode is called
    Then the series is treated as known
    And the file is moved to "/media/tv/The Office (2005)/Season 03/"

  Scenario: Ambiguous match resolved via TMDB — tmdbId matches no candidate
    Given the file "the.office.s03e05.mkv"
    And both "The Office (2001)" and "The Office (2005)" exist in /media/tv
    And TMDB disambiguation resolves to a tvId that matches neither folder
    When rename_episode is called
    Then the UC-02 flow is used and the series is treated as new

  Scenario: Malformed metadata file
    Given the file "breaking.bad.s03e05.mkv"
    And "Breaking Bad (2008)" exists in /media/tv but its metadata file is malformed
    When rename_episode is called
    Then a domain error is returned with reason "Series metadata file is malformed"
    And no files are moved
```

---

## Subtasks

- [ ] Extend the `rename_episode` orchestrator in `Scoutarr.Core` with the known series branch
- [ ] Implement refresh-or-skip logic based on season airing status
- [ ] Implement episode title resolution from local metadata with fallback to `ITvEpisodeLookupService`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Black Widow writes E2E tests in red
- [ ] Tony Stark implements the extension
- [ ] Hawkeye reviews
