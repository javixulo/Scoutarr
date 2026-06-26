# UC2-TASK-010 — MCP Server — identify series, identify episode, rename episode

**Requirements:** [requirements/interfaces.md](requirements/interfaces.md) — "MCP Server", "Responses and notifications"
**Requirements:** [identification.md](identification.md) — "Disambiguation flow"
**Requirements:** [ARCHITECTURE.md](ARCHITECTURE.md) — "MCP Server"
**Dependencies:** UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService)

---

## Context

Three MCP tools, all thin layers over Core services. The MCP layer is responsible for enriching results with additional TMDB data before returning them to the agent, and for providing `instructions` that guide the agent through disambiguation and multi-step flows.

Core does not enrich. Core does not know about MCP. The MCP layer calls Core, then optionally calls `ITmdbClient` directly to fetch enrichment data.

The TV episode flow spans two tools: `identify_series` first (with possible disambiguation), then `identify_episode` once the series is confirmed.

---

## Tools

### `identify_series`

Identifies the TV series from a dirty filename. Returns a confirmed match (with enriched metadata) or a disambiguation result for the agent to resolve with the user.

### `identify_episode`

Given a confirmed series `tvId`, identifies the episode from the filename and returns the suggested output filename.

### `rename_episode`

Identifies series and episode, then moves the file to the correct destination in `/media/tv`. Returns the destination path and any moved subtitle files.

---

## Enrichment — series identification

When `identify_series` returns a confirmed match, the MCP layer fetches additional data from TMDB and includes it in the result:

```
enrichment
{
    Creator:        string?                  // first credited creator
    Cast:           IReadOnlyList<string>    // top 5 cast names
    Network:        string?                  // first network or streaming platform
    Genres:         IReadOnlyList<string>
    Status:         string                   // "Returning Series", "Ended", "Canceled", etc. — as returned by TMDB
    Seasons:        IReadOnlyList<SeasonSummary>
}

SeasonSummary
{
    SeasonNumber:   int
    EpisodeCount:   int
}
```

Season 0 (specials) is included in the `Seasons` list if present, using `SeasonNumber: 0`.

When disambiguation is needed, the candidates list is also enriched with `overview` and `originalLanguage` per candidate — same as UC1.

---

## Tool result shapes

### `identify_series` — confirmed match

```json
{
  "tmdbId": 1396,
  "name": "Breaking Bad",
  "year": 2008,
  "confidence": 0.97,
  "creator": "Vince Gilligan",
  "cast": ["Bryan Cranston", "Aaron Paul", "Anna Gunn"],
  "network": "AMC",
  "genres": ["Drama", "Crime"],
  "status": "Ended",
  "seasons": [
    { "seasonNumber": 1, "episodeCount": 7 },
    { "seasonNumber": 2, "episodeCount": 13 }
  ],
  "instructions": "The series has been identified. Present the match to the user for confirmation. If confirmed, proceed with identify_episode using tvId 1396."
}
```

### `identify_series` — disambiguation needed

```json
{
  "disambiguationNeeded": true,
  "candidates": [
    {
      "index": 1,
      "tmdbId": 1396,
      "name": "Breaking Bad",
      "year": 2008,
      "overview": "...",
      "originalLanguage": "en"
    }
  ],
  "instructions": "No candidate exceeded the confidence threshold. Present the list to the user and ask them to select one. Then call identify_series again with the chosen candidateIndex."
}
```

### `identify_episode` — success

```json
{
  "originalFilename": "Breaking.Bad.S01E02.720p.mkv",
  "filename": "Breaking Bad - S01E02 - Cat's in the Bag.mkv",
  "seriesName": "Breaking Bad",
  "season": 1,
  "episode": 2,
  "episodeTitle": "Cat's in the Bag",
  "instructions": "The episode has been identified. Present the suggested filename to the user. If confirmed, call rename_episode to apply the rename."
}
```

### `rename_episode` — success

```json
{
  "originalFilename": "Breaking.Bad.S01E02.720p.mkv",
  "filename": "Breaking Bad - S01E02 - Cat's in the Bag.mkv",
  "destinationPath": "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv",
  "movedSubtitles": [
    {
      "originalFilename": "Breaking.Bad.S01E02.en.srt",
      "filename": "Breaking Bad - S01E02 - Cat's in the Bag.en.srt"
    }
  ],
  "instructions": "The episode has been renamed and moved. Inform the user of the new location and any moved subtitle files."
}
```

### Error result (`isError: true`)

```json
{
  "error": "Episode not found on TMDB: season 99, episode 99 does not exist for this series.",
  "instructions": "Inform the user of the error and suggest they verify the season and episode numbers."
}
```

---

## `instructions` field — agent guidance

The `instructions` field is always present in the tool result. It tells the agent what to do next in plain language. The agent must follow these instructions rather than decide autonomously.

Key scenarios covered by `instructions`:

- **Confirmed series match** — present to user for confirmation, then call `identify_episode` with `tvId`.
- **Series disambiguation** — present candidate list to user, ask them to pick, then retry `identify_series` with `candidateIndex`.
- **Confirmed episode** — present suggested filename to user, then call `rename_episode` if confirmed.
- **Rename success** — inform user of destination path and moved subtitles.
- **Error** — inform user and suggest next steps.

---

## Conversational flows (reference)

These flows illustrate the expected agent behaviour. They are not automated tests.

### Flow A — Clean identification, no disambiguation

```
User:   Rename Breaking.Bad.S01E02.720p.mkv
Agent:  [calls identify_series]
Agent:  I found Breaking Bad (2008) — an AMC crime drama created by Vince Gilligan,
        starring Bryan Cranston and Aaron Paul. It ran for 5 seasons and has ended.
        Is this correct?
User:   Yes
Agent:  [calls identify_episode with tvId 1396]
Agent:  Episode 2 of season 1 is "Cat's in the Bag". The new filename would be
        "Breaking Bad - S01E02 - Cat's in the Bag.mkv". Shall I rename it?
User:   Go ahead
Agent:  [calls rename_episode]
Agent:  Done. The file has been moved to
        /media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv
```

### Flow B — Series disambiguation

```
User:   Rename The.Office.S01E01.mkv
Agent:  [calls identify_series — disambiguation returned]
Agent:  I found several series called "The Office". Which one did you mean?
        1. The Office (2005) — NBC, US version, starring Steve Carell
        2. The Office (2001) — BBC, UK version, starring Ricky Gervais
        3. The Office (2019) — Indian remake
User:   The US one
Agent:  [calls identify_series with candidateIndex 1]
Agent:  Got it — The Office (2005), NBC sitcom, 9 seasons, ended.
        Is this correct?
User:   Yes
Agent:  [calls identify_episode with tvId 2316]
        ...
```

### Flow C — Anime with non-Latin title

```
User:   Rename Shingeki.no.Kyojin.S01E01.mkv
Agent:  [calls identify_series]
Agent:  I found Attack on Titan (2013) — a Japanese anime series about humanity
        living behind walls to survive giant humanoid creatures. 4 seasons, ended.
        Is this correct?
User:   Yes
Agent:  [calls identify_episode]
Agent:  Episode 1 of season 1 is "To You, in 2000 Years: The Fall of Shiganshina, Part 1".
        The new filename would be
        "Attack on Titan - S01E01 - To You, in 2000 Years: The Fall of Shiganshina, Part 1.mkv".
        Shall I rename it?
```

---

## Notes for Black Widow

- Mock `ITvShowIdentificationService`, `ITvEpisodeLookupService`, `ITvEpisodeMoveService`, and `ITmdbClient` for all tests.
- Test that confirmed series match triggers enrichment call to `ITmdbClient`.
- Test that disambiguation result does not trigger enrichment — candidates are enriched with `overview` and `originalLanguage` only, no extra TMDB call needed.
- Test that `instructions` field is always present in the tool result.
- Test that errors return `isError: true` with an `error` field and `instructions`.
- Test that season 0 appears in the `seasons` list when present in TMDB data.
- Test `rename_episode` returns `movedSubtitles` list (empty list when no subtitles, not null).

## Notes for Tony Stark

- Implement `identify_series`, `identify_episode`, and `rename_episode` tools in `Scoutarr.Mcp`.
- Extend the MCP server system prompt to cover the TV episode flow, disambiguation behaviour, and the two-step series → episode confirmation pattern.
- For enrichment, call `GetTvShowDetailsAsync` (already returns seasons and episode counts) and `GetTvShowEnrichmentAsync` (new method on `ITmdbClient` returning creator, cast, network, genres, status). Add `GetTvShowEnrichmentAsync` to `ITmdbClient` and implement in `TmdbClient`.
- `movedSubtitles` must always be an array in the response — use an empty array when no subtitles were moved.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: MCP Server — identify series, identify episode, rename episode
  As an AI agent
  I want to identify TV series and episodes and rename episode files via MCP tools
  So that I can guide the user through the rename flow conversationally

  # ─────────────────────────────────────────
  # IDENTIFY SERIES — confirmed match
  # ─────────────────────────────────────────

  Scenario: Series identified above threshold — enriched result returned
    Given identify_series is called with filename "Breaking.Bad.S01E02.mkv"
    And Core returns a confirmed match for "Breaking Bad" (2008) with tmdbId 1396
    And TMDB enrichment returns creator "Vince Gilligan", network "AMC", status "Ended"
    When the tool result is returned
    Then the result contains tmdbId 1396, name "Breaking Bad", year 2008
    And the result contains creator "Vince Gilligan" and network "AMC"
    And the result contains status "Ended"
    And the result contains a seasons list
    And the result contains an instructions field

  Scenario: Season 0 appears in seasons list when present
    Given identify_series is called with filename "Breaking.Bad.S01E02.mkv"
    And TMDB returns season 0 with 3 specials
    When the tool result is returned
    Then the seasons list contains an entry with seasonNumber 0 and episodeCount 3

  # ─────────────────────────────────────────
  # IDENTIFY SERIES — disambiguation
  # ─────────────────────────────────────────

  Scenario: Disambiguation needed — candidates returned without enrichment call
    Given identify_series is called with filename "The.Office.S01E01.mkv"
    And Core returns a disambiguation result with 3 candidates
    When the tool result is returned
    Then the result contains disambiguationNeeded true
    And the result contains 3 candidates each with index, tmdbId, name, year, overview, originalLanguage
    And GetTvShowEnrichmentAsync is never called
    And the result contains an instructions field

  Scenario: candidateIndex provided — Core uses selected candidate
    Given identify_series is called with filename "The.Office.S01E01.mkv" and candidateIndex 1
    And Core returns a confirmed match for the first candidate
    When the tool result is returned
    Then the result contains a confirmed match with the selected series

  # ─────────────────────────────────────────
  # IDENTIFY EPISODE
  # ─────────────────────────────────────────

  Scenario: Episode identified — suggested filename returned
    Given identify_episode is called with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And Core returns episode title "Cat's in the Bag" for S01E02
    When the tool result is returned
    Then the result contains filename "Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the result contains season 1 and episode 2
    And the result contains an instructions field

  Scenario: No episode pattern in filename — isError true returned
    Given identify_episode is called with filename "Breaking.Bad.mkv" and tvId 1396
    And Core returns a no episode pattern found error
    When the tool result is returned
    Then the result has isError true
    And the result contains an instructions field

  Scenario: Episode not found on TMDB — isError true returned
    Given identify_episode is called with filename "Breaking.Bad.S99E99.mkv" and tvId 1396
    And Core returns an episode not found error
    When the tool result is returned
    Then the result has isError true
    And the result contains an instructions field

  # ─────────────────────────────────────────
  # RENAME EPISODE
  # ─────────────────────────────────────────

  Scenario: Episode renamed and moved — destination path returned
    Given rename_episode is called with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And Core identifies and moves the episode successfully
    When the tool result is returned
    Then the result contains destinationPath "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the result contains an instructions field

  Scenario: Moved subtitles included in result
    Given rename_episode is called with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service reports one moved subtitle
    When the tool result is returned
    Then the result contains movedSubtitles with one entry

  Scenario: No subtitles moved — movedSubtitles is empty array, not null
    Given rename_episode is called with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service reports no moved subtitles
    When the tool result is returned
    Then the result contains movedSubtitles as an empty array

  Scenario: File already exists at destination — isError true returned
    Given rename_episode is called with filename "Breaking.Bad.S01E02.mkv" and tvId 1396
    And the move service returns a file already exists error
    When the tool result is returned
    Then the result has isError true
    And the result contains an instructions field

  # ─────────────────────────────────────────
  # INSTRUCTIONS FIELD
  # ─────────────────────────────────────────

  Scenario: instructions field is always present regardless of outcome
    Given any tool call to identify_series, identify_episode, or rename_episode
    When the tool result is returned
    Then the result always contains a non-empty instructions field
```

---

## End-to-end tests (`Scoutarr.E2E.Tests`)

E2E tests run against the built Docker container using a real TMDB API key and temporary directories mounted as `/input/tv` and `/media/tv`. They cover the full MCP stack — no mocks. An MCP test client connects to the container over HTTP/SSE.

**Setup and teardown:** before the E2E test session starts, fresh temporary directories are created on the host and mounted into the container. After the session ends, those directories are deleted in full. Each individual test uses uniquely named files (e.g. with a UUID suffix) to avoid collisions with other tests running in the same session. No test depends on the state left by another.

```gherkin
Feature: E2E — MCP TV episode flow
  As an MCP client connected to the live container
  I want to identify and rename a TV episode end-to-end via MCP tools
  So that the full stack is verified including TMDB, filesystem, and the agent flow

  Scenario: Full flow — identify series, identify episode, rename episode via MCP
    Given the container is running
    And a file "Breaking.Bad.S01E02.720p.mkv" exists in /input/tv
    When identify_series is called with filename "Breaking.Bad.S01E02.720p.mkv"
    Then the tool result contains a confirmed match for "Breaking Bad" with enriched metadata
    And the result contains creator, network, status, and seasons list
    When identify_episode is called with the confirmed tvId and the same filename
    Then the tool result contains filename "Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    When rename_episode is called with the confirmed tvId and the same filename
    Then the tool result contains the destination path
    And the file exists at "/media/tv/Breaking Bad (2008)/Season 01/Breaking Bad - S01E02 - Cat's in the Bag.mkv"
    And the operation is logged to /config/logs/scoutarr.log

  Scenario: Series disambiguation flow via MCP
    Given the container is running
    And a file "The.Office.S01E01.mkv" exists in /input/tv
    When identify_series is called with filename "The.Office.S01E01.mkv"
    Then the tool result contains disambiguationNeeded true
    And the result contains multiple candidates with index, name, year, and overview
    When identify_series is called again with the same filename and candidateIndex 1
    Then the tool result contains a confirmed match

  Scenario: Error returns isError true from live container
    Given the container is running
    And no file "notexist.S01E02.mkv" exists in /input/tv
    When rename_episode is called with that filename and a valid tvId
    Then the tool result has isError true
    And the result contains a non-empty error field
```

---

## Subtasks

- [ ] Add `GetTvShowEnrichmentAsync` to `ITmdbClient` and implement in `TmdbClient`
- [ ] Implement `identify_series` tool in `Scoutarr.Mcp`
- [ ] Implement `identify_episode` tool in `Scoutarr.Mcp`
- [ ] Implement `rename_episode` tool in `Scoutarr.Mcp`
- [ ] Extend MCP server system prompt to cover TV episode flow
- [ ] Black Widow writes tests in red (all unit/integration scenarios above)
- [ ] Black Widow writes E2E tests in red (`Scoutarr.E2E.Tests`)
- [ ] Tony Stark implements all three tools
- [ ] Hawkeye reviews
