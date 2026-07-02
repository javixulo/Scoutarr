# UC6-TASK-006 — REST API — identify and rename remap

**Requirements:**
- [requirements/interfaces.md](../requirements/interfaces.md) — "REST API", "Responses and notifications"
- [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
- [ARCHITECTURE.md](../ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457

**Dependencies:** UC6-TASK-003 (Identify), UC6-TASK-005 (Rename)

---

## Context

Two endpoints, both thin layers over the UC-06 Identify/Rename logic. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

---

## Endpoints

### `POST /identify/remap`
Computes the proposed mapping without touching the media file. Writes/updates the entry in `{Series Name} ({Year}).remapping.json`.

### `POST /rename/remap`
Applies an already-`resolved` mapping to disk. Internally calls identify first if no entry exists yet — if identification results in `conflict`, no rename occurs.

---

## Request shape (both endpoints)

```json
{
  "path": "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv",
  "destination": { "season": 1, "episode": 45 },
  "force": false
}
```

- `path`: path to the file, relative to `MEDIA_ROOT_TV`.
- `destination`: required, explicit season/episode as per UC-06 (no heuristics).
- `force`: optional, defaults to `false`. Only has effect when it matches the destination of an existing `conflict` entry for this file (see UC6-TASK-003).

---

## Response shapes

### Success — `resolved` (identify) or `applied` (rename) (`200`)

```json
{
  "originalPath": "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv",
  "proposedPath": "Season 01/Breaking Bad - S01E45 - The Beginning.mkv",
  "mode": "identify",
  "status": "resolved",
  "origin": { "season": 3, "episode": 10 },
  "destination": { "season": 1, "episode": 45 },
  "titleCheck": { "result": "match", "currentTitle": "The Beginning", "destinationTitle": "The Beginning" }
}
```

### Conflict, not confirmed (`409`)

Same shape as above, with `status: "conflict"` and both titles present in `titleCheck`, so the caller can decide whether to retry with `force: true`.

### Destination does not exist in TMDB (`422`)

RFC 9457, with the validation failure detail already defined in UC6-TASK-001.

### Other errors (`400`, `403`, `404`, `500`)

RFC 9457, consistent with the rest of the system's endpoints.

---

## HTTP status code mapping

| Scenario | Status |
|---|---|
| `resolved` (identify) / `applied` (rename) | `200` |
| Missing `path` or `destination` in the request | `400` |
| Insufficient write permissions (rename only) | `403` |
| File not found | `404` |
| `conflict` — title mismatch, requires `force` to confirm | `409` |
| Destination path already occupied on disk (rename only) | `409` |
| Destination season/episode does not exist in TMDB | `422` |
| TMDB unreachable, internal error | `500` |

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: REST API — identify and rename remap
  As a caller using the REST API
  I want to identify or rename a single-file remap via HTTP
  So that I receive a structured response I can act on

  Scenario: Identify returns a resolved mapping
    Given a POST request to /identify/remap with path "Season 03/Breaking Bad - S03E10 - The Beginning.mkv" and destination S01E45
    And the title cross-check is a match
    When the request is processed
    Then the response status is 200
    And the response contains mode "identify" and status "resolved"

  Scenario: Identify returns a conflict
    Given a POST request to /identify/remap with path "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv" and destination S01E45
    And the title cross-check is a mismatch
    When the request is processed
    Then the response status is 409
    And the response contains status "conflict" with both titles

  Scenario: Identify with force confirms a matching conflict
    Given a prior conflict entry exists for this file with destination S01E45
    And a POST request to /identify/remap with the same path, the same destination S01E45, and force true
    When the request is processed
    Then the response status is 200
    And the response contains status "resolved"

  Scenario: Destination does not exist in TMDB
    Given a POST request to /identify/remap with a destination season/episode that does not exist
    When the request is processed
    Then the response status is 422
    And the response follows RFC 9457 format

  Scenario: Missing destination returns 400
    Given a POST request to /identify/remap with no destination
    When the request is processed
    Then the response status is 400
    And the response follows RFC 9457 format

  Scenario: Rename applies a resolved entry
    Given a resolved entry exists for this file with destination S01E45
    And a POST request to /rename/remap with the same path and destination
    When the request is processed
    Then the response status is 200
    And the response contains mode "rename" and status "applied"

  Scenario: Rename rejects an unconfirmed conflict
    Given a conflict entry exists for this file with destination S01E45
    And a POST request to /rename/remap with the same path and destination, without force
    When the request is processed
    Then the response status is 409
    And no file is moved

  Scenario: Rename target already occupied on disk
    Given a resolved entry whose proposed destination file already exists on disk
    And a POST request to /rename/remap for that file
    When the request is processed
    Then the response status is 409
    And the response follows RFC 9457 format
    And no file is moved

  Scenario: File not found returns 404
    Given a POST request to /rename/remap with a path that does not exist on disk
    When the request is processed
    Then the response status is 404
    And the response follows RFC 9457 format
```

---

## Notes for Black Widow

- Use `WebApplicationFactory` for integration tests — no real filesystem, no real TMDB.
- Mock the UC-06 Identify/Rename logic at the HTTP layer.
- Test correct HTTP status code for each scenario above.
- Test that `mode` is `"identify"` for the identify endpoint and `"rename"` for the rename endpoint.
- Test that `force` is correctly ignored when it doesn't match the existing conflicted destination (per UC6-TASK-003), returning `409` unchanged.

## Notes for Tony Stark

- Implement controllers in `Scoutarr.Api`.
- Translate `status: resolved/applied` → `200`.
- Translate `status: conflict` → `409` with both titles present.
- Translate destination-validation failures (UC6-TASK-001) → `422`.
- Translate destination-already-occupied-on-disk failures (UC6-TASK-005) → `409`.
- Domain error mapping should reuse the existing RFC 9457 infrastructure already established for other endpoints.

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `RemapController` with `POST /identify/remap` and `POST /rename/remap`
- [ ] Implement domain error → HTTP status code mapping (including the new `409` for title conflicts)
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements controllers
- [ ] Hawkeye reviews
