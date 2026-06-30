# File Handling

> Part of [Scoutarr Requirements](../REQUIREMENTS.md).

---

## Modes of operation

Scoutarr operates in two distinct modes, both exposed via REST API and MCP:

### Identify mode

Returns suggested filenames without touching the filesystem. Useful when the user does not have a volume mounted, wants to preview changes before applying them, or needs the suggested names to rename manually.

- Does not write, rename, move, or modify any media file or directory.
- For a single file, returns the original filename and the suggested filename.
- For a folder (series), returns the suggested root folder name and a dictionary mapping each original filename to its suggested filename.
- Non-media files are silently omitted from the response — since nothing is modified, the user does not need to be informed about them.
- **Writes the series metadata file** to the root series folder after a successful series identification (TV episode and series folder only). This allows subsequent passes to skip identification even when no rename has been performed yet.

### Rename mode

Applies the rename operation to disk. Internally calls Identify first, then applies the changes.

- For a **movie**: renames the video file and all auxiliary files in place (no moving).
- For a **TV episode**: moves the video file and all auxiliary files to the correct destination within `MEDIA_ROOT_TV`, creating the series and season folder structure if needed. After the move, any empty directories left behind in the source path are deleted, walking upward until reaching `MEDIA_ROOT_TV`.
- For a **series folder**: renames all video and subtitle files in place within the existing folder structure. Does not move files between directories.
- Writes the series metadata file to the root series folder after a successful series identification (TV episode and series folder only).
- Requires the target path to be accessible inside the container (mounted as a volume).

The Rename mode depends on Identify — if identification fails, no renaming occurs.

---

## Mount points

Scoutarr expects five fixed mount points inside the container, all mounted as Docker volumes by the user at container startup:

| Mount point | Purpose |
|---|---|
| `/config` | Configuration and logs |
| `/input/movies` | Source movie files to be identified or renamed |
| `/input/tv` | Source TV files and folders to be identified or renamed |
| `/media/movies` | Destination for renamed movies (Jellyfin library root) |
| `/media/tv` | Destination for renamed TV episodes and series (Jellyfin library root) |

These are not environment variables — they are fixed paths inside the container. Scoutarr never reads from `/media` or writes to `/input`.

Example `docker-compose.yml`:

```yaml
volumes:
  - ./config:/config
  - /your/downloads/movies:/input/movies
  - /your/downloads/tv:/input/tv
  - /your/jellyfin/movies:/media/movies
  - /your/jellyfin/tv:/media/tv
```

---

## Input

- Input: single file or folder.
- A folder is always treated as a TV show.
- A single file requires the caller to specify whether it is a movie or a TV show episode.
- Video file and all subtitle files are renamed (and moved, for TV episodes) together.
- Subtitle files are renamed to match the video filename exactly, with the language code appended before the extension when the language is known (e.g. `Series Name - S01E01 - Episode Title.en.srt`). If the language is unknown, the subtitle is renamed to match the video filename without a language suffix (e.g. `Series Name - S01E01 - Episode Title.srt`). This ensures compatibility with common media players that rely on filename matching to auto-load subtitles.
- Files that are not recognised as video or subtitle files (e.g. `.txt`, `.pdf`) are never touched, regardless of where they are in the folder structure.

### Subtitle language detection

A subtitle file is considered language-coded if its name matches the pattern `{stem}.{lang}.{ext}` where `{lang}` is a 2- or 3-letter ISO language code (e.g. `en`, `es`, `fra`). If the pattern does not match, the subtitle is treated as language-unknown and renamed without a language suffix.

### Recognised file extensions

| Type     | Extensions                                              |
|----------|---------------------------------------------------------|
| Video    | `.mkv`, `.mp4`, `.avi`, `.mov`, `.m4v`, `.ts`, `.m2ts`  |
| Subtitle | `.srt`, `.ass`, `.ssa`, `.sub`, `.idx`                  |

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
    Season 00/
        Series Name - S00E01 - Special Title.mkv
    Season 02/
        Series Name - S02E01 - Episode Title.mkv
```

Rules:

- The root folder is named `Series Name (Year)`.
- Season folders are named `Season 01`, `Season 02`, etc. (zero-padded to 2 digits, never abbreviated to `S01` or `SE01`).
- Season 0 (specials) uses folder name `Season 00`.
- Season folders do not include the series name (to avoid Jellyfin misdetection).
- Non-media files at any level of the folder structure are left in place untouched.

---

## TV episode move behaviour

When renaming a single TV episode in Rename mode:

1. The destination is resolved as `{MEDIA_ROOT_TV}/{Series Name} ({Year})/Season {Season:00}/`.
2. The series and season folders are created if they do not exist.
3. If a file with the target name already exists at the destination, the operation is aborted and an error is returned. Nothing is moved.
4. The video file and all associated subtitle files are moved to the destination and renamed.
5. After the move, empty directories are cleaned up: starting from the source directory, walk upward and delete any directory that is now empty. Stop at `MEDIA_ROOT_TV` — never delete it or any ancestor.

---

## Folder rename corner cases

When renaming the root series folder, two conflict scenarios must be handled:

**A folder with the target name already exists:** The user must be asked whether to merge the two folders or abort the operation.

- If merge is chosen: media files from the source folder are moved into the existing target folder. Files already present in the target folder are not overwritten.
- If abort is chosen: no changes are made and an error is returned.

**A non-media file exists at any level:** The file is left in place untouched.

---

## Series state

After a series is identified successfully, a metadata file is written to the root series folder — both for a single TV episode (UC-02) and for a series folder operation (UC-03), in either Identify or Rename mode. A single episode therefore creates (or updates) the same metadata file that a full series-folder pass would, so that later operations — whether on the same episode, another episode of the same series, or the whole folder — can rely on it being present.

### Metadata file

- Format: JSON.
- Filename: `{Series Name} ({Year}).json` (e.g. `Breaking Bad (2008).json`), written to the root series folder.
- Written after a successful series identification, for both single TV episode and series folder operations, in both Identify and Rename mode.
- If the metadata file is incorrect, the user deletes it and runs the operation again.

The metadata file contains:

- TMDB ID of the series.
- Series title and year.
- Total number of seasons.
- For each season: episode count, whether the season is still airing, and whether the season is complete.
- Whether the series is still airing, has ended, or was cancelled.

Season airing status must be kept up to date. On each pass, Scoutarr refreshes season and episode counts from TMDB for any season that was previously marked as still airing. This ensures that a season with 10 episodes today may correctly show 12 episodes on the next pass if new episodes have aired.

### Validation on subsequent passes

When a metadata file already exists, Scoutarr:

- Skips the series identification step.
- Validates each new episode against the known series data (season number and episode count).
- Reports anomalies as errors (e.g. S01E22 when season 1 only has 21 episodes) for the user to resolve.
