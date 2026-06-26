# UC2-TASK-008 — TV episode file move service

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Rename mode", "Folder structure", "Recognised file extensions"
**Dependencies:** UC2-TASK-007 (TvFormatterInput / formatted filename)

---

## Context

Given a source file path and a formatted episode filename, move the video file and all associated subtitle files to the correct destination within `MEDIA_ROOT_TV`, creating the series and season folder structure if needed. After the move, clean up any empty directories left behind in the source path.

This service runs in **Rename mode only**. Identify mode does not touch the filesystem.

No TMDB calls. No filename parsing. No formatting. Pure filesystem operations.

---

## Mount points

| Mount point | Purpose |
|---|---|
| `/input/tv` | Source TV files to be processed |
| `/media/tv` | Destination for renamed TV episodes (Jellyfin library root) |

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
    SourcePath:        string
    SeriesName:        string
    SeriesYear:        int
    Season:            int
    FormattedFilename: string
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
    DestinationPath:       string
    MovedSubtitles:        IReadOnlyList<SubtitleMove>
    CleanedDirectories:    IReadOnlyList<string>
}

SubtitleMove
{
    SourcePath:       string
    DestinationPath:  string
}

TvEpisodeMoveError
{
    Reason:   string
    Detail:   string?
}
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
