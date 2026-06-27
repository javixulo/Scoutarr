# UC3-TASK-005 — REST API — identify_folder, rename_folder

**Requirements:** [requirements/interfaces.md](../requirements/interfaces.md) — "REST API", "Responses and notifications"
**Requirements:** [ARCHITECTURE.md](../ARCHITECTURE.md) — "REST API", HTTP status codes, RFC 9457
**Dependencies:** UC3-TASK-003 (IFolderRenameOrchestrator)

---

## Context

Two endpoints, both thin layers over the UC-03 orchestrator. They validate the incoming request, call Core, and translate the result into HTTP responses. No business logic lives here.

Unlike the episode endpoints in UC-02, there is no mandatory two-step flow here — series identification and episode processing happen inside the orchestrator in a single call. If disambiguation occurs, it is returned as a response and the caller retries with a confirmed `tmdbId`.

---

## Endpoints

### `POST /identify/folder`

Scans the folder, identifies the series, and returns suggested filenames for all episodes. Writes or updates the metadata file. Does not move any files.

**Request:**
```json
{
  "folderPath": "/downloads/breaking.bad.season1",
  "tmdbId": 1396
}
```

`tmdbId` is optional. If provided, series identification is skipped.

**Response 200 — all episodes identified:**
```json
{
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "episodes": [
    { "originalFilename": "breaking.bad.s01e01.mkv", "suggestedFilename": "Breaking Bad - S01E01 - Pilot.mkv" },
    { "originalFilename": "breaking.bad.s01e02.mkv", "suggestedFilename": "Breaking Bad - S01E02 - Cat's in the Bag.mkv" }
  ],
  "failures": []
}
```

**Response 200 — partial failures:**
```json
{
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "episodes": [ "..." ],
  "failures": [
    { "originalFilename": "extra.file.mkv", "reason": "Episode number not found", "detail": "..." }
  ]
}
```

**Response 300 — disambiguation needed:**
```json
{
  "candidates": [
    { "tmdbId": 1396, "title": "Breaking Bad", "year": 2008, "overview": "...", "originalLanguage": "en" }
  ]
}
```

---

### `POST /rename/folder`

Scans the folder, identifies the series, moves all episodes to their destination, and writes or updates the metadata file.

**Request:**
```json
{
  "folderPath": "/downloads/breaking.bad.season1",
  "tmdbId": 1396
}
```

`tmdbId` is optional. If provided, series identification is skipped.

**Response 200 — all episodes renamed:**
```json
{
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "successes": [
    { "originalFilename": "breaking.bad.s01e01.mkv", "newFilename": "Breaking Bad - S01E01 - Pilot.mkv", "destination": "/media/tv/Breaking Bad (2008)/Season 01/" }
  ],
  "failures": []
}
```

**Response 200 — partial failures:**
```json
{
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "successes": [ "..." ],
  "failures": [
    { "originalFilename": "extra.file.mkv", "reason": "Episode number not found", "detail": "..." }
  ]
}
```

**Response 300 — disambiguation needed:** same format as `/identify/folder`.

---

## HTTP status codes

| Situation | Status |
|---|---|
| Success (full or partial) | 200 |
| Disambiguation needed | 300 |
| Folder not found | 404 |
| Metadata file malformed | 422 |
| Unexpected error | 500 |

Partial failures (some episodes fail) return 200 — the process completed correctly and the failures are described in the response body.

---

## Notes for Black Widow

- Test `POST /identify/folder`: success, partial failures, disambiguation, folder not found, metadata malformed.
- Test `POST /rename/folder`: success, partial failures, disambiguation, folder not found, metadata malformed.
- Test that providing `tmdbId` in the request skips series identification in both endpoints.
- Test that partial failures return 200 with the correct body.
- E2E: real folder with real episode files, mocked TMDB, verify response and that no files are moved for identify.
- E2E: real folder with real episode files, mocked TMDB, verify that files are moved to the correct destination for rename.

## Notes for Tony Stark

- Implement `FolderController` in `Scoutarr.Api` with both endpoints.
- Reuse the same domain error → HTTP status mapping from UC-02, extended with the new UC-03 domain errors.
- Request and response DTOs in `Scoutarr.Api` — do not expose Core types directly.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: REST API — folder endpoints
  As an API caller
  I want to identify or rename all episodes in a folder via HTTP
  So that I can integrate Scoutarr into automated workflows

  Scenario: POST /identify/folder — all episodes identified
    Given a folder with 2 valid episode files
    And the series is identified successfully
    When POST /identify/folder is called
    Then the response is 200
    And suggested filenames are returned for both episodes
    And no files are moved

  Scenario: POST /identify/folder — partial failures
    Given a folder with 2 valid episode files and 1 unrecognised file
    When POST /identify/folder is called
    Then the response is 200
    And 2 suggested filenames are returned
    And 1 failure is returned with the appropriate reason

  Scenario: POST /identify/folder — disambiguation needed
    Given a folder whose series name matches multiple TMDB results
    When POST /identify/folder is called without tmdbId
    Then the response is 300
    And a list of candidates is returned

  Scenario: POST /identify/folder — tmdbId provided, identification skipped
    Given a folder with valid episode files
    When POST /identify/folder is called with a confirmed tmdbId
    Then series identification is skipped
    And suggested filenames are returned directly

  Scenario: POST /rename/folder — all episodes renamed
    Given a folder with 2 valid episode files
    And the series is identified successfully
    When POST /rename/folder is called
    Then the response is 200
    And both episodes appear in the successes list
    And both files are moved to their destination

  Scenario: POST /rename/folder — partial failures
    Given a folder with 2 valid episode files and 1 file that already exists at destination
    When POST /rename/folder is called
    Then the response is 200
    And 2 episodes appear in the successes list
    And 1 episode appears in the failures list with reason "File already exists at destination"

  Scenario: Folder not found
    Given a folder path that does not exist
    When POST /rename/folder is called
    Then the response is 404

  Scenario: Metadata file malformed
    Given a folder with a malformed metadata file
    When POST /rename/folder is called
    Then the response is 422
```

---

## Subtasks

- [ ] Define request and response DTOs in `Scoutarr.Api`
- [ ] Implement `FolderController` with `POST /identify/folder` and `POST /rename/folder`
- [ ] Extend the domain error → HTTP status mapping with the new UC-03 domain errors
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Black Widow writes E2E tests in red in `Scoutarr.E2E.Tests`
- [ ] Tony Stark implements the controller
- [ ] Hawkeye reviews
