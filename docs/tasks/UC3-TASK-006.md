# UC3-TASK-006 — MCP Server — identify_folder, rename_folder

**Requirements:** [requirements/interfaces.md](../requirements/interfaces.md) — "MCP Server", "Responses and notifications"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow"
**Requirements:** [ARCHITECTURE.md](../ARCHITECTURE.md) — "MCP Server"
**Dependencies:** UC3-TASK-003 (IFolderRenameOrchestrator)

---

## Context

Two MCP tools, both thin layers over the UC-03 orchestrator. The MCP layer enriches results with additional TMDB data and provides `instructions` that guide the agent through disambiguation and multi-step flows.

Unlike the REST endpoints, the MCP layer is the natural home for interactive disambiguation — the agent asks the user focused questions one at a time using the enriched candidate data, and retries with the confirmed `tmdbId` once the user has decided.

The folder flow is simpler than the episode flow in UC-02: series identification and episode processing happen in a single tool call. The only multi-step interaction is disambiguation.

---

## Tools

### `identify_folder`
Scans the folder, identifies the series, and returns suggested filenames for all episodes. Writes or updates the metadata file. Does not move any files.

### `rename_folder`
Scans the folder, identifies the series, moves all episodes to their destination, and writes or updates the metadata file.

---

## Tool input schema (both tools)

```json
{
  "folderPath": "/downloads/breaking.bad.season1",
  "tmdbId": null
}
```

`tmdbId` is optional. If provided, series identification is skipped entirely.

---

## MCP server system prompt (folder section)

```
When processing a folder and series disambiguation is needed:
- Never present a numbered list of candidates to the user.
- Ask one focused, natural question at a time using the enriched candidate data.
- Use creator, cast, network, and status to phrase contextual questions.
- Examples: "Is this the AMC series created by Vince Gilligan, or the BBC documentary of the same name?",
  "Does the folder contain episodes starring Bryan Cranston?"
- If the user explicitly cancels, confirm the cancellation and do not modify any files.

When the series is confirmed and identify_folder was called:
- Present the series match naturally: title, year, network, and season count.
- Show the list of suggested filenames grouped by season.
- Highlight any failures clearly and explain what went wrong for each.
- Offer to call rename_folder with the confirmed tmdbId if the user approves.

When rename_folder succeeds:
- Report how many episodes were moved successfully.
- List any failures clearly, explaining what went wrong for each.
- If all episodes failed, warn the user prominently.

When an error occurs:
- Explain what went wrong in plain language and suggest what the user can do next.
- For metadata file malformed: tell the user the metadata file at the folder path is corrupted and needs to be deleted or fixed manually before retrying.
```

---

## Enrichment — confirmed series match

When a series is confirmed (either via identification or provided `tmdbId`), the MCP layer fetches:
- Creator, Cast (top 5), Network, Genres, Status, Seasons (with episode counts)

Season 0 (specials) is included in the `Seasons` list if present.

When disambiguation is needed, candidates are enriched with `overview`, `originalLanguage`, `firstAirDate`, and `network` — one extra TMDB call per candidate, max 10 candidates.

---

## Response shapes

### Confirmed match — identify_folder

```json
{
  "disambiguationNeeded": false,
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "network": "AMC",
  "status": "Ended",
  "genres": ["Crime", "Drama", "Thriller"],
  "creator": "Vince Gilligan",
  "topCast": ["Bryan Cranston", "Aaron Paul", "Anna Gunn", "Dean Norris", "Betsy Brandt"],
  "seasons": [
    { "seasonNumber": 1, "episodeCount": 7 }
  ],
  "episodes": [
    { "originalFilename": "breaking.bad.s01e01.mkv", "suggestedFilename": "Breaking Bad - S01E01 - Pilot.mkv" }
  ],
  "failures": [],
  "instructions": "Present the series match and suggested filenames to the user. Offer to call rename_folder with tmdbId if confirmed."
}
```

### Confirmed match — rename_folder

```json
{
  "disambiguationNeeded": false,
  "seriesTitle": "Breaking Bad",
  "tmdbId": 1396,
  "network": "AMC",
  "status": "Ended",
  "genres": ["Crime", "Drama", "Thriller"],
  "creator": "Vince Gilligan",
  "topCast": ["Bryan Cranston", "Aaron Paul", "Anna Gunn", "Dean Norris", "Betsy Brandt"],
  "successes": [
    { "originalFilename": "breaking.bad.s01e01.mkv", "newFilename": "Breaking Bad - S01E01 - Pilot.mkv", "destination": "/media/tv/Breaking Bad (2008)/Season 01/" }
  ],
  "failures": [],
  "instructions": "Report the rename summary to the user. All episodes were moved successfully."
}
```

### Disambiguation needed

```json
{
  "disambiguationNeeded": true,
  "instructions": "Ask the user one focused, natural question to narrow down the candidates. Never show a numbered list.",
  "candidates": [
    {
      "tmdbId": 1396,
      "title": "Breaking Bad",
      "year": 2008,
      "overview": "A high school chemistry teacher...",
      "originalLanguage": "en",
      "firstAirDate": "2008-01-20",
      "network": "AMC"
    }
  ]
}
```

### Business error

```json
{
  "isError": true,
  "error": "No TMDB results found for 'xzqfnm'."
}
```

---

## `instructions` field

The `instructions` field is always present in every tool result. Key scenarios:

- **Confirmed match, identify mode** — present series metadata and suggested filenames grouped by season; highlight failures; offer to call `rename_folder` with confirmed `tmdbId`.
- **Confirmed match, rename mode** — report rename summary: successes and failures separately; warn prominently if all episodes failed.
- **Disambiguation** — ask one focused natural question using enriched candidate data; never show a numbered list; retry with `tmdbId` once confirmed.
- **Folder not found** — inform the user and suggest checking the path.
- **Metadata file malformed** — inform the user the metadata file is corrupted and needs to be deleted or fixed manually.
- **Error** — explain what went wrong in plain language and suggest next steps.

---

## Notes for Black Widow

- Mock `IFolderRenameOrchestrator` and `ITmdbClient`.
- Test `identify_folder`: confirmed match with enrichment, partial failures, disambiguation, folder not found, metadata malformed.
- Test `rename_folder`: confirmed match with enrichment, partial failures, all failures, disambiguation, folder not found, metadata malformed.
- Test that `tmdbId` in parameters skips identification in both tools.
- Test that enrichment is fetched on confirmed match and not fetched during disambiguation beyond `overview`/`originalLanguage`/`firstAirDate`/`network`.
- Test that `instructions` is present and non-empty in every result variant.
- Test that partial failures are reported correctly in both tools.
- E2E: disambiguation flow — agent receives candidates, asks contextual question, user selects, retry with `tmdbId` succeeds.
- E2E: full happy path `identify_folder` → user confirms → `rename_folder` with `tmdbId`.
- E2E: `rename_folder` with partial failures — correct summary returned.

## Notes for Tony Stark

- Implement `FolderMcpTools` in `Scoutarr.Mcp` with both tools.
- Reuse `GetTvShowEnrichmentAsync` from UC-02 — no new TMDB call needed for confirmed match enrichment.
- Enrich disambiguation candidates with `overview`, `originalLanguage`, `firstAirDate`, and `network` — one TMDB call per candidate, max 10.
- Extend the MCP server system prompt with the folder section defined above.
- `instructions` must always be present — never omit it even on error.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: MCP Server — folder tools
  As an AI agent
  I want to call identify_folder and rename_folder MCP tools
  So that I can guide the user conversationally through folder processing

  Scenario: identify_folder — confirmed match
    Given a call to identify_folder with folderPath "/downloads/breaking.bad.season1"
    And the series is identified successfully as "Breaking Bad"
    When the tool is called
    Then disambiguationNeeded is false
    And enriched series metadata is returned
    And suggested filenames are returned for all episodes
    And the instructions field tells the agent to present the results and offer rename_folder

  Scenario: identify_folder — partial failures
    Given a call to identify_folder with a folder containing 2 valid episodes and 1 unrecognised file
    When the tool is called
    Then 2 suggested filenames are returned
    And 1 failure is returned with the appropriate reason
    And the instructions field tells the agent to highlight the failure to the user

  Scenario: identify_folder — disambiguation needed
    Given a call to identify_folder whose folder name matches multiple TMDB results
    When the tool is called without tmdbId
    Then disambiguationNeeded is true
    And each candidate is enriched with overview, originalLanguage, firstAirDate, and network
    And the instructions field tells the agent to ask one focused natural question

  Scenario: Disambiguation — agent asks contextual question, user selects, retry succeeds
    Given identify_folder returns disambiguation candidates
    And the agent asks "Is this the AMC series with Bryan Cranston, or the BBC documentary?"
    And the user selects the AMC series
    When identify_folder is called again with the confirmed tmdbId
    Then the series is confirmed without disambiguation
    And suggested filenames are returned

  Scenario: identify_folder — tmdbId provided, identification skipped
    Given a call to identify_folder with a confirmed tmdbId
    When the tool is called
    Then series identification is skipped
    And suggested filenames are returned directly

  Scenario: rename_folder — all episodes renamed
    Given a call to rename_folder with a folder of 2 valid episodes
    And the series is identified successfully
    When the tool is called
    Then disambiguationNeeded is false
    And both episodes appear in the successes list
    And both files are moved to their destination
    And the instructions field tells the agent to report the summary to the user

  Scenario: rename_folder — partial failures
    Given a folder with 2 valid episodes and 1 file that already exists at destination
    When rename_folder is called
    Then 2 episodes appear in the successes list
    And 1 episode appears in the failures list with reason "File already exists at destination"
    And the instructions field tells the agent to report successes and failures separately

  Scenario: rename_folder — all episodes failed
    Given a folder where no episode numbers can be parsed
    When rename_folder is called
    Then the successes list is empty
    And all episodes appear in the failures list
    And the instructions field warns the agent to communicate this prominently to the user

  Scenario: Folder not found
    Given a call to rename_folder with a path that does not exist
    When the tool is called
    Then isError is true
    And the instructions field tells the agent to ask the user to check the path

  Scenario: Metadata file malformed
    Given a folder with a malformed metadata file
    When rename_folder is called
    Then isError is true
    And the instructions field tells the agent to inform the user the metadata file needs to be fixed or deleted
```

---

## Subtasks

- [ ] Implement `identify_folder` tool in `Scoutarr.Mcp`
- [ ] Implement `rename_folder` tool in `Scoutarr.Mcp`
- [ ] Extend MCP server system prompt with the folder section defined above
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Black Widow writes E2E tests in red in `Scoutarr.E2E.Tests`
- [ ] Tony Stark implements both tools
- [ ] Hawkeye reviews
