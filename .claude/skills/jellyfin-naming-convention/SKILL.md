---
name: jellyfin-naming-convention
description: Use this skill when working on any task that involves output filename or folder name formatting for movies or TV shows.
---

# Jellyfin Naming Conventions Skill

## Movies

```
{title} ({year}).{ext}
```

Examples:
```
The Batman (2022).mkv
Pulp Fiction (1994).mkv
```

---

## TV Shows — Folder structure

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
- Root folder: `Series Name (Year)` — includes the year.
- Season folders: `Season 01`, `Season 02` — zero-padded, never `S01` or `SE01`.
- Season folders do not include the series name.
- Episode files: `Series Name - S{season}E{episode} - Episode Title.{ext}` — season and episode zero-padded.

---

## Subtitle files

Subtitles must match the video filename exactly, with the language code appended before the extension when known:

```
Series Name - S01E01 - Episode Title.en.srt   ← language known
Series Name - S01E01 - Episode Title.srt       ← language unknown
```

---

## Naming format templates

Templates are configured via environment variables. Available tokens:

**Movies:**
| Token | Value |
|---|---|
| `{title}` | Movie title from TMDB |
| `{year}` | Release year |
| `{resolution}` | e.g. 1080p, 4K — omitted if not detected |
| `{edition}` | e.g. Director's Cut — omitted if not available |

**TV episodes:**
| Token | Value |
|---|---|
| `{series}` | Series title from TMDB |
| `{year}` | Series start year |
| `{season}` | Season number, zero-padded |
| `{episode}` | Episode number, zero-padded |
| `{title}` | Episode title from TMDB |
| `{resolution}` | e.g. 1080p, 4K — omitted if not detected |

**Default formats:**
- Movie: `{title} ({year})`
- Episode: `{series} - S{season}E{episode} - {title}`

---

## Non-media files

Files not recognised as media or auxiliary (e.g. `.txt`, `.pdf`) are never renamed or moved, at any level of the folder structure.
