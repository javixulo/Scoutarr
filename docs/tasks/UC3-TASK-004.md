# UC3-TASK-004 — Folder Merge Logic

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "TV episode move behaviour"

---

## Context

When `rename_folder` moves episodes to their destination in `/media/tv/{Series Name} ({Year})/Season XX/`, the destination folder may or may not exist, and the destination file may or may not already be there. This task defines the exact behaviour for every combination of those cases.

The rule is simple but must be tested exhaustively because the consequences of getting it wrong are destructive: overwriting or losing media files is unacceptable.

This task covers:

- Creating destination folders when they do not exist
- Reusing destination folders when they do exist
- Skipping files that already exist at destination
- Reporting skipped files in the failures list with a clear reason
- Handling subtitle files consistently with their parent video file
- Handling permission errors at destination

No series identification. No episode parsing. No metadata. Pure filesystem move logic.

---

## Behaviour rules

| Destination folder | Destination file | Action |
|---|---|---|
| Does not exist | — | Create folder, move file |
| Exists | Does not exist | Move file |
| Exists | Exists | Skip, report as failure |
| Exists as file, not directory | — | Skip, report as failure |
| Insufficient permissions | — | Skip, report as failure |

These rules apply equally to video files and their associated subtitle files. If a video file is skipped, its subtitles are also skipped and each is reported independently. If a video file is moved successfully but a subtitle already exists at destination, the subtitle is skipped and reported independently.

---

## Notes for Black Widow

- All tests mock `IFileSystem` — no real filesystem access.
- Test destination folder does not exist: folder is created, file is moved.
- Test destination folder exists: folder is not recreated, file is moved.
- Test destination file exists: file is skipped, reported as failure with reason "File already exists at destination" and full destination path as detail.
- Test video skipped: all its subtitles are also skipped, each reported as a separate failure.
- Test video moved, one subtitle already exists: video moves, existing subtitle skipped and reported, other subtitles move.
- Test video moved, all subtitles already exist: video moves, all subtitles skipped and reported.
- Test multiple files in same folder: each is evaluated independently.
- Test nested season folder does not exist: all intermediate folders are created correctly (e.g. `/media/tv/Breaking Bad (2008)/Season 01/`).
- Test path with special characters in series or episode title: handled correctly.
- Test destination folder path is occupied by a file, not a directory: reported as failure with reason "Destination path is not a directory".
- Test insufficient permissions at destination: reported as failure with reason "Insufficient permissions at destination".

## Notes for Tony Stark

- This logic lives in `ITvEpisodeMoveService` / `TvEpisodeMoveService` in `Scoutarr.Core`, already introduced in UC-02.
- Extend the existing service rather than creating a new one — the single-file move behaviour from UC-02 already handles some of these cases.
- Never overwrite. Check existence before every move.
- Report each skipped file as an `EpisodeFolderFailure` with the appropriate reason and the full destination path as detail.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Folder merge logic
  As the Folder Rename Orchestrator
  I want to move episode files safely to their destination
  So that existing files are never overwritten

  Scenario: Destination folder does not exist
    Given the destination folder "/media/tv/Breaking Bad (2008)/Season 01/" does not exist
    When the episode file is moved
    Then the folder is created
    And the file is moved successfully

  Scenario: Destination folder already exists
    Given the destination folder "/media/tv/Breaking Bad (2008)/Season 01/" already exists
    When the episode file is moved
    Then the folder is not recreated
    And the file is moved successfully

  Scenario: Destination file already exists — video skipped
    Given the destination file "Breaking.Bad.S01E01.mkv" already exists at destination
    When the episode file is moved
    Then the file is not moved
    And the failure is reported with reason "File already exists at destination"
    And the full destination path is included in the failure detail

  Scenario: Video skipped — all subtitles skipped
    Given the destination file "Breaking.Bad.S01E01.mkv" already exists at destination
    And the episode has subtitles "Breaking.Bad.S01E01.en.srt" and "Breaking.Bad.S01E01.es.srt"
    When the episode file is moved
    Then the video file is not moved
    And both subtitle files are not moved
    And each subtitle is reported as a separate failure

  Scenario: Video moved — one subtitle already exists
    Given the destination file "Breaking.Bad.S01E01.mkv" does not exist at destination
    And "Breaking.Bad.S01E01.en.srt" already exists at destination
    And "Breaking.Bad.S01E01.es.srt" does not exist at destination
    When the episode file is moved
    Then the video file is moved successfully
    And "Breaking.Bad.S01E01.en.srt" is not moved and reported as failure
    And "Breaking.Bad.S01E01.es.srt" is moved successfully

  Scenario: Video moved — all subtitles already exist
    Given the destination file "Breaking.Bad.S01E01.mkv" does not exist at destination
    And all subtitle files already exist at destination
    When the episode file is moved
    Then the video file is moved successfully
    And all subtitles are skipped and reported as failures

  Scenario: Multiple files processed independently
    Given a folder with 3 episode files
    And the destination of the second episode already exists
    When all episode files are moved
    Then the first and third episodes are moved successfully
    And the second episode is skipped and reported as failure

  Scenario: Nested season folder does not exist
    Given the path "/media/tv/Breaking Bad (2008)/Season 01/" does not exist
    When the episode file is moved
    Then all intermediate folders are created
    And the file is moved successfully

  Scenario: Destination folder path is occupied by a file
    Given "/media/tv/Breaking Bad (2008)/Season 01" exists as a file, not a directory
    When the episode file is moved
    Then the file is not moved
    And the failure is reported with reason "Destination path is not a directory"

  Scenario: Insufficient permissions at destination
    Given the process does not have write permissions at "/media/tv/Breaking Bad (2008)/Season 01/"
    When the episode file is moved
    Then the file is not moved
    And the failure is reported with reason "Insufficient permissions at destination"
```

---

## Subtasks

- [ ] Extend `TvEpisodeMoveService` in `Scoutarr.Core` with folder merge behaviour
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements the changes in `TvEpisodeMoveService`
- [ ] Hawkeye reviews
