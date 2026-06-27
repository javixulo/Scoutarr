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
- `400` — bad request (missing parameter, invalid format).
- `403` — insufficient write permissions.
- `404` — file or folder not found.
- `409` — a file with the target name already exists.
- `422` — no TMDB match found above the confidence threshold.
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
- In **Identify mode**, only `/input` is read. The destination volumes are never touched.
- In **Rename mode**, files are moved from `/input/movies` or `/input/tv` to the appropriate destination volume.

### docker-compose.yml example

```yaml
services:
  scoutarr:
    image: scoutarr:latest
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
- `POST /identify/movie` — identify movie, returns success or disambiguation (422)
- `POST /rename/movie` — rename movie on disk

UC-02 adds:
- `POST /identify/series` — identify TV series, returns success or disambiguation (422)
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
- `IEpisodeHeuristic` — common interface for episode number heuristics chain
- `ITvEpisodeNumberParser` / `TvEpisodeNumberParser` — heuristic chain; initially only Heuristic 1 (SxEy pattern)
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
| `IEpisodeHeuristic` | Common interface for episode number heuristics | **Reused directly** — UC-03 processes many files using the same chain |
| `ITvEpisodeNumberParser` | Heuristic chain for season/episode extraction | **Reused directly** |
| `ITvShowSearchService` | Searches TMDB for TV shows, scores candidates | **Reused directly** |
| `ITvShowIdentificationService` | Orchestrates series identification flow | **Reused directly** — series is identified once for the whole folder |
| `ITvEpisodeLookupService` | Retrieves episode title from TMDB | **Reused directly** — called once per episode |
| `ITvFilenameFormatter` | Formats episode output filename | **Reused directly** |
| `ITvEpisodeMoveService` | Moves episode and subtitles to destination | **Reused directly** — called once per episode; extended with folder merge logic |
| `ITmdbClient` | TMDB HTTP client | **Reused and extended** — adds `GetTvSeasonDetailsAsync` for episode-level metadata |
| `ConfidenceScorer` | Shared confidence scoring utility | **Reused directly** |

---

### Scoutarr.Core — types

| Type | Description | Reuse in UC-03 |
|---|---|---|
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

UC-03 adds:
| Type | Description |
|---|---|
| `ScannedFolder` | Raw folder name + list of `ScannedVideoFile` + ignored files |
| `ScannedVideoFile` | Video file path + list of `ScannedSubtitle` |
| `ScannedSubtitle` | Subtitle path + optional language code |
| `SeriesMetadata` | Full series metadata persisted to JSON: tmdbId, title, year, seasons, episodes |
| `SeasonMetadata` | Season number, episode count, airing/complete status, episode list |
| `EpisodeMetadata` | Episode number, absolute episode number, title, runtime |
| `SeriesMetadataReadResult` | Discriminated union: `Found`, `NotFound`, `Error` |
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
- `GetTvSeasonDetailsAsync(int tvId, int season)` → full episode list for a season including name and runtime per episode

---

### Scoutarr.Core — filesystem

`IFileSystem` abstraction is in place. After UC-03, `ITvEpisodeMoveService` additionally handles:
- Reusing an existing series destination folder without recreating it (folder merge)
- Skipping files that already exist at destination and reporting them as failures
- Reporting permission errors and path-is-not-a-directory errors as failures

`ISeriesMetadataService` is new in UC-03 and handles:
- Writing `{Series Title} ({Year}).json` to the series root folder
- Reading and deserialising the metadata file on subsequent passes
- Refreshing series and season status from TMDB on every pass
- Refreshing episode list from TMDB for seasons still marked as airing
- Recalculating `AbsoluteEpisodeNumber` after each refresh
- Validating episode numbers against known season data

---

### Scoutarr.Api — REST endpoints

After UC-03:
- `POST /identify/movie` — identify movie
- `POST /rename/movie` — rename movie on disk
- `POST /identify/series` — identify TV series, returns success or disambiguation
- `POST /identify/episode` — identify episode given confirmed series `tvId`
- `POST /rename/episode` — move episode to `/media/tv`
- `POST /identify/folder` — scan folder, identify series, return suggested filenames for all episodes; write metadata file
- `POST /rename/folder` — scan folder, identify series, move all episodes, write metadata file; returns aggregated successes and failures

Both folder endpoints accept an optional `tmdbId` parameter to skip series identification. Partial failures return `200` with the failures described in the response body.

---

### Scoutarr.Mcp — MCP tools

After UC-03:
- `identify_movie`, `rename_movie`
- `identify_series`, `identify_episode`, `rename_episode`
- `identify_folder` — scan folder, identify series, return suggested filenames grouped by season; write metadata file; guide agent through disambiguation if needed
- `rename_folder` — scan folder, identify series, move all episodes, write metadata file; return aggregated summary; guide agent to report successes and failures clearly

The MCP server system prompt is extended with a folder section covering disambiguation behaviour, how to present suggested filenames grouped by season, and how to report partial failures.

---

### What UC-04 must build from scratch

UC-04 will be defined separately. At a minimum, it will find the following infrastructure ready to use:

- Complete movie identification and renaming pipeline (UC-01)
- Complete TV episode identification and renaming pipeline, single file (UC-02)
- Complete TV folder processing pipeline with series metadata persistence (UC-03)
- `IFileSystem` abstraction covering all filesystem operations
- `ITmdbClient` covering movie search, TV search, episode details, season details, and series enrichment
- `ISeriesMetadataService` for reading, writing, refreshing, and validating series metadata
- `IFolderRenameOrchestrator` for end-to-end folder processing
- REST API with six endpoints across movie, episode, and folder operations
- MCP server with six tools and a system prompt covering all disambiguation and multi-step flows
- Full test coverage at unit, integration, and E2E layers
