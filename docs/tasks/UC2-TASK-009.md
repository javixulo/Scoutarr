# UC2-TASK-009 — REST API — identify series, identify episode, rename episode

**Requirements:** [requirements/interfaces.md](../requirements/interfaces.md) — "REST API", "Responses and notifications"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow", "Optional parameters"
**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Series state"
**Requirements:** [ARCHITECTURE.md](../ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457
**Dependencies:** UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-005 (ITvEpisodeNumberParser), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService), UC3-TASK-002 (ISeriesMetadataService — write capability only; see note below)

---

## Context

Three endpoints, all thin layers over Core services. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

The TV episode flow is split into two explicit steps — series identification and episode identification — because the caller must confirm the series before proceeding to the episode.

After a successful series identification (`identify/series`, `identify/episode`, or `rename/episode`), the orchestrator writes the series metadata file to the root series folder via `ISeriesMetadataService`, per [file-handling.md](../requirements/file-handling.md) — "Series state". This applies even for a single episode: a single-episode operation creates or updates the same metadata file that a full series-folder pass (UC-03) would.

> **Note on sequencing:** `ISeriesMetadataService` is fully specified in UC3-TASK-002, including read, refresh, and validation logic that only make sense once folder-level passes exist. UC-02 only needs the **write** path — given a confirmed `SeriesMetadata`, persist it as JSON to the series root folder. This task (and UC2-TASK-010) should implement or stub the minimal write capability needed for UC-02, and UC3-TASK-002 then extends it with read/refresh/validation for UC-03. If UC3-TASK-002 is implemented first, UC-02 simply reuses the existing service instead.

---

## Endpoints

### `POST /identify/series`
Returns a confirmed series match or a disambiguation result.

### `POST /identify/episode`
Given a confirmed series `tmdbId`, identifies the episode. On a confirmed match, writes the series metadata file to the root series folder (Identify mode does not touch any other part of the filesystem).

### `POST /rename/episode`
Identifies series and episode, then moves the file to `/media/tv`. On success, writes (or updates) the series metadata file to the root series folder.

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `TvSeriesController` with `POST /identify/series`
- [ ] Implement `TvEpisodeController` with `POST /identify/episode` and `POST /rename/episode`
- [ ] Wire `ISeriesMetadataService` write into `identify/episode` and `rename/episode` after a successful series match
- [ ] Extend domain error → HTTP status code mapping with TV-specific error reasons
- [ ] Black Widow writes tests in red (all unit/integration scenarios, including metadata file write)
- [ ] Black Widow writes E2E tests in red (`Scoutarr.E2E.Tests`)
- [ ] Tony Stark implements controllers
- [ ] Hawkeye reviews
