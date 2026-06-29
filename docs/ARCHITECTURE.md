# Scoutarr – Architecture

> This document contains technical implementation decisions derived from [REQUIREMENTS.md](REQUIREMENTS.md).

---

## 1. Technology Stack

- **Language:** C# / .NET (latest LTS)
- **Runtime target:** Linux (Docker), cross-platform
- **Metadata source:** TMDB API via [TMDbLib](https://github.com/LordMike/TMDbLib) (.NET client)

---

## 2. Interfaces

### REST API

- Follows **RFC 9457 (Problem Details for HTTP APIs)** for error responses.

Success response shape:
```json
{
  "originalFilename": "The.Batman.2022.1080p.mkv",
  "filename": "The Batman (2022).mkv",
  "mode": "identify",
  "match": {
    "tmdbId": 414906,
    "title": "The Batman",
    "year": 2022,
    "confidence": 0.97
  }
}
```

The `filename` field contains the suggested filename in both modes. In `identify` mode it is a suggestion only — no files are modified. In `rename` mode it is the actual new name on disk.

Error response shape (RFC 9457):
```json
{
  "type": "https://scoutarr.io/errors/file-not-found",
  "title": "File not found",
  "status": 404,
  "detail": "The file '/media/movies/batman.mkv' does not exist or is not accessible.",
  "instance": "/rename/movie"
}
```

HTTP status codes:
- `200` — success.
- `300` — disambiguation needed (multiple TMDB candidates above threshold; the caller selects one and retries with `candidateIndex`).
- `400` — bad request (missing parameter, invalid format).
- `403` — insufficient write permissions.
- `404` — file or folder not found.
- `409` — a file with the target name already exists.
- `422` — no TMDB match found above the confidence threshold, or metadata file malformed.
- `500` — internal error (TMDB unreachable, etc.).

### MCP Server

- Transport: **HTTP/SSE** — the container exposes the MCP server on a configurable port, allowing AI agent clients to connect over HTTP.
- Response model:
  - **Successful tool result** — returns structured data the agent can use to inform the user naturally.
  - **Tool result with `isError: true`** — for business errors (file not found, no match, permission denied, etc.). The agent can read the error, reason about it, and decide how to proceed.
  - **MCP protocol error** — reserved for server-level or protocol-level failures.

Client configuration example (Claude Desktop):
```json
{
  "mcpServers": {
    "scoutarr": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

---

## 3. Logging

Logs are written simultaneously to two destinations:

- **stdout/stderr** — Docker captures these streams via its logging drivers. Accessible with `docker logs scoutarr` or any compatible log management system.
- **Log file** — written to `/config/logs/scoutarr.log` inside the container. Since `/config` is mounted as a volume, logs persist on the host machine across container restarts or destruction.

This follows the same convention used by Radarr, Sonarr, and other *arr applications:

```
/config
    logs/
        scoutarr.log
```

---

## 4. Deployment

### Volume structure

Scoutarr uses five mount points inside the container:

```
/config          ← configuration and logs (mounted from host)
/input/movies    ← source movie files to be identified or renamed (mounted from host)
/input/tv        ← source TV files and folders to be identified or renamed (mounted from host)
/media/movies    ← destination for renamed movies, following Jellyfin convention (mounted from host)
/media/tv        ← destination for renamed TV episodes and series, following Jellyfin convention (mounted from host)
```

- `/input/movies` and `/input/tv` are where the user places dirty files or folders to be processed. They can point to a downloads directory or any staging area on the host.
- `/media/movies` and `/media/tv` are the Jellyfin library roots. Scoutarr moves renamed files here.
- In **Identify mode**, only `/input` is read for media files. The series metadata file is still written to the series folder after a successful identification.
- In **Rename mode**, files are moved from `/input/movies` or `/input/tv` to the appropriate destination volume.

### Branching strategy and image tags

Scoutarr uses a two-branch model. No image is published to `latest` until the project owner explicitly decides a version is ready for release.

| Branch | Trigger | Image tag | Purpose |
|--------|---------|-----------|--------|
| `dev` | push to `dev` | `edge` | Pre-release testing — local use only, may be unstable |
| `main` | push of a `v*` tag (e.g. `v0.1.0`) | `v0.1.0` + `latest` | Public release — only when the project owner tags |

**Development flow:**
1. All feature branches are merged into `dev` via pull request.
2. CI runs on every PR (unit + integration tests).
3. On merge to `dev`, CD builds the image and publishes it as `edge` to `ghcr.io/javixulo/scoutarr:edge`. This image is used for local pre-release testing.
4. When a version is considered release-ready, the project owner merges `dev` into `main` and pushes a `v*` tag.
5. CD detects the tag, builds the image, and publishes it as both `ghcr.io/javixulo/scoutarr:<tag>` and `ghcr.io/javixulo/scoutarr:latest`.

**Local testing with the `edge` image:**
```yaml
services:
  scoutarr:
    image: ghcr.io/javixulo/scoutarr:edge
    # ... rest of config
```

**Production use with a pinned release:**
```yaml
services:
  scoutarr:
    image: ghcr.io/javixulo/scoutarr:v0.1.0
    # ... rest of config
```

### docker-compose.yml example

```yaml
services:
  scoutarr:
    image: ghcr.io/javixulo/scoutarr:latest
    container_name: scoutarr
    environment:
      - TMDB_API_KEY=your_key_here
      - SCRAPING_LANGUAGE=en
      - CONFIDENCE_THRESHOLD=0.80
      - NAMING_FORMAT_MOVIE={title} ({year})
      - NAMING_FORMAT_EPISODE={series} - S{season:00}E{episode:00} - {title}
    volumes:
      - ./config:/config
      - /your/downloads/movies:/input/movies
      - /your/downloads/tv:/input/tv
      - /your/jellyfin/movies:/media/movies
      - /your/jellyfin/tv:/media/tv
    ports:
      - "8080:8080"
    restart: unless-stopped
```

### Environment variables

| Variable | Description | Default |
|---|---|---|
| `TMDB_API_KEY` | TMDB API key (required) | — |
| `SCRAPING_LANGUAGE` | Language for TMDB metadata | `en` |
| `CONFIDENCE_THRESHOLD` | Minimum confidence for auto-accept (0.0–1.0) | `0.80` |
| `NAMING_FORMAT_MOVIE` | Output filename format for movies | `{title} ({year})` |
| `NAMING_FORMAT_EPISODE` | Output filename format for episodes | `{series} - S{season:00}E{episode:00} - {title}` |

---

## 5. Project Structure

Scoutarr is a .NET solution composed of multiple projects, following a layered architecture where `Scoutarr.Core` is the single source of business logic, and both interfaces (`Api` and `Mcp`) are thin layers that delegate to it.

```
Scoutarr.sln
    src/
        Scoutarr.Core        ← business logic: scraping, renaming, heuristics, TMDB client
        Scoutarr.Api         ← REST API (ASP.NET Core)
        Scoutarr.Mcp         ← MCP Server (HTTP/SSE)
    tests/
        Scoutarr.Core.Tests  ← unit tests
        Scoutarr.Api.Tests   ← integration tests for REST API
        Scoutarr.Mcp.Tests   ← integration tests for MCP Server
        Scoutarr.E2E.Tests   ← end-to-end tests against the Docker container
```

### Dependency rules

- `Scoutarr.Core` has no knowledge of HTTP, MCP, or any interface concern.
- `Scoutarr.Api` and `Scoutarr.Mcp` depend on `Scoutarr.Core`, never on each other.
- Both interfaces use the same service layer in Core via dependency injection.

### Error handling consistency

Core defines its own error model (domain errors). Each interface is responsible for translating those domain errors into its own format — RFC 9457 for REST, `isError: true` for MCP. This ensures consistent behaviour across both interfaces without duplicating business logic.

### Validation responsibility

- Interfaces validate the shape and format of incoming requests (e.g. required fields, valid paths).
- Core validates business rules (e.g. file exists, episode number is valid for the series).

---

## 6. Testing Strategy

All test layers are part of the MVP. The application is not considered releasable without passing tests at every layer.

### Unit tests (`Scoutarr.Core.Tests`)

Cover pure business logic with no external dependencies:
- Filename parsing and heuristics (e.g. detecting season/episode numbers from dirty filenames).
- Confidence score calculation.
- Output filename formatting.
- Series metadata validation (e.g. episode out of range).

### Integration tests (`Scoutarr.Api.Tests`, `Scoutarr.Mcp.Tests`)

Cover each interface in isolation, with external dependencies mocked (TMDB API, file system):
- Correct response shapes for success and error cases.
- Correct HTTP status codes (REST).
- Correct `isError` behaviour (MCP).
- Validation of incoming requests.

### End-to-end tests (`Scoutarr.E2E.Tests`)

Cover the full system running inside Docker:
- The container starts and both interfaces are reachable.
- A real rename operation completes successfully end-to-end.
- Errors are correctly returned from the live container.
- Logs are written to `/config/logs/scoutarr.log` inside the mounted volume.

E2E tests use a real TMDB API key (from environment) and a temporary directory with test media files. They run against the built Docker image, not the source code directly.

---

## 7. State after UC-01 — What exists for UC-02 to build on

This section summarises what is implemented and available after UC-01 (rename a single movie file) is complete. UC-02 (rename a single TV show episode) should extend or reuse these pieces rather than rebuilding them.

---

### Scoutarr.Core — interfaces

| Interface | Purpose | Reuse in UC-02 |
|---|---|---|
| `IMovieFilenameParser` | Parses dirty movie filename → `ParsedMovieFilename` | Not reused — TV needs `ITvSeriesTitleParser` |
| `IMovieSearchService` | Searches TMDB for movies, scores candidates | Pattern to follow for `ITvShowSearchService` |
| `IMovieFilenameFormatter` | Formats output filename from template | Pattern to follow for `ITvFilenameFormatter` |
| `IMovieIdentificationService` | Orchestrates parse → search → format | Pattern to follow for `ITvShowIdentificationService` |
| `ITmdbClient` | TMDB HTTP client | **Reused and extended** — add `SearchTvShowAsync`, `GetTvShowDetailsAsync`, and `GetEpisodeDetailsAsync` |

---

### Scoutarr.Core — types

| Type | Description | Reuse in UC-02 |
|---|---|---|
| `ParsedMovieFilename` | Title, year, resolution, release group | New `ParsedTvSeriesTitle` — same shape for series title extraction |
| `MovieCandidate` | TmdbId, title, year, overview, originalLanguage, confidence | New `TvShowCandidate` — same shape, different TMDB source |
| `MovieSearchResult` | Ordered list of `MovieCandidate` | New `TvShowSearchResult` — same pattern |
| `MovieFormatterInput` | Title, year, resolution, edition, template | New `TvFormatterInput` — adds series, season, episode, episodeTitle |
| `MovieIdentificationRequest` | Filename + optional hints + tmdbId | New `TvShowIdentificationRequest` — same optional hints |
| `MovieIdentificationSuccess` | originalFilename, suggestedFilename, match | New `TvShowIdentificationSuccess` — same shape |
| `MovieDisambiguationNeeded` | Candidates list | New `TvShowDisambiguationNeeded` — same pattern |
| `MovieIdentificationError` | Reason + detail | Shared domain error model — **reused directly** |

---

### Scoutarr.Core — confidence scoring

The confidence scoring formula (60% title similarity / 30% year match / 10% popularity) is implemented in `MovieSearchService` and extracted into a shared `ConfidenceScorer` utility class (added as part of UC-01 completion). UC-02 reuses `ConfidenceScorer` directly in `TvShowSearchService`.

---

### Scoutarr.Core — ITmdbClient

After UC-01, `ITmdbClient` has:
- `SearchMovieAsync(string title, int? year, ...)` → list of movie candidates
- `GetMovieDetailsAsync(int movieId)` → enriched movie details (director, cast, collection, genres)

UC-02 adds:
- `SearchTvShowAsync(string title, int? year, ...)` → list of TV show candidates
- `GetTvShowDetailsAsync(int tvId)` → TV show details including season/episode structure and series status
- `GetEpisodeDetailsAsync(int tvId, int season, int episode)` → episode title

The real implementation (`TmdbClient`) is extended. Tests continue to mock `ITmdbClient`.

---

### Scoutarr.Api — REST endpoints

After UC-01:
- `POST /identify/movie` — identify movie, returns success or disambiguation (300)
- `POST /rename/movie` — rename movie on disk

UC-02 adds:
- `POST /identify/series` — identify TV series, returns success or disambiguation (300)
- `POST /identify/episode` — identify episode given confirmed series tmdbId
- `POST /rename/episode` — move episode to correct destination in `/media/tv`

The domain error → HTTP status code mapping is shared. UC-02 reuses it directly.

---

### Scoutarr.Mcp — MCP tools

After UC-01:
- `identify_movie` — returns success or enriched disambiguation result
- `rename_movie` — renames on disk

UC-02 adds:
- `identify_series` — same pattern, TV series; enriched with creator, cast, network, seasons with episode counts, and series status
- `identify_episode` — given confirmed series tmdbId, identifies the episode
- `rename_episode` — moves episode to correct destination in `/media/tv`

The MCP server system prompt is extended to cover TV show and episode disambiguation behaviour.

---

### What UC-02 must build from scratch

- `ITvSeriesTitleParser` / `TvSeriesTitleParser` — extracts series title (and optional year, resolution) from dirty filename
- `EpisodeParseContext` — record carrying `Filename`, `Metadata?`, and `FileDuration?`; passed to every heuristic instead of individual parameters
- `IEpisodeHeuristic` — common interface for episode number heuristics chain; `TryParse(EpisodeParseContext)` → `ParsedEpisodeNumber?`
- `ITvEpisodeNumberParser` / `TvEpisodeNumberParser` — heuristic chain; initially only H1 (SxEy pattern)
- `ITvShowSearchService` / `TvShowSearchService` — searches TMDB for TV shows, scores candidates using `ConfidenceScorer`
- `ITvShowIdentificationService` / `TvShowIdentificationService` — orchestrates series identification flow
- `ITvEpisodeLookupService` / `TvEpisodeLookupService` — retrieves episode title from TMDB given confirmed series and episode number
- `ITvFilenameFormatter` / `TvFilenameFormatter` — formats episode output filename
- `ITvEpisodeMoveService` / `TvEpisodeMoveService` — moves episode and subtitles to correct destination, creates folder structure, cleans up empty directories
- REST controllers and MCP tools for TV series and episode operations

---

## 8. State after UC-02 — What exists for UC-03 to build on

This section summarises what is implemented and available after UC-02 (rename a single TV show episode) is complete. UC-03 (rename all episodes in a folder) should extend or reuse these pieces rather than rebuilding them.

---

### Scoutarr.Core — interfaces

| Interface | Purpose | Reuse in UC-03 |
|---|---|---|
| `IMovieFilenameParser` | Parses dirty movie filename | Not relevant for UC-03 |
| `IMovieIdentificationService` | Orchestrates movie identification | Not relevant for UC-03 |
| `ITvSeriesTitleParser` | Extracts series title from dirty filename | **Reused directly** |
| `IEpisodeHeuristic` | Common interface for episode number heuristics; `TryParse(EpisodeParseContext)` → `ParsedEpisodeNumber?` | **Reused directly** — UC-03 processes many files using the same chain |
| `ITvEpisodeNumberParser` | Heuristic chain for season/episode extraction | **Reused directly** |
| `ITvShowSearchService` | Searches TMDB for TV shows, scores candidates | **Reused directly** |
| `ITvShowIdentificationService` | Orchestrates series identification flow | **Reused directly** — series is identified once for the whole folder |
| `ITvEpisodeLookupService` | Retrieves episode title from TMDB | **Reused directly** — called once per episode |
| `ITvFilenameFormatter` | Formats episode output filename | **Reused directly** |
| `ITvEpisodeMoveService` | Moves episode and subtitles to destination | **Reused directly** — called once per episode |
| `ITmdbClient` | TMDB HTTP client | **Reused** — UC-03 may extend with batch or season-level calls if needed |
| `ConfidenceScorer` | Shared confidence scoring utility | **Reused directly** |

---

### Scoutarr.Core — types

| Type | Description | Reuse in UC-03 |
|---|---|---|
| `EpisodeParseContext` | Filename, optional `SeriesMetadata`, optional `FileDuration` (TimeSpan) | **Reused directly** |
| `ParsedTvSeriesTitle` | Series title, optional year and resolution | **Reused directly** |
| `ParsedEpisodeNumber` | Season and episode number | **Reused directly** |
| `TvShowCandidate` | TMDB series candidate with confidence score | **Reused directly** |
| `TvShowIdentificationSuccess` | Confirmed series match | **Reused directly** |
| `TvShowDisambiguationNeeded` | Candidate list for disambiguation | **Reused directly** |
| `TvEpisodeLookupSuccess` | Confirmed episode with title | **Reused directly** |
| `TvFormatterInput` | Input for episode filename formatter | **Reused directly** |
| `TvEpisodeMoveSuccess` | Move result including destination path and subtitles | **Reused directly** |
| `TvShowDetails` | Season/episode structure from TMDB | **Reused directly** |
| `TvShowEnrichment` | Creator, cast, network, genres, status from TMDB | **Reused directly** |
| `SubtitleMove` | Source and destination of a moved subtitle file | **Reused directly** |
| Shared domain error model | Reason + detail | **Reused directly** |

---

### Scoutarr.Core — ITmdbClient

After UC-02, `ITmdbClient` has:
- `SearchMovieAsync` → list of movie candidates
- `GetMovieDetailsAsync` → enriched movie details
- `SearchTvShowAsync` → list of TV show candidates
- `GetTvShowDetailsAsync` → season/episode structure
- `GetTvShowEnrichmentAsync` → creator, cast, network, genres, status
- `GetEpisodeDetailsAsync` → episode title

UC-03 may add:
- Season-level episode list fetching if bulk lookup is needed for performance

---

### Scoutarr.Core — filesystem

`IFileSystem` abstraction is in place. `ITvEpisodeMoveService` handles:
- Creating series and season folder structure
- Moving video and subtitle files with rollback on failure
- Cleaning up empty source directories

UC-03 reuses `ITvEpisodeMoveService` directly for each episode in the folder. UC-03 adds:
- Series metadata file write (`{Series Name} ({Year}).json`) to the series root folder after a successful pass, in both Identify and Rename mode
- Folder merge logic when a series folder with the target name already exists

---

### Scoutarr.Api — REST endpoints

After UC-02:
- `POST /identify/movie` — identify movie
- `POST /rename/movie` — rename movie on disk
- `POST /identify/series` — identify TV series, returns success or disambiguation
- `POST /identify/episode` — identify episode given confirmed series `tvId`
- `POST /rename/episode` — move episode to `/media/tv` (requires confirmed `tvId`)

UC-03 adds:
- `POST /identify/folder` — identify series from a folder path
- `POST /rename/folder` — rename all episodes in a folder

---

### Scoutarr.Mcp — MCP tools

After UC-02:
- `identify_movie`, `rename_movie`
- `identify_series` — with enrichment (creator, cast, network, seasons, status)
- `identify_episode` — given confirmed series `tvId`
- `rename_episode` — moves to `/media/tv`, returns destination path and moved subtitles

UC-03 adds:
- `identify_folder` — identify series from a folder
- `rename_folder` — rename all episodes in a folder, write metadata file

---

### What UC-03 must build from scratch

- Series metadata file writer — writes `{Series Name} ({Year}).json` to the series root folder after a successful folder identification (both Identify and Rename mode)
- Folder scanner — discovers all video files in a folder, grouped by season
- Folder rename orchestrator — identifies the series once, then processes each episode using existing UC-02 services
- Folder merge logic — handles the case where a series folder with the target name already exists
- REST controllers and MCP tools for folder operations

---

## 9. State after UC-03 — What exists for UC-04 to build on

This section summarises what is implemented and available after UC-03 (rename all episodes in a folder) is complete. UC-04 should extend or reuse these pieces rather than rebuilding them.

---

### Scoutarr.Core — interfaces

| Interface | Purpose | Reuse in UC-04 |
|---|---|---|
| `IMovieFilenameParser` | Parses dirty movie filename | **Reused directly** |
| `IMovieIdentificationService` | Orchestrates movie identification | **Reused directly** |
| `ITvSeriesTitleParser` | Extracts series title from dirty filename | **Reused directly** |
| `IEpisodeHeuristic` | Common interface for episode number heuristics; `TryParse(EpisodeParseContext)` → `ParsedEpisodeNumber?` | **Reused directly** |
| `ITvEpisodeNumberParser` | Heuristic chain for season/episode extraction | **Reused directly** |
| `ITvShowSearchService` | Searches TMDB for TV shows, scores candidates | **Reused directly** |
| `ITvShowIdentificationService` | Orchestrates series identification flow | **Reused directly** |
| `ITvEpisodeLookupService` | Retrieves episode title from TMDB | **Reused directly** |
| `ITvFilenameFormatter` | Formats episode output filename | **Reused directly** |
| `ITvEpisodeMoveService` | Moves episode and subtitles to destination; handles folder merge | **Reused directly** |
| `IFolderScanner` | Discovers all video and subtitle files in a folder | **Reused directly** |
| `ISeriesMetadataService` | Reads, writes, refreshes, and validates series metadata | **Reused directly** |
| `IFolderRenameOrchestrator` | End-to-end folder processing: identify once, rename all | **Reused directly** |
| `ITmdbClient` | TMDB HTTP client | **Reused directly** |
| `ConfidenceScorer` | Shared confidence scoring utility | **Reused directly** |

---

### Scoutarr.Core — types

All types from UC-01 and UC-02 remain available. UC-03 adds:

| Type | Description |
|---|---|
| `ScannedFolder` | Raw folder name + list of `ScannedVideoFile` + ignored file paths |
| `ScannedVideoFile` | Video file path + list of `ScannedSubtitle` |
| `ScannedSubtitle` | Subtitle path + optional ISO language code |
| `SeriesMetadata` | Full series metadata persisted to JSON: tmdbId, title, year, airing status, seasons |
| `SeasonMetadata` | Season number, episode count, airing/complete status, episode list |
| `EpisodeMetadata` | Episode number, absolute episode number (null for Season 0), title, runtime in minutes, air date (DateOnly, nullable) |
| `SeriesMetadataReadResult` | Discriminated union: `SeriesMetadataFound`, `SeriesMetadataNotFound`, `SeriesMetadataError` |
| `FolderIdentifyResult` | Series title, tmdbId, suggested filenames per episode, failures |
| `FolderRenameResult` | Series title, tmdbId, rename successes, failures |
| `EpisodeIdentifyResult` | Original filename + suggested filename |
| `EpisodeRenameSuccess` | Original filename, new filename, destination path |
| `EpisodeFolderFailure` | Original filename, reason, detail |

---

### Scoutarr.Core — ITmdbClient

After UC-03, `ITmdbClient` has:
- `SearchMovieAsync` → list of movie candidates
- `GetMovieDetailsAsync` → enriched movie details
- `SearchTvShowAsync` → list of TV show candidates
- `GetTvShowDetailsAsync` → season/episode structure and series status
- `GetTvShowEnrichmentAsync` → creator, cast, network, genres, status
- `GetEpisodeDetailsAsync` → episode title
- `GetTvSeasonDetailsAsync(int tvId, int season)` → full episode list for a season including name, runtime, and air date per episode

---

### Scoutarr.Core — ISeriesMetadataService

New in UC-03. Handles the full lifecycle of `{Series Title} ({Year}).json`:
- Writing the file to the series root folder after a successful identification in both Identify and Rename mode
- Reading and deserialising on subsequent passes (`SeriesMetadataReadResult`)
- Refreshing series and season airing status from TMDB on every pass — always, regardless of whether any season is currently airing, to detect reactivations of cancelled or ended series
- Refreshing the full episode list (title, runtime, air date) only for seasons still marked as airing after the status update
- Recalculating `AbsoluteEpisodeNumber` after each refresh
- Validating episode numbers against known season data

---

### Scoutarr.Core — ITvEpisodeMoveService (extended)

Extended in UC-03 with folder merge behaviour:
- Reuses an existing series destination folder without recreating it
- Skips files that already exist at destination and reports them as `EpisodeFolderFailure` with reason "File already exists at destination"
- Reports permission errors as `EpisodeFolderFailure` with reason "Insufficient permissions at destination"
- Reports path-is-not-a-directory errors as `EpisodeFolderFailure` with reason "Destination path is not a directory"

---

### Scoutarr.Api — REST endpoints

After UC-03:
- `POST /identify/movie` — identify movie
- `POST /rename/movie` — rename movie on disk
- `POST /identify/series` — identify TV series, returns success or disambiguation
- `POST /identify/episode` — identify episode given confirmed series `tvId`
- `POST /rename/episode` — move episode to `/media/tv`
- `POST /identify/folder` — scan folder, identify series, return suggested filenames for all episodes; write metadata file; accepts optional `tmdbId` to skip identification
- `POST /rename/folder` — scan folder, identify series, move all episodes, write metadata file; returns aggregated successes and failures; partial failures return `200`

---

### Scoutarr.Mcp — MCP tools

After UC-03:
- `identify_movie`, `rename_movie`
- `identify_series`, `identify_episode`, `rename_episode`
- `identify_folder` — scan folder, identify series, return suggested filenames grouped by season; write metadata file; guide agent through disambiguation using one focused question at a time; offer to call `rename_folder` if user confirms
- `rename_folder` — scan folder, identify series, move all episodes, write metadata file; return aggregated summary; guide agent to report successes and failures clearly, with prominent warning if all episodes failed

The MCP server system prompt covers movie, episode, and folder disambiguation flows, including how to present suggested filenames grouped by season and how to handle partial failures.

---

### What UC-04 must build from scratch

UC-04 will be defined separately. At a minimum, it will find the following infrastructure ready to use:

- Complete movie identification and renaming pipeline (UC-01)
- Complete TV episode identification and renaming pipeline, single file (UC-02)
- Complete TV folder processing pipeline with series metadata persistence (UC-03)
- `IFileSystem` abstraction covering all filesystem operations
- `ITmdbClient` with seven methods covering movies, TV shows, episodes, seasons, and enrichment
- `ISeriesMetadataService` for the full metadata file lifecycle
- `IFolderScanner` for recursive folder discovery with subtitle pairing
- `IFolderRenameOrchestrator` for end-to-end folder processing in both identify and rename modes
- REST API with seven endpoints across movie, episode, and folder operations
- MCP server with seven tools and a system prompt covering all disambiguation and multi-step flows
- Full test coverage at unit, integration, and E2E layers

---

## 10. State after UC-04 — What exists for UC-05 to build on

This section summarises what is implemented and available after UC-04 (subsequent pass on a known series — single new episode) is complete. UC-05 should extend or reuse these pieces rather than rebuilding them.

---

### Scoutarr.Core — interfaces

| Interface | Purpose | Reuse in UC-05 |
|---|---|---|
| All interfaces from UC-01 through UC-03 | See sections 7, 8, 9 | **All reused directly** |
| `IKnownSeriesLocator` | Searches `/media/tv` for an existing series folder matching a given title and optional year; returns `KnownSeriesFound`, `KnownSeriesAmbiguous`, or `KnownSeriesNotFound` | **Reused directly** |

---

### Scoutarr.Core — types

All types from UC-01 through UC-03 remain available. UC-04 adds:

| Type | Description |
|---|---|
| `KnownSeriesMatchResult` | Discriminated union: `KnownSeriesFound`, `KnownSeriesAmbiguous`, `KnownSeriesNotFound` |
| `KnownSeriesFound` | Series name, year, absolute folder path in `/media/tv`, loaded `SeriesMetadata` |
| `KnownSeriesAmbiguous` | List of `KnownSeriesCandidate` — each with series name, year, and folder path |
| `KnownSeriesCandidate` | Series name, year, absolute folder path in `/media/tv` |
| `KnownSeriesNotFound` | No fields — signals caller to use the standard UC-02 flow |

---

### Scoutarr.Core — rename_episode orchestration (extended)

UC-04 extends the existing `rename_episode` orchestrator with a new branch at the start of the flow:

1. Call `IKnownSeriesLocator` with the title extracted from the parent folder name (preferred) or the filename.
2. If `KnownSeriesNotFound` → fall through to the existing UC-02 flow unchanged.
3. If `KnownSeriesAmbiguous` → go to TMDB disambiguation (same flow as UC-02); if the resolved `tmdbId` matches a candidate folder, continue as `KnownSeriesFound`; otherwise fall through to UC-02.
4. If `KnownSeriesFound`:
   - Parse the episode number with `ITvEpisodeNumberParser`.
   - If the episode's season is marked as **not airing** in the metadata: validate episode against metadata as-is — no TMDB call.
   - If the episode's season is marked as **airing**: refresh that season via `ISeriesMetadataService`, persist the updated metadata, then validate.
   - If validation fails (episode out of range or season not found): return a domain error, no files moved.
   - If validation passes: read episode title from `SeriesMetadata` (no `ITvEpisodeLookupService` call); format output filename; move file and subtitles with `ITvEpisodeMoveService`.

No changes to `Scoutarr.Api`, `Scoutarr.Mcp`, or any interface layer — the new branch is entirely within `Scoutarr.Core`.

---

### Scoutarr.Core — IKnownSeriesLocator

New in UC-04. Responsibilities:

- List all folders in `/media/tv` using `IFileSystem`.
- Parse each folder name as `{Series Name} ({Year})` — folders that do not match this pattern are silently ignored.
- Match against the extracted title (and optional year) using case-insensitive comparison.
- When a match is found, delegate to `ISeriesMetadataService` to read the metadata file. If the file is malformed, return a domain error rather than a `KnownSeriesFound` result.
- Title extraction prefers the parent folder name over the filename; both are normalised using the same cleaning rules as `ITvSeriesTitleParser`.

---

### Scoutarr.Api — REST endpoints

No new endpoints added in UC-04. The existing `POST /rename/episode` endpoint now transparently handles both new series (UC-02 flow) and known series (UC-04 flow) based on whether a matching folder exists in `/media/tv`.

---

### Scoutarr.Mcp — MCP tools

No new tools added in UC-04. The existing `rename_episode` tool now transparently handles both flows. No changes to the system prompt are needed — the outcome is the same from the agent's perspective.

---

### What UC-05 must build from scratch

UC-05 adds four new episode number heuristics (H2–H5) to the existing chain in `TvEpisodeNumberParser`. At a minimum, it will find the following infrastructure ready to use:

- Complete movie identification and renaming pipeline (UC-01)
- Complete TV episode identification and renaming pipeline, single file (UC-02)
- Complete TV folder processing pipeline with series metadata persistence (UC-03)
- Known series local matching via `IKnownSeriesLocator`, integrated into `rename_episode` (UC-04)
- Selective metadata refresh: TMDB is only called for seasons still marked as airing
- `EpisodeParseContext` — carries `Filename`, `Metadata?`, and `FileDuration?` into every heuristic
- `IEpisodeHeuristic` with `TryParse(EpisodeParseContext)` → `ParsedEpisodeNumber?`
- `TvEpisodeNumberParser` with H1 registered; UC-05 adds H2–H5 in order
- `SeriesMetadata` with `EpisodeMetadata.AirDate` and `EpisodeMetadata.RuntimeMinutes` available to heuristics
- `IFileSystem` abstraction covering all filesystem operations
- `ITmdbClient` with seven methods
- `ISeriesMetadataService` for the full metadata file lifecycle including selective season refresh
- `IFolderScanner`, `IFolderRenameOrchestrator`
- REST API with seven endpoints
- MCP server with seven tools
- Full test coverage at unit, integration, and E2E layers

---

## 11. State after UC-05 — What exists after all heuristics are implemented

This section summarises what is implemented and available after UC-05 (episode number heuristics H2–H5) is complete.

---

### Scoutarr.Core — heuristic chain

The `TvEpisodeNumberParser` heuristic chain after UC-05:

| Order | Class | Strategy |
|---|---|---|
| H1 | `SxEyHeuristic` | Explicit `SxxEyy` pattern in filename |
| H2 | `AbsoluteEpisodeNumberHeuristic` | Bare number matched against `AbsoluteEpisodeNumber` in metadata; compact `SxEy` encoding as fallback; title words for disambiguation |
| H3 | `SpecialEpisodeDurationHeuristic` | File duration matched against `RuntimeMinutes` of Season 0 episodes; title words for disambiguation; requires `FileDuration` in context |
| H4 | `AirDateHeuristic` | Date extracted from filename matched against `AirDate` in metadata; title words for disambiguation |
| H5 | `TitleMatchHeuristic` | All significant title words matched against episode titles in metadata; retry without stop words if no match |

All heuristics implement `IEpisodeHeuristic.TryParse(EpisodeParseContext)` → `ParsedEpisodeNumber?`. The chain stops at the first non-null result.

### External dependency added in UC-05

`MediaInfo.Wrapper.Core` (NuGet) is added to `Scoutarr.Core` to read file duration upstream before invoking the heuristic chain. The duration is placed in `EpisodeParseContext.FileDuration`. On Linux/Docker, requires system packages `libzen0v5 libmms0 zlib1g libnghttp2-14 librtmp1 libcurl4` (see TASK-000).
