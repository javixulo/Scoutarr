# TASK-005 — REST API — identify and rename movie

**Requirements:**
- [requirements/interfaces.md](../requirements/interfaces.md) — "REST API", "Responses and notifications"
- [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow", "Optional parameters"
- [ARCHITECTURE.md](../ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457

**Dependencies:** TASK-004 (Core orchestration)

---

## Context

Two endpoints, both thin layers over `IMovieIdentificationService`. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

---

## Endpoints

### `POST /identify/movie`
Returns a suggested filename without touching the filesystem.

### `POST /rename/movie`
Applies the rename to disk. Internally calls identify first — if identification fails, no rename occurs. Requires the file path to be accessible inside the container.

---

## Request shape (both endpoints)

```json
{
  "filename": "The.Batman.2022.1080p.mkv",
  "candidateIndex": null,
  "year": null,
  "originalLanguage": null,
  "genre": null
}
```

- `filename` — required.
- `candidateIndex` — 1-based index from a previous disambiguation response. Optional.
- `year`, `originalLanguage`, `genre` — optional hints passed to Core.

---

## Response shapes

### Success (`200`)

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

For `rename` mode, `mode` is `"rename"` and the rename has been applied on disk.

### Disambiguation needed (`422`)

```json
{
  "type": "https://scoutarr.io/errors/disambiguation-needed",
  "title": "No match above confidence threshold",
  "status": 422,
  "detail": "No TMDB candidate exceeded the confidence threshold. Select a candidate and retry with candidateIndex.",
  "instance": "/identify/movie",
  "candidates": [
    {
      "index": 1,
      "tmdbId": 414906,
      "title": "The Batman",
      "year": 2022,
      "overview": "...",
      "originalLanguage": "en"
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
  "detail": "The file '/media/movies/batman.mkv' does not exist or is not accessible.",
  "instance": "/rename/movie"
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
| TMDB unreachable, internal error | `500` |

---

## Notes for Black Widow

- Use `WebApplicationFactory` for integration tests — no real filesystem, no real TMDB.
- Mock `IMovieIdentificationService` at the HTTP layer — do not re-test Core logic here.
- Test request validation (missing filename, invalid candidateIndex type).
- Test correct HTTP status code for each scenario.
- Test correct response shape for success, disambiguation, and each error type.
- Test that `mode` field is `"identify"` for the identify endpoint and `"rename"` for the rename endpoint.

## Notes for Tony Stark

- Implement controllers in `Scoutarr.Api`.
- Translate `MovieIdentificationSuccess` → `200`.
- Translate `MovieDisambiguationNeeded` → `422` with candidates list (1-based index added here).
- Translate `MovieIdentificationError` → appropriate HTTP status via a domain error → status code mapping.
- For rename: after a successful identification, apply the rename on disk before returning. Filesystem errors (not found, permissions, conflict) are caught here and returned as RFC 9457 errors.
- The `candidates` list in the `422` response must include the 1-based `index` field so the caller can pass it back as `candidateIndex`.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: REST API — identify and rename movie
  As a caller using the REST API
  I want to identify or rename a movie file via HTTP
  So that I receive a structured response I can act on

  Scenario: Identify returns suggested filename and match
    Given a POST request to /identify/movie with filename "The.Batman.2022.1080p.mkv"
    And Core returns a successful identification with suggested filename "The Batman (2022).mkv"
    When the request is processed
    Then the response status is 200
    And the response contains originalFilename "The.Batman.2022.1080p.mkv"
    And the response contains filename "The Batman (2022).mkv"
    And the response contains mode "identify"
    And the response contains match with title "The Batman" and year 2022

  Scenario: Low confidence returns 422 with candidate list
    Given a POST request to /identify/movie with filename "Batman.mkv"
    And Core returns a disambiguation result with 7 candidates
    When the request is processed
    Then the response status is 422
    And the response contains 7 candidates
    And each candidate has a 1-based index field
    And the first candidate has index 1

  Scenario: Missing filename returns 400
    Given a POST request to /identify/movie with no filename
    When the request is processed
    Then the response status is 400
    And the response follows RFC 9457 format

  Scenario: Rename applies the rename on disk and returns success
    Given a POST request to /rename/movie with filename "The.Batman.2022.1080p.mkv"
    And Core returns a successful identification with suggested filename "The Batman (2022).mkv"
    And the filesystem rename succeeds
    When the request is processed
    Then the response status is 200
    And the response contains mode "rename"
    And the response contains filename "The Batman (2022).mkv"

  Scenario: File not found returns 404
    Given a POST request to /rename/movie with filename "notexist.mkv"
    And the file does not exist on disk
    When the request is processed
    Then the response status is 404
    And the response follows RFC 9457 format
```

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `MovieController` with `POST /identify/movie` and `POST /rename/movie`
- [ ] Implement domain error → HTTP status code mapping
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements controllers and filesystem rename logic
- [ ] Hawkeye reviews
