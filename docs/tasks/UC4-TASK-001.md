# UC4-TASK-001 — Known series matching in /media/tv

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Rename mode", "TV episode move behaviour"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Information extracted from the filename"
**Dependencies:** UC2-TASK-001 (ITvSeriesTitleParser), UC3-TASK-002 (ISeriesMetadataService)

---

## Context

When a single TV episode arrives in `/input/tv`, before going to TMDB to identify the series, Scoutarr should check whether that series already exists in `/media/tv` — because if it does, the full TMDB identification flow can be skipped entirely.

This task implements that local lookup. It is the only new piece of logic UC-04 introduces before handing off to the existing UC-02 pipeline.

The title to match against is taken from the most reliable source available, in order:

1. The name of the parent folder in `/input/tv`, if the file is not at the root of that mount (e.g. `/input/tv/Breaking Bad 2008/episode.mkv` → use `Breaking Bad 2008`).
2. The filename itself, parsed by `ITvSeriesTitleParser` (e.g. `breaking.bad.s03e05.mkv` → `Breaking Bad`).

Matching is done against the folder names present in `/media/tv`. Each folder is expected to follow the Jellyfin convention `{Series Name} ({Year})`.

---

## Matching rules

**Exact match (year present in source):** if the extracted title and year together match a single folder name exactly (case-insensitive), that folder is the result. No ambiguity.

**Title-only match (no year in source):** compare the extracted title against the series name part of each folder name (stripping the year). If exactly one folder matches, that is the result. If more than one folder matches (e.g. `The Office (2001)` and `The Office (2005)`), the result is ambiguous.

**No match:** no folder in `/media/tv` matches the extracted title. This is not an error — it means the series is new and the caller should fall through to the UC-02 flow.

---

## Output types

```
KnownSeriesMatchResult  (discriminated union)
    | KnownSeriesFound
    | KnownSeriesAmbiguous
    | KnownSeriesNotFound

KnownSeriesFound
{
    SeriesName:    string        // e.g. "Breaking Bad"
    SeriesYear:    int           // e.g. 2008
    FolderPath:    string        // absolute path to the series folder in /media/tv
    Metadata:      SeriesMetadata
}

KnownSeriesAmbiguous
{
    Candidates:    IReadOnlyList<KnownSeriesCandidate>
}

KnownSeriesCandidate
{
    SeriesName:    string
    SeriesYear:    int
    FolderPath:    string
}

KnownSeriesNotFound  // no fields — signals caller to use UC-02 flow
```

If the metadata file is present but malformed, the service returns a domain error — it does not silently ignore it.

---

## Notes for Black Widow

- Mock `IFileSystem` and `ISeriesMetadataService`. No TMDB calls in this task.
- Test exact match with year: title + year in parent folder name matches one folder → `KnownSeriesFound`.
- Test exact match with year: title + year in filename matches one folder → `KnownSeriesFound`.
- Test title-only match: no year available, one folder matches → `KnownSeriesFound`.
- Test ambiguous match: no year available, two folders match (e.g. `The Office (2001)` and `The Office (2005)`) → `KnownSeriesAmbiguous` with both candidates.
- Test ambiguous match resolved by year: same two folders, but year is present in source and matches one → `KnownSeriesFound`.
- Test no match: no folder in `/media/tv` matches → `KnownSeriesNotFound`.
- Test malformed metadata file: domain error is returned.
- Test that parent folder name is preferred over filename when both are available.
- Test that matching is case-insensitive.

## Notes for Tony Stark

- Implement the new service in `Scoutarr.Core`. Name it as you see fit.
- Use `IFileSystem` to list folders in `/media/tv` — no direct filesystem calls.
- Use `ISeriesMetadataService` to read the metadata file from the matched folder.
- The title extracted from the parent folder name should be parsed with the same cleaning rules as `ITvSeriesTitleParser` (strip separators, year, release group) — reuse or extract a shared utility rather than duplicating logic.
- Register the new service in the DI containers of both `Scoutarr.Api` and `Scoutarr.Mcp`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Known series matching in /media/tv
  As the UC-04 orchestrator
  I want to check whether an incoming episode belongs to a series already present in /media/tv
  So that I can skip TMDB identification and use the existing metadata instead

  Scenario: Exact match — year present in parent folder name
    Given the file is at "/input/tv/Breaking Bad 2008/s03e05.mkv"
    And "/media/tv/Breaking Bad (2008)" exists with a valid metadata file
    When the matching service is called
    Then the result is KnownSeriesFound
    And the series name is "Breaking Bad" and the year is 2008

  Scenario: Exact match — year present in filename
    Given the file is at "/input/tv/breaking.bad.2008.s03e05.mkv"
    And "/media/tv/Breaking Bad (2008)" exists with a valid metadata file
    When the matching service is called
    Then the result is KnownSeriesFound
    And the series name is "Breaking Bad" and the year is 2008

  Scenario: Title-only match — no year available, one folder matches
    Given the file is at "/input/tv/breaking.bad.s03e05.mkv"
    And "/media/tv/Breaking Bad (2008)" is the only matching folder
    When the matching service is called
    Then the result is KnownSeriesFound

  Scenario: Ambiguous match — no year, multiple folders match
    Given the file is at "/input/tv/the.office.s03e05.mkv"
    And both "/media/tv/The Office (2001)" and "/media/tv/The Office (2005)" exist
    When the matching service is called
    Then the result is KnownSeriesAmbiguous
    And both candidates are returned

  Scenario: Ambiguous match resolved by year
    Given the file is at "/input/tv/The Office 2005/s03e05.mkv"
    And both "/media/tv/The Office (2001)" and "/media/tv/The Office (2005)" exist
    When the matching service is called
    Then the result is KnownSeriesFound
    And the series year is 2005

  Scenario: No match
    Given the file is at "/input/tv/severance.s02e01.mkv"
    And no folder in "/media/tv" matches "Severance"
    When the matching service is called
    Then the result is KnownSeriesNotFound

  Scenario: Malformed metadata file
    Given the file is at "/input/tv/breaking.bad.s03e05.mkv"
    And "/media/tv/Breaking Bad (2008)" exists but its metadata file is malformed
    When the matching service is called
    Then a domain error is returned with reason "Series metadata file is malformed"

  Scenario: Matching is case-insensitive
    Given the file is at "/input/tv/BREAKING.BAD.S03E05.mkv"
    And "/media/tv/Breaking Bad (2008)" exists with a valid metadata file
    When the matching service is called
    Then the result is KnownSeriesFound

  Scenario: Parent folder name is preferred over filename
    Given the file is at "/input/tv/Severance 2022/breaking.bad.s03e05.mkv"
    And "/media/tv/Severance (2022)" exists with a valid metadata file
    When the matching service is called
    Then the result is KnownSeriesFound with series name "Severance"
```

---

## Subtasks

- [ ] Define `KnownSeriesMatchResult`, `KnownSeriesFound`, `KnownSeriesAmbiguous`, `KnownSeriesNotFound`, `KnownSeriesCandidate` in `Scoutarr.Core`
- [ ] Define the new service interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements the service
- [ ] Hawkeye reviews
