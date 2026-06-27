# UC3-TASK-003 — Folder Rename Orchestrator

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Rename mode", "Identify mode", "TV episode move behaviour", "Series state"

---

## Context

The orchestrator ties together all the pieces built in UC-02 and UC-03 to process a series folder end to end. It identifies the series once, processes each episode independently using existing services, and writes or updates the metadata file on every pass regardless of mode.

This task covers:

- Reading the metadata file to determine whether this is a first pass or a subsequent pass
- Identifying the series once for the whole folder (first pass only)
- Processing each episode independently using UC-02 services
- Accumulating successes and failures into an aggregated result
- Writing or updating the metadata file after processing in both modes

No new filesystem logic. No new TMDB calls. No new parsing or formatting. Pure orchestration of existing services.

---

## Two modes

**`identify_folder`** — scans the folder, identifies the series, refreshes status, parses episode numbers, validates against metadata, and returns suggested filenames. Writes or updates the metadata file. Does not move any files.

**`rename_folder`** — does everything `identify_folder` does, and additionally moves each file to its destination.

The only difference between the two modes is whether files are moved or not. The metadata file is always written or updated.

---

## Flow

```
identify_folder / rename_folder
│
1. Read metadata file (ISeriesMetadataService)
   ├── SeriesMetadataNotFound → identify series (ITvShowIdentificationService)
   │       ├── DisambiguationNeeded → abort, return disambiguation result
   │       ├── IdentificationError  → abort, return error
   │       └── IdentificationSuccess → continue with confirmed tmdbId
   ├── SeriesMetadataFound → skip identification, use known tmdbId
   │       └── Refresh series and season status from TMDB (always)
   └── SeriesMetadataError → abort, return error

2. Scan folder (IFolderScanner)
   └── Error → abort, return error

3. For each video file in scan result:
   ├── Parse episode number (ITvEpisodeNumberParser)
   ├── Validate episode against metadata (ISeriesMetadataService)
   ├── Look up episode (ITvEpisodeLookupService)
   ├── Format filename (ITvFilenameFormatter)
   ├── [rename_folder only] Move file and subtitles (ITvEpisodeMoveService)
   └── on any failure: accumulate error, continue with next file

4. Write or update metadata file (ISeriesMetadataService)
   — always, in both modes, as long as series was identified successfully

5. Return aggregated result
```

---

## Output types

```
FolderIdentifyResult
{
    SeriesTitle:  string
    TmdbId:       int
    Episodes:     IReadOnlyList<EpisodeIdentifyResult>
    Failures:     IReadOnlyList<EpisodeFolderFailure>
}

EpisodeIdentifyResult
{
    OriginalFilename:   string
    SuggestedFilename:  string
}

FolderRenameResult
{
    SeriesTitle:  string
    TmdbId:       int
    Successes:    IReadOnlyList<EpisodeRenameSuccess>
    Failures:     IReadOnlyList<EpisodeFolderFailure>
}

EpisodeRenameSuccess
{
    OriginalFilename:  string
    NewFilename:       string
    Destination:       string
}

EpisodeFolderFailure
{
    OriginalFilename:  string
    Reason:            string
    Detail:            string
}
```

On abort: the existing domain error model is reused directly — disambiguation needed, identification error, folder not found, metadata malformed.

---

## Notes for Black Widow

- Mock all dependencies — `IFolderScanner`, `ITvShowIdentificationService`, `ITvEpisodeNumberParser`, `ISeriesMetadataService`, `ITvEpisodeLookupService`, `ITvFilenameFormatter`, `ITvEpisodeMoveService`.
- Test `identify_folder` first pass: metadata not found → identification → parse and format → write metadata → return suggested filenames. No files moved.
- Test `identify_folder` subsequent pass: metadata found → skip identification → refresh status → parse and format → update metadata → return suggested filenames. No files moved.
- Test `rename_folder` first pass: metadata not found → identification → process episodes → write metadata.
- Test `rename_folder` subsequent pass: metadata found → skip identification → refresh status → process episodes → update metadata.
- Test disambiguation abort: no episodes processed, no metadata written, disambiguation result returned.
- Test identification error abort: same pattern.
- Test metadata malformed abort: same pattern.
- Test episode failure accumulation: one episode fails, others succeed, result contains both.
- Test all episodes fail: successes list is empty, metadata file is still written.
- Test `identify_folder` never calls `ITvEpisodeMoveService`.
- Test metadata file is written in both `identify_folder` and `rename_folder`.

## Notes for Tony Stark

- Implement `IFolderRenameOrchestrator` and `FolderRenameOrchestrator` in `Scoutarr.Core`.
- Episode processing order is not significant — sequential is fine for MVP.
- The metadata file is written even if some episodes fail, as long as the series was identified successfully.
- `identify_folder` and `rename_folder` can share a private method for the common steps (steps 1–4).

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Folder rename orchestrator
  As the REST API and MCP server
  I want to process a series folder end to end
  So that all episodes are identified or renamed and the metadata file is kept up to date

  Scenario: identify_folder — first pass, all episodes identified
    Given a folder with no metadata file
    And the series is identified successfully
    And all episodes parse without errors
    When identify_folder is called
    Then suggested filenames are returned for all episodes
    And the metadata file is written
    And no files are moved

  Scenario: identify_folder — subsequent pass
    Given a folder with a valid metadata file
    When identify_folder is called
    Then series identification is skipped
    And series and season status is refreshed from TMDB
    And suggested filenames are returned for all episodes
    And the metadata file is updated
    And no files are moved

  Scenario: rename_folder — first pass, all episodes succeed
    Given a folder with no metadata file
    And the series is identified successfully
    And all episodes parse and move without errors
    When rename_folder is called
    Then all episodes appear in the successes list
    And the metadata file is written

  Scenario: rename_folder — subsequent pass
    Given a folder with a valid metadata file
    When rename_folder is called
    Then series identification is skipped
    And series and season status is refreshed from TMDB
    And episodes are processed using the known tmdbId
    And the metadata file is updated

  Scenario: Disambiguation needed — abort
    Given a folder with no metadata file
    And series identification returns disambiguation candidates
    When rename_folder is called
    Then no episodes are processed
    And no metadata file is written
    And the disambiguation result is returned

  Scenario: Metadata file malformed — abort
    Given a folder with a malformed metadata file
    When rename_folder is called
    Then no episodes are processed
    And a domain error is returned with reason "Metadata file malformed"

  Scenario: One episode fails, others succeed
    Given a folder with 3 video files
    And one episode number cannot be parsed
    When rename_folder is called
    Then 2 episodes appear in the successes list
    And 1 episode appears in the failures list with the parse error reason
    And the metadata file is written

  Scenario: All episodes fail
    Given a folder with 3 video files
    And all episode numbers cannot be parsed
    When rename_folder is called
    Then the successes list is empty
    And 3 episodes appear in the failures list
    And the metadata file is still written

  Scenario: identify_folder never moves files
    Given a folder with 3 video files
    And the series is identified successfully
    When identify_folder is called
    Then ITvEpisodeMoveService is never called
```

---

## Subtasks

- [ ] Define `FolderIdentifyResult`, `FolderRenameResult`, `EpisodeIdentifyResult`, `EpisodeRenameSuccess`, and `EpisodeFolderFailure` records in `Scoutarr.Core`
- [ ] Define `IFolderRenameOrchestrator` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `FolderRenameOrchestrator`
- [ ] Hawkeye reviews
