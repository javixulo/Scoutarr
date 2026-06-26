# UC2-TASK-009 — REST API — identify series, identify episode, rename episode

**Requirements:** [requirements/interfaces.md](requirements/interfaces.md) — "REST API", "Responses and notifications"
**Requirements:** [identification.md](identification.md) — "Disambiguation flow", "Optional parameters"
**Requirements:** [ARCHITECTURE.md](ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457
**Dependencies:** UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-005 (ITvEpisodeNumberParser), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService)

---

## Context

Three endpoints, all thin layers over Core services. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

The TV episode flow is split into two explicit steps — series identification and episode identification — because the caller must confirm the series before proceeding to the episode. This mirrors the disambiguation flow from UC1 but adds an intermediate step.

---

## Endpoints

### `POST /identify/series`

Returns a confirmed series match or a disambiguation result. Does not touch the filesystem.

### `POST /identify/episode`

Given a confirmed series `tmdbId`, identifies the episode from the filename. Returns a suggested output filename. Does not touch the filesystem.

### `POST /rename/episode`

Identifies the series and episode, then moves the file to the correct destination in `/media/tv`. Requires the file path to be accessible inside the container at `/input/tv`.

---

## Request shapes

### `/identify/series`

```json
{
  "filename": "Breaking.Bad.S01E02.720p.mkv",
  "candidateIndex": null,
  "year": null,
  "originalLanguage": null
}
```

- `filename` — required.
- `candidateIndex` — 1-based index from a previous disambiguation response. Optional.
- `year`, `originalLanguage` — optional hints passed to Core.

### `/identify/episode`

```json
{
  "filename": "Breaking.Bad.S01E02.720p.mkv",
  "tvId": 1396
}
```

- `filename` — required.
- `tvId` — required. Must be a confirmed TMDB series ID from a previous `/identify/series` call.

### `/rename/episode`

```json
{
  "filename": "Breaking.Bad.S01E02.720p.mkv",
  "tvId": 1396
}
```

- `filename` — required.
- `tvId` — required. Must be a confirmed TMDB series ID from a previous `/identify/series` call. Disambiguation must be resolved before calling this endpoint — the REST API has no mechanism to present candidates and await user input mid-request.

---

## Response shapes

### Series identification success (`200`)

```json
{
  "mode": "identify",
  "match": {
    "tmdbId": 1396,
    "name": "Breaking Bad",
    "year": 2008,
    "confidence": 0.97
  }
}
```

### Series disambiguation needed (`422`)

```json
{
  "type": "https://scoutarr.io/errors/disambiguation-needed",
  "title": "No match above confidence threshold",
  "status": 422,
  "detail": "No TMDB candidate exceeded the confidence threshold. Select a candidate and retry with candidateIndex.",
  "instance": "/identify/series",
  "candidates": [
    {
      "index": 1,
      "tmdbId": 1396,
      "name": "Breaking Bad",
      "year": 2008,
      "overview": "...",
      "originalLanguage": "en"
    }
  ]
}
```

### Episode identification success (`200`)

```json
{
  "mode": "identify",
  "originalFilename": "Breaking.Bad.S01E02.720p.mkv",
  "filename": "Breaking Bad - S01E02 - Cat's in the Bag.mkv",
  "match": {
    "tmdbId": 1396,
    "seriesName": "Breaking Bad",
    "season": 1,
    "episode": 2,
    "episodeTitle": "Cat's in the Bag"
  }
}
```

### Rename success (`200`)

Same shape as episode identification success, with `mode` set to `"rename"`. Additionally includes:

```json
{
  "destinationPath": "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv",
  "movedSubtitles": [
    {
      "originalFilename": "Breaking.Bad.S01E02.en.srt",
      "filename": "Breaking Bad - S01E02 - Cat's in the Bag.en.srt"
    }
  ]
}
```

### Error (`400`, `403`, `404`, `409`, `500`) — RFC 9457

```json
{
  "type": "https://scoutarr.io/errors/file-not-found",
  "title": "File not found",
  "status": 404,
  "detail": "The file '/input/tv/Breaking.Bad.S01E02.720p.mkv' does not exist or is not accessible.",
  "instance": "/rename/episode"
}
```

---

## HTTP status code mapping

| Scenario | Status |
|---|---|
| Success | `200` |
| Missing or invalid request field | `400` |
| Insufficient write permissions (rename only) | `403` |
| File not found (rename only) | `404` |
| Target filename already exists (rename only) | `409` |
| No match above confidence threshold | `422` |
| No episode pattern found in filename | `422` |
| Episode not found on TMDB | `422` |
| TMDB unreachable, internal error | `500` |

---

## Notes for Black Widow

- Use `WebApplicationFactory` for integration tests — no real filesystem, no real TMDB.
- Mock `ITvShowIdentificationService`, `ITvEpisodeLookupService`, and `ITvEpisodeMoveService` at the HTTP layer — do not re-test Core logic here.
- Test request validation: missing filename, missing tvId on `/identify/episode`, invalid candidateIndex type.
- Test correct HTTP status code for each scenario.
- Test correct response shape for success, disambiguation, and each error type.
- Test that `mode` is `"identify"` for identify endpoints and `"rename"` for the rename endpoint.
- Test that the `movedSubtitles` list is present in the rename success response.

## Notes for Tony Stark

- Implement controllers in `Scoutarr.Api`.
- Translate `TvShowIdentificationSuccess` → `200`.
- Translate `TvShowDisambiguationNeeded` → `422` with candidates list (1-based index added here, same as movie).
- Translate `TvEpisodeLookupSuccess` → `200` for `/identify/episode`.
- Translate `TvEpisodeMoveSuccess` → `200` for `/rename/episode`, including `destinationPath` and `movedSubtitles`.
- Reuse the existing domain error → HTTP status code mapping from UC1. Extend it with TV-specific error reasons.
- For `/rename/episode`: `tvId` is required — validate at the controller level and return `400` if absent. Call episode lookup then move service directly, no series identification step needed.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: REST API — identify series, identify episode, rename episode
  As a caller using the REST API
  I want to identify a TV series and episode or rename an episode file via HTTP
  So that I receive a structured response I can act on

  # ─────────────────────────────────────────
  # IDENTIFY SERIES — happy path
  # ─────────────────────────────────────────

  Scenario: Series identified above threshold — success returned
    Given a POST request to /identify/series with filename "Breaking.Bad.S01E02.mkv"
    And Core returns a successful series identification for "Breaking Bad" (2008)
    When the request is processed
    Then the response status is 200
    And the response contains match with name "Breaking Bad" and year 2008

  Scenario: candidateIndex provided — Core bypasses search
    Given a POST request to /identify/series with filename "The.Office.S01E01.mkv" and candidateIndex 2
    And Core returns a successful identification using the second candidate
    When the request is processed
    Then the response status is 200

  # ─────────────────────────────────────────
  # IDENTIFY SERIES — disambiguation
  # ─────────────────────────────────────────

  Scenario: Low confidence returns 422 with candidate list
    Given a POST request to /identify/series with filename "The.Office.S01E01.mkv"
    And Core returns a disambiguation result with 3 candidates
    When the request is processed
    Then the response status is 422
    And the response contains 3 candidates each with a 1-based index field

  Scenario: candidateIndex out of range returns 400
    Given a POST request to /identify/series with filename "The.Office.S01E01.mkv" and candidateIndex 99
    And Core returns a candidate index out of range error
    When the request is processed
    Then the response status is 400
    And the response follows RFC 9457 format

  # ─────────────────────────────────────────
  # IDENTIFY SERIES — errors
  # ─────────────────────────────────────────

  Scenario: Missing filename returns 400
    Given a POST request to /identify/series with no filename
    When the request is processed
    Then the response status is 400
    And the response follows RFC 9457 format

  Scenario: No TMDB results returns 422
    Given a POST request to /identify/series with filename "Zxqfnmvp.S01E01.mkv"
    And Core returns a no TMDB results error
    When the request is processed
    Then the response status is 422
    And the response follows RFC 9457 format

  Scenario: TMDB unreachable returns 500
    Given a POST request to /identify/series with filename "Breaking.Bad.S01E02.mkv"
    And Core returns a TMDB unreachable error
    When the request is processed
    Then the response status is 500
    And the response follows RFC 9457 format

  # ─────────────────────────────────────────
  # IDENTIFY EPISODE — happy path
  # ─────────────────────────────────────────

  Scenario: Episode identified — success returned
    Given a POST request to /identify/episode with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And Core returns episode title "Cat's in the Bag" for S01E02
    When the request is processed
    Then the response status is 200
    And the response contains originalFilename "Breaking.Bad.S01E02.mkv"
    And the response contains filename "Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the response contains mode "identify"

  # ─────────────────────────────────────────
  # IDENTIFY EPISODE — errors
  # ─────────────────────────────────────────

  Scenario: Missing tvId returns 400
    Given a POST request to /identify/episode with filename "Breaking.Bad.S01E02.mkv" and no tvId
    When the request is processed
    Then the response status is 400
    And the response follows RFC 9457 format

  Scenario: No episode pattern in filename returns 422
    Given a POST request to /identify/episode with filename "Breaking.Bad.mkv" and tvId 1396
    And Core returns a no episode pattern found error
    When the request is processed
    Then the response status is 422
    And the response follows RFC 9457 format

  Scenario: Episode not found on TMDB returns 422
    Given a POST request to /identify/episode with filename "Breaking.Bad.S99E99.mkv" and tvId 1396
    And Core returns an episode not found error
    When the request is processed
    Then the response status is 422
    And the response follows RFC 9457 format

  # ─────────────────────────────────────────
  # RENAME EPISODE — happy path
  # ─────────────────────────────────────────

  Scenario: Episode renamed and moved — success returned
    Given a POST request to /rename/episode with filename "Breaking.Bad.S01E02.720p.mkv" and tvId 1396
    And Core identifies the episode as "Breaking Bad - S01E02 - Cat's in the Bag"
    And the move service succeeds
    When the request is processed
    Then the response status is 200
    And the response contains mode "rename"
    And the response contains destinationPath "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"

  Scenario: Moved subtitles are included in the response
    Given a POST request to /rename/episode with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service reports one moved subtitle "Breaking Bad - S01E02 - Cat's in the Bag.en.srt"
    When the request is processed
    Then the response contains movedSubtitles with one entry

  # ─────────────────────────────────────────
  # RENAME EPISODE — filesystem errors
  # ─────────────────────────────────────────

  Scenario: Source file not found returns 404
    Given a POST request to /rename/episode with filename "notexist.S01E02.mkv" and tvId 1396
    And the move service returns a source file not found error
    When the request is processed
    Then the response status is 404
    And the response follows RFC 9457 format

  Scenario: Insufficient write permissions returns 403
    Given a POST request to /rename/episode with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service returns an insufficient permissions error
    When the request is processed
    Then the response status is 403
    And the response follows RFC 9457 format

  Scenario: File already exists at destination returns 409
    Given a POST request to /rename/episode with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service returns a file already exists error
    When the request is processed
    Then the response status is 409
    And the response follows RFC 9457 format
```

---

## End-to-end tests (`Scoutarr.E2E.Tests`)

E2E tests run against the built Docker container using a real TMDB API key and temporary directories mounted as `/input/tv` and `/media/tv`. They cover the full stack — no mocks.

**Setup and teardown:** before the E2E test session starts, fresh temporary directories are created on the host and mounted into the container. After the session ends, those directories are deleted in full. Each individual test uses uniquely named files (e.g. with a UUID suffix) to avoid collisions with other tests running in the same session. No test depends on the state left by another.

```gherkin
Feature: E2E — REST API TV episode flow
  As a caller using the REST API against the live container
  I want to identify and rename a TV episode end-to-end
  So that the full stack is verified including TMDB, filesystem, and logging

  Scenario: Full flow — identify series, identify episode, rename episode via REST
    Given the container is running
    And a file "Breaking.Bad.S01E02.720p.mkv" exists in /input/tv
    When POST /identify/series is called with filename "Breaking.Bad.S01E02.720p.mkv"
    Then the response status is 200
    And the response contains a confirmed match for "Breaking Bad"
    When POST /identify/episode is called with the confirmed tvId and the same filename
    Then the response status is 200
    And the response contains filename "Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    When POST /rename/episode is called with the confirmed tvId and the same filename
    Then the response status is 200
    And the file exists at "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the original file no longer exists at /input/tv
    And the operation is logged to /config/logs/scoutarr.log

  Scenario: Rename episode with subtitle file — subtitle moved and renamed
    Given a file "Breaking.Bad.S01E02.mkv" and "Breaking.Bad.S01E02.en.srt" exist in /input/tv
    When the full rename flow is completed via REST
    Then the subtitle exists at "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.en.srt"

  Scenario: File not found returns 404 from live container
    Given no file "notexist.S01E02.mkv" exists in /input/tv
    When POST /rename/episode is called with that filename and a valid tvId
    Then the response status is 404
    And the response follows RFC 9457 format

  Scenario: Both REST and MCP interfaces are reachable on container startup
    Given the container has started
    When the health endpoints for REST and MCP are checked
    Then both return a successful response
```

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `TvSeriesController` with `POST /identify/series`
- [ ] Implement `TvEpisodeController` with `POST /identify/episode` and `POST /rename/episode`
- [ ] Extend domain error → HTTP status code mapping with TV-specific error reasons
- [ ] Black Widow writes tests in red (all unit/integration scenarios above)
- [ ] Black Widow writes E2E tests in red (`Scoutarr.E2E.Tests`)
- [ ] Tony Stark implements controllers
- [ ] Hawkeye reviews
