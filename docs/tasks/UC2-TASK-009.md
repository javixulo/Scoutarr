# UC2-TASK-009 — REST API — identify series, identify episode, rename episode

**Requirements:** [requirements/interfaces.md](../requirements/interfaces.md) — "REST API", "Responses and notifications"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow", "Optional parameters"
**Requirements:** [ARCHITECTURE.md](../ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457
**Dependencies:** UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-005 (ITvEpisodeNumberParser), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService)

---

## Context

Three endpoints, all thin layers over Core services. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

The TV episode flow is split into two explicit steps — series identification and episode identification — because the caller must confirm the series before proceeding to the episode.

---

## Endpoints

### `POST /identify/series`
Returns a confirmed series match or a disambiguation result.

### `POST /identify/episode`
Given a confirmed series `tmdbId`, identifies the episode.

### `POST /rename/episode`
Identifies series and episode, then moves the file to `/media/tv`.

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `TvSeriesController` with `POST /identify/series`
- [ ] Implement `TvEpisodeController` with `POST /identify/episode` and `POST /rename/episode`
- [ ] Extend domain error → HTTP status code mapping with TV-specific error reasons
- [ ] Black Widow writes tests in red (all unit/integration scenarios)
- [ ] Black Widow writes E2E tests in red (`Scoutarr.E2E.Tests`)
- [ ] Tony Stark implements controllers
- [ ] Hawkeye reviews
