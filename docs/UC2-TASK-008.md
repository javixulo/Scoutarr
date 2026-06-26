# UC2-TASK-008 — TV episode file move service

**Requirements:** [file-handling.md](file-handling.md) — "Rename mode", "Folder structure", "Recognised file extensions"
**Dependencies:** UC2-TASK-007 (TvFormatterInput / formatted filename)

---

## Context

Given a source file path and a formatted episode filename, move the video file and all associated subtitle files to the correct destination within `MEDIA_ROOT_TV`, creating the series and season folder structure if needed. After the move, clean up any empty directories left behind in the source path.

This service runs in **Rename mode only**. Identify mode does not touch the filesystem.

No TMDB calls. No filename parsing. No formatting. Pure filesystem operations.

---

## Mount points

This task depends on fixed Docker volume mount points:

| Mount point | Purpose |
|---|---|
| `/input/tv` | Source TV files to be processed |
| `/media/tv` | Destination for renamed TV episodes (Jellyfin library root) |

These are mounted by the user at container startup. The application reads them as fixed paths — they are not configurable via environment variables. Scoutarr never reads from `/media/tv` as a source, and never writes to `/input/tv` as a destination.

The subtask to update `ARCHITECTURE.md` below covers adding these mount points to the volume structure documentation.

---

## Interface

```
ITvEpisodeMoveService

MoveAsync(TvEpisodeMoveRequest request, CancellationToken ct)
    → TvEpisodeMoveResult
```

---

## Input type

```
TvEpisodeMoveRequest
{
    SourcePath:        string   // full path to the source video file
    SeriesName:        string   // from TMDB — used to name the series folder
    SeriesYear:        int      // from TMDB — used to name the series folder
    Season:            int      // used to name the season folder
    FormattedFilename: string   // output of TvFilenameFormatter, no extension
}
```

---

## Output types

```
TvEpisodeMoveResult  (discriminated union)
    | TvEpisodeMoveSuccess
    | TvEpisodeMoveError

TvEpisodeMoveSuccess
{
    DestinationPath:       string                      // final path of the video file
    MovedSubtitles:        IReadOnlyList<SubtitleMove> // one entry per subtitle moved
    CleanedDirectories:    IReadOnlyList<string>       // directories removed after cleanup
}

SubtitleMove
{
    SourcePath:       string
    DestinationPath:  string
}

TvEpisodeMoveError  (reuses shared domain error model)
{
    Reason:   string
    Detail:   string?
}
```

---

## Operation steps

```
1. Resolve destination:
   - Series folder:  {MEDIA_ROOT_TV}/{SeriesName} ({SeriesYear})/
   - Season folder:  {MEDIA_ROOT_TV}/{SeriesName} ({SeriesYear})/Season {Season:00}/
   - Video target:   {season folder}/{FormattedFilename}{original extension}

2. Create series and season folders if they do not exist.

3. Check for collision: if a file with the target name already exists at the destination → return TvEpisodeMoveError with reason "File already exists at destination".

4. Discover associated subtitle files: files in the same directory as the source video that share the same filename stem, with a recognised subtitle extension (.srt, .ass, .ssa, .sub, .idx). Language-coded subtitles are also matched (e.g. episode.en.srt, episode.es.srt).

5. Move the video file to the destination.

6. Move and rename each subtitle file to the destination:
   - Subtitle with no language code: {FormattedFilename}.{ext}
   - Subtitle with language code:    {FormattedFilename}.{lang}.{ext}

7. If any move operation fails mid-way (e.g. copy+delete on cross-filesystem move fails after partial copy):
   - Attempt to delete any partially written destination file.
   - Attempt to restore any already-moved files back to the source directory.
   - Log the rollback attempt and its outcome (success or partial failure) to the Scoutarr log.
   - Return `TvEpisodeMoveError` with reason "Move failed — partial rollback attempted" and detail describing which files were restored and which were not.

8. Clean up empty directories: starting from the source directory, walk upward and delete any directory that is now empty. Stop at `/media/tv` — never delete it or any ancestor. Only runs if the move completed successfully.

9. Return TvEpisodeMoveSuccess.
```

---

## Subtitle language detection

A subtitle file is considered language-coded if its name matches the pattern `{stem}.{lang}.{ext}` where `{lang}` is a 2- or 3-letter ISO language code (e.g. `en`, `es`, `fra`). If the pattern does not match, the subtitle is treated as language-unknown and renamed without a language suffix.

---

## Notes for Black Widow

- Mock the filesystem for all tests — never touch real files.
- Test the happy path: video moved, subs moved and renamed, source directory cleaned up.
- Test with no subtitle files: only the video is moved.
- Test with multiple subtitle files: language-coded and language-unknown in the same directory.
- Test collision: file already exists at destination → error, nothing is moved.
- Test destination folder creation: series folder exists but season folder does not.
- Test destination folder creation: neither series nor season folder exists.
- Test empty directory cleanup: source directory is empty after move → deleted. Parent directory is also empty → deleted. Walk stops at MEDIA_ROOT_TV.
- Test cleanup does not delete MEDIA_ROOT_TV itself.
- Test cleanup does not delete a directory that still contains other files.
- Test rollback: copy+delete fails mid-way after video is moved but before subtitle is moved → video is restored to source, error returned, rollback outcome is logged.
- Test rollback partial failure: rollback itself fails (e.g. source directory no longer accessible) → error returned with detail listing unrestored files.
- Test insufficient write permissions → TvEpisodeMoveError.
- Test source file not found → TvEpisodeMoveError.

## Notes for Tony Stark

- Implement `ITvEpisodeMoveService` and `TvEpisodeMoveService` in `Scoutarr.Core`.
- Inject `IFileSystem` (abstraction over System.IO) via constructor to allow filesystem mocking in tests.
- Season folder naming: `Season {Season:00}` — zero-padded to 2 digits (e.g. `Season 01`, `Season 00` for specials).
- Series folder naming: `{SeriesName} ({SeriesYear})` — characters invalid in directory names must be sanitised (same rules as filename sanitisation).
- The cleanup walk must be bounded: stop when reaching `MEDIA_ROOT_TV` or when the current directory is not empty.
- Move is atomic where the OS supports it (same filesystem). Cross-filesystem moves fall back to copy + delete.
- On cross-filesystem move failure: attempt rollback by deleting partial destination files and restoring moved files. Log each rollback step via the Scoutarr logger regardless of outcome.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: TV episode file move service
  As the Core rename orchestration service
  I want to move a TV episode and its subtitles to the correct destination
  So that the file is correctly placed within the Jellyfin folder structure

  # ─────────────────────────────────────────
  # HAPPY PATH
  # ─────────────────────────────────────────

  Scenario: Video file moved to correct destination, no subtitles
    Given source file "/downloads/breaking.bad.s01e02.mkv"
    And MEDIA_ROOT_TV is "/media/tv"
    And series name "Breaking Bad", year 2008, season 1
    And formatted filename "Breaking Bad - S01E02 - Cat's in the Bag"
    When MoveAsync is called
    Then the file is moved to "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the source directory "/downloads" is deleted if empty

  Scenario: Video and language-coded subtitle moved and renamed correctly
    Given source file "/downloads/breaking.bad.s01e02.mkv"
    And a subtitle file "/downloads/breaking.bad.s01e02.en.srt"
    And formatted filename "Breaking Bad - S01E02 - Cat's in the Bag"
    When MoveAsync is called
    Then the subtitle is moved to "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.en.srt"

  Scenario: Video and language-unknown subtitle moved and renamed correctly
    Given source file "/downloads/breaking.bad.s01e02.mkv"
    And a subtitle file "/downloads/breaking.bad.s01e02.srt"
    And formatted filename "Breaking Bad - S01E02 - Cat's in the Bag"
    When MoveAsync is called
    Then the subtitle is moved to "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.srt"

  Scenario: Multiple subtitles with different languages moved correctly
    Given source file "/downloads/breaking.bad.s01e02.mkv"
    And subtitle files "/downloads/breaking.bad.s01e02.en.srt" and "/downloads/breaking.bad.s01e02.es.srt"
    When MoveAsync is called
    Then both subtitles are moved and renamed with their respective language codes

  Scenario: Season 0 special placed in Season 00 folder
    Given source file "/downloads/breaking.bad.s00e01.mkv"
    And series name "Breaking Bad", year 2008, season 0
    And formatted filename "Breaking Bad - S00E01 - Pilot (Unaired)"
    When MoveAsync is called
    Then the file is moved to "/media/tv/Breaking Bad (2008)/Season 00/Breaking Bad - S00E01 - Pilot (Unaired).mkv"

  # ─────────────────────────────────────────
  # FOLDER CREATION
  # ─────────────────────────────────────────

  Scenario: Series and season folders are created when neither exists
    Given MEDIA_ROOT_TV "/media/tv" contains no folder for "Breaking Bad (2008)"
    When MoveAsync is called
    Then "/media/tv/Breaking Bad (2008)/Season 01/" is created
    And the file is moved into it

  Scenario: Season folder is created when series folder exists but season folder does not
    Given "/media/tv/Breaking Bad (2008)/" already exists
    And "/media/tv/Breaking Bad (2008)/Season 01/" does not exist
    When MoveAsync is called
    Then "/media/tv/Breaking Bad (2008)/Season 01/" is created
    And the file is moved into it

  Scenario: No folders are created when destination already exists
    Given "/media/tv/Breaking Bad (2008)/Season 01/" already exists
    When MoveAsync is called
    Then no new directories are created
    And the file is moved into the existing folder

  # ─────────────────────────────────────────
  # EMPTY DIRECTORY CLEANUP
  # ─────────────────────────────────────────

  Scenario: Source directory is empty after move — directory is deleted
    Given source file "/downloads/new/breaking.bad.s01e02.mkv"
    And no other files exist in "/downloads/new/"
    When MoveAsync is called
    Then "/downloads/new/" is deleted after the move

  Scenario: Cleanup walks upward through multiple empty directories
    Given source file "/downloads/new/sub/breaking.bad.s01e02.mkv"
    And no other files exist in "/downloads/new/sub/" or "/downloads/new/"
    When MoveAsync is called
    Then both "/downloads/new/sub/" and "/downloads/new/" are deleted

  Scenario: Cleanup stops when parent directory still contains other files
    Given source file "/downloads/new/breaking.bad.s01e02.mkv"
    And "/downloads/new/" contains another file after the move
    When MoveAsync is called
    Then "/downloads/new/" is not deleted

  Scenario: Cleanup never deletes MEDIA_ROOT_TV
    Given source file is directly inside MEDIA_ROOT_TV
    And the source directory is empty after the move
    When MoveAsync is called
    Then MEDIA_ROOT_TV is not deleted

  # ─────────────────────────────────────────
  # ROLLBACK — cross-filesystem failure
  # ─────────────────────────────────────────

  Scenario: Copy+delete fails after video copied but before subtitle moved — video restored
    Given source file "/input/tv/breaking.bad.s01e02.mkv" and subtitle "/input/tv/breaking.bad.s01e02.en.srt"
    And the move is cross-filesystem
    And the copy of the subtitle fails mid-way
    When MoveAsync is called
    Then the result is TvEpisodeMoveError with reason "Move failed — partial rollback attempted"
    And the video file is restored to "/input/tv/breaking.bad.s01e02.mkv"
    And the partial destination file is deleted
    And the rollback outcome is written to the Scoutarr log

  Scenario: Rollback itself fails — error returned with unrestored files listed
    Given source file "/input/tv/breaking.bad.s01e02.mkv"
    And the copy+delete fails mid-way
    And the source directory is no longer accessible during rollback
    When MoveAsync is called
    Then the result is TvEpisodeMoveError with reason "Move failed — partial rollback attempted"
    And the error detail lists the files that could not be restored

  # ─────────────────────────────────────────
  # COLLISION
  # ─────────────────────────────────────────

  Scenario: File already exists at destination — error returned, nothing moved
    Given a file already exists at "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    When MoveAsync is called
    Then the result is TvEpisodeMoveError with reason "File already exists at destination"
    And the source file is not moved

  # ─────────────────────────────────────────
  # ERROR CASES
  # ─────────────────────────────────────────

  Scenario: Source file not found — error returned
    Given source path "/downloads/nonexistent.mkv" does not exist
    When MoveAsync is called
    Then the result is TvEpisodeMoveError with reason "Source file not found"

  Scenario: Insufficient write permissions on destination — error returned
    Given the destination directory is not writable
    When MoveAsync is called
    Then the result is TvEpisodeMoveError with reason "Insufficient permissions"
```

---

## Subtasks

- [ ] Add `/media/movies` and `/media/tv` mount points to the volume structure in `ARCHITECTURE.md`
- [ ] Define `IFileSystem` abstraction in `Scoutarr.Core` if not already present from UC1
- [ ] Define `TvEpisodeMoveRequest`, `TvEpisodeMoveSuccess`, `SubtitleMove`, `TvEpisodeMoveError` records in `Scoutarr.Core`
- [ ] Define `ITvEpisodeMoveService` interface in `Scoutarr.Core`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvEpisodeMoveService`
- [ ] Hawkeye reviews
