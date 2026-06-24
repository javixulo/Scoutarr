# File Handling

> Part of [Scoutarr Requirements](../REQUIREMENTS.md).

---

## Modes of operation

Scoutarr operates in two distinct modes, both exposed via REST API and MCP:

### Identify mode

Returns suggested filenames without touching the filesystem. Useful when the user does not have a volume mounted, wants to preview changes before applying them, or needs the suggested names to rename manually.

- Does not write, rename, or modify any file.
- Does not write the series metadata file to the folder.
- For a single file, returns the original filename and the suggested filename.
- For a folder (series), returns the suggested root folder name and a dictionary mapping each original filename to its suggested filename.
- Non-media files are silently omitted from the response — since nothing is modified, the user does not need to be informed about them.

### Rename mode

Applies the rename operation to disk. Internally calls Identify first, then applies the changes.

- Renames the video file and all auxiliary files in place (no moving).
- Writes the series metadata file to the folder after a successful series identification.
- Requires the target path to be accessible inside the container (mounted as a volume).

The Rename mode depends on Identify — if identification fails, no renaming occurs.

---

## Input

- Input: single file or folder.
- A folder is always treated as a TV show.
- A single file requires the caller to specify whether it is a movie or a TV show episode.
- Video file and all subtitle files are renamed together.
- Subtitle files are renamed to match the video filename exactly, with the language code appended before the extension when the language is known (e.g. `Series Name - S01E01 - Episode Title.en.srt`). If the language is unknown, the subtitle is renamed to match the video filename without a language suffix (e.g. `Series Name - S01E01 - Episode Title.srt`). This ensures compatibility with common media players that rely on filename matching to auto-load subtitles.
- Files that are not recognised as video or subtitle files (e.g. `.txt`, `.pdf`) are never touched, regardless of where they are in the folder structure.

### Recognised file extensions

| Type | Extensions |
|---|---|
| Video | `.mkv`, `.mp4`, `.avi`, `.mov`, `.m4v`, `.ts`, `.m2ts` |
| Subtitle | `.srt`, `.ass`, `.ssa`, `.sub`, `.idx` |

Any file with an extension not in this list is considered a non-media file and is never touched.

---

## Folder structure (Jellyfin convention)

When renaming a series, Scoutarr produces the following folder structure, which follows the official Jellyfin naming convention:

```
Series Name (Year)/
    Season 01/
        Series Name - S01E01 - Episode Title.mkv
        Series Name - S01E01 - Episode Title.en.srt
        Series Name - S01E02 - Episode Title.mkv
    Season 02/
        Series Name - S02E01 - Episode Title.mkv
```

Rules:
- The root folder is renamed to `Series Name (Year)`.
- Season folders are named `Season 01`, `Season 02`, etc. (zero-padded, never abbreviated to `S01` or `SE01`).
- Season folders do not include the series name (to avoid Jellyfin misdetection).
- Non-media files at any level of the folder structure are left in place untouched.

---

## Folder rename corner cases

When renaming the root series folder, two conflict scenarios must be handled:

**A folder with the target name already exists:**
The user must be asked whether to merge the two folders or abort the operation.
- If merge is chosen: media files from the source folder are moved into the existing target folder. Files already present in the target folder are not overwritten.
- If abort is chosen: no changes are made and an error is returned.

**A non-media file exists at any level:**
The file is left in place untouched.

---

## Series state

After a series is identified in Rename mode, a metadata file is written to the root series folder. In Identify mode, no metadata file is written.

### Metadata file

- Format: JSON.
- Filename: `{Series Name} ({Year}).json` (e.g. `Breaking Bad (2008).json`), written to the root series folder.
- Written after a successful series identification in Rename mode.
- If the metadata file is incorrect, the user deletes it and runs the operation again.

The metadata file contains:
- TMDB ID of the series.
- Series title and year.
- Total number of seasons.
- For each season: episode count, whether the season is still airing, and whether the season is complete.
- Whether the series is still airing or has ended.

Season airing status must be kept up to date. On each pass, Scoutarr refreshes season and episode counts from TMDB for any season that was previously marked as still airing. This ensures that a season with 10 episodes today may correctly show 12 episodes on the next pass if new episodes have aired.

### Validation on subsequent passes

When a metadata file already exists, Scoutarr:
- Skips the series identification step.
- Validates each new episode against the known series data (season number and episode count).
- Reports anomalies as errors (e.g. S01E22 when season 1 only has 21 episodes) for the user to resolve.
