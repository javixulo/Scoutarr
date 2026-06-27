# UC3-TASK-001 — Folder Scanner

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Input", "Recognised file extensions"

---

## Context

Given a folder path, discover all video files and their associated subtitle files, regardless of the internal folder structure (flat, organised by season, or mixed). The result feeds directly into the Folder Rename Orchestrator.

This task covers:

- Recursive traversal of the folder
- Classification of files as video, subtitle, or non-media
- Pairing each subtitle file with its corresponding video file based on filename stem
- Detecting the subtitle language code when present
- Exposing the raw folder name for use as a series identification hint

No TMDB calls. No renaming. No episode number extraction. Pure filesystem traversal and classification.

---

## Output types

```
ScannedFolder
{
    FolderName:   string                          // raw name of the root folder, not cleaned
    VideoFiles:   IReadOnlyList<ScannedVideoFile>
    IgnoredFiles: IReadOnlyList<string>           // paths of non-media files and unmatched subtitles, for logging
}

ScannedVideoFile
{
    Path:      string
    Subtitles: IReadOnlyList<ScannedSubtitle>
}

ScannedSubtitle
{
    Path:         string
    LanguageCode: string?   // null if not detected
}
```

On failure: a domain error when the folder does not exist or is not accessible, or when the folder contains no video files.

---

## Pairing rules

A subtitle file is paired with a video file when its stem matches the video file's stem, with or without a language suffix. Examples:

- `Breaking.Bad.S01E01.mkv` + `Breaking.Bad.S01E01.srt` → paired, no language code
- `Breaking.Bad.S01E01.mkv` + `Breaking.Bad.S01E01.en.srt` → paired, language `en`
- `Breaking.Bad.S01E01.mkv` + `Breaking.Bad.S01E02.srt` → not paired

Subtitles with no matching video file are included in `IgnoredFiles`.

---

## Notes for Black Widow

- All tests are pure — mock `IFileSystem`, no real filesystem access.
- Test flat folder, season subfolders, and mixed structures.
- Test all recognised video and subtitle extensions.
- Test subtitle pairing: exact stem match, with language code, no match.
- Test language code detection: 2-letter, 3-letter, absent, ambiguous (e.g. stem ending in a 2-letter word that is not a language code).
- Test non-media files are collected in `IgnoredFiles`.
- Test unmatched subtitle files are collected in `IgnoredFiles`.
- Test folder not found returns a domain error.
- Test folder with no video files returns a domain error.
- Test that `FolderName` is the raw folder name, not cleaned.

## Notes for Tony Stark

- Implement `IFolderScanner` and `FolderScanner` in `Scoutarr.Core`.
- Depends on `IFileSystem` — no direct filesystem calls.
- `FolderName` is the raw name of the root folder passed in. Do not clean or normalise it — that is the responsibility of the series title parser downstream.
- Language code detection reuses the same logic defined in file-handling.md: a suffix matching a 2- or 3-letter ISO code between the stem and the extension.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Folder scanner
  As the Folder Rename Orchestrator
  I want to discover all video and subtitle files in a folder
  So that each video file is paired with its subtitles before processing

  Scenario: Flat folder with video and paired subtitle
    Given a folder containing "Breaking.Bad.S01E01.mkv" and "Breaking.Bad.S01E01.srt"
    When the folder is scanned
    Then the result contains one video file
    And that video file has one subtitle with no language code

  Scenario: Subtitle with language code
    Given a folder containing "Breaking.Bad.S01E01.mkv" and "Breaking.Bad.S01E01.en.srt"
    When the folder is scanned
    Then the subtitle has language code "en"

  Scenario: Season subfolders
    Given a folder containing "Season 01/Breaking.Bad.S01E01.mkv" and "Season 02/Breaking.Bad.S02E01.mkv"
    When the folder is scanned
    Then the result contains two video files

  Scenario: Non-media file is ignored
    Given a folder containing "Breaking.Bad.S01E01.mkv" and "readme.txt"
    When the folder is scanned
    Then the result contains one video file
    And "readme.txt" is in the ignored files list

  Scenario: Subtitle with no matching video is ignored
    Given a folder containing "Breaking.Bad.S01E01.srt" with no matching video file
    When the folder is scanned
    Then the result contains no video files
    And "Breaking.Bad.S01E01.srt" is in the ignored files list

  Scenario: Raw folder name is preserved
    Given a folder named "breaking_bad"
    When the folder is scanned
    Then FolderName is "breaking_bad"

  Scenario: Folder does not exist
    Given a folder path that does not exist
    When the folder is scanned
    Then a domain error is returned with reason "Folder not found"

  Scenario: Folder contains no video files
    Given a folder containing only "readme.txt" and "cover.jpg"
    When the folder is scanned
    Then a domain error is returned with reason "No video files found"
```

---

## Subtasks

- [ ] Define `ScannedFolder`, `ScannedVideoFile`, and `ScannedSubtitle` records in `Scoutarr.Core`
- [ ] Define `IFolderScanner` interface in `Scoutarr.Core`
- [ ] Evaluate whether language code detection can be shared with subtitle handling in UC-02
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `FolderScanner`
- [ ] Hawkeye reviews
