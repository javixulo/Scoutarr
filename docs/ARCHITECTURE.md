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

```
/config     ← configuration and logs (mounted from host)
/media      ← media files to be renamed (mounted from host)
```

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
      - /your/media:/media
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
| `IMovieFilenameParser` | Parses dirty movie filename → `ParsedMovieFilename` | Not reused — TV needs `ITvFilenameParser` |
| `IMovieSearchService` | Searches TMDB for movies, scores candidates | Pattern to follow for `ITvShowSearchService` |
| `IMovieFilenameFormatter` | Formats output filename from template | Pattern to follow for `ITvFilenameFormatter` |
| `IMovieIdentificationService` | Orchestrates parse → search → format | Pattern to follow for `ITvIdentificationService` |
| `ITmdbClient` | TMDB HTTP client | **Reused and extended** — add `SearchTvShowAsync` and `GetTvShowDetailsAsync` |

---

### Scoutarr.Core — types

| Type | Description | Reuse in UC-02 |
|---|---|---|
| `ParsedMovieFilename` | Title, year, resolution, release group | New `ParsedTvFilename` adds season, episode number |
| `MovieCandidate` | TmdbId, title, year, overview, originalLanguage, confidence | New `TvShowCandidate` — same shape, different TMDB source |
| `MovieSearchResult` | Ordered list of `MovieCandidate` | New `TvShowSearchResult` — same pattern |
| `MovieFormatterInput` | Title, year, resolution, edition, template | New `TvFormatterInput` — adds series, season, episode, episodeTitle |
| `MovieIdentificationRequest` | Filename + optional hints + tmdbId | New `TvIdentificationRequest` — same optional hints, adds episode hints |
| `MovieIdentificationSuccess` | originalFilename, suggestedFilename, match | New `TvIdentificationSuccess` — same shape |
| `MovieDisambiguationNeeded` | Candidates list | New `TvDisambiguationNeeded` — same pattern |
| `MovieIdentificationError` | Reason + detail | Shared domain error model — **reused directly** |

---

### Scoutarr.Core — confidence scoring

The confidence scoring formula (60% title similarity / 30% year match / 10% popularity) is implemented in `MovieSearchService`. UC-02 should apply the same formula in `TvShowSearchService` — extract the scoring logic into a shared `ConfidenceScorer` utility class to avoid duplication.

---

### Scoutarr.Core — ITmdbClient

After UC-01, `ITmdbClient` has:
- `SearchMovieAsync(string title, int? year, ...)` → list of movie candidates
- `GetMovieDetailsAsync(int movieId)` → enriched movie details (director, cast, collection, genres)

UC-02 adds:
- `SearchTvShowAsync(string title, int? year, ...)` → list of TV show candidates
- `GetTvShowDetailsAsync(int tvId)` → enriched TV show details (creator, cast, network, seasons, status)

The real implementation (`TmdbClient`) is extended. Tests continue to mock `ITmdbClient`.

---

### Scoutarr.Api — REST endpoints

After UC-01:
- `POST /identify/movie` — identify movie, returns success or disambiguation (422)
- `POST /rename/movie` — rename movie on disk

UC-02 adds:
- `POST /identify/episode` — same pattern, TV episode
- `POST /rename/episode` — same pattern, TV episode

The domain error → HTTP status code mapping is shared. UC-02 reuses it directly.

---

### Scoutarr.Mcp — MCP tools

After UC-01:
- `identify_movie` — returns success or enriched disambiguation result
- `rename_movie` — renames on disk

UC-02 adds:
- `identify_episode` — same pattern, TV episode; enriched with creator, cast, network, seasons
- `rename_episode` — same pattern

The MCP server system prompt is extended to cover TV show disambiguation behaviour.

---

### What UC-02 must build from scratch

- `ITvFilenameParser` / `TvFilenameParser` — extracts series title, season, episode number using heuristics (SxEy pattern, compact numeric with TMDB validation)
- `ITvShowSearchService` / `TvShowSearchService` — searches TMDB for TV shows, scores candidates
- `ITvFilenameFormatter` / `TvFilenameFormatter` — formats episode output filename
- `ITvIdentificationService` / `TvIdentificationService` — orchestrates the TV episode identification flow
- REST controllers and MCP tools for TV episodes
