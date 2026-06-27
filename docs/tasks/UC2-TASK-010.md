# UC2-TASK-010 — MCP Server — identify series, identify episode, rename episode

**Requirements:** [requirements/interfaces.md](../requirements/interfaces.md) — "MCP Server", "Responses and notifications"
**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow"
**Requirements:** [ARCHITECTURE.md](../ARCHITECTURE.md) — "MCP Server"
**Dependencies:** UC2-TASK-004 (ITvShowIdentificationService), UC2-TASK-006 (ITvEpisodeLookupService), UC2-TASK-008 (ITvEpisodeMoveService)

---

## Context

Three MCP tools, all thin layers over Core services. The MCP layer is responsible for enriching results with additional TMDB data before returning them to the agent, and for providing `instructions` that guide the agent through disambiguation and multi-step flows.

The TV episode flow spans two tools: `identify_series` first (with possible disambiguation), then `identify_episode` once the series is confirmed.

---

## Tools

### `identify_series`
Identifies the TV series. Returns a confirmed match (with enriched metadata) or a disambiguation result.

### `identify_episode`
Given a confirmed series `tvId`, identifies the episode and returns the suggested output filename.

### `rename_episode`
Identifies series and episode, then moves the file to `/media/tv`.

---

## MCP server system prompt (TV episode section)

```
When identifying a TV series and disambiguation is needed:
- Never present a numbered list of candidates to the user.
- Ask one focused, natural question at a time using the enriched candidate data.
- Use creator, cast, network, genre, and status information to phrase contextual questions.
- Examples: "Is this the AMC series created by Vince Gilligan, or the BBC documentary series?",
  "Does this star Bryan Cranston and Aaron Paul?"
- If the user explicitly cancels, confirm the cancellation and do not modify any files.

When a series is confirmed:
- Present the match naturally: title, year, network, and season count.
- Then call identify_episode with the confirmed tvId.

When an episode is confirmed:
- Present the suggested filename to the user for confirmation.
- Then call rename_episode if the user approves.

When rename succeeds:
- Inform the user of the destination path and any moved subtitle files.

When an error occurs:
- Explain what went wrong in plain language and suggest what the user can do next.
```

---

## Enrichment — series identification

When `identify_series` returns a confirmed match, the MCP layer fetches:
- Creator, Cast (top 5), Network, Genres, Status, Seasons (with episode counts)

Season 0 (specials) is included in the `Seasons` list if present.

When disambiguation is needed, candidates are enriched with `overview`, `originalLanguage`, `firstAirDate`, and `network` — one extra TMDB call per candidate, max 10 candidates.

---

## Response shapes

### Confirmed series match

```json
{
  "disambiguationNeeded": false,
  "tvId": 1396,
  "title": "Breaking Bad",
  "year": 2008,
  "network": "AMC",
  "status": "Ended",
  "genres": ["Crime", "Drama", "Thriller"],
  "creator": "Vince Gilligan",
  "topCast": ["Bryan Cranston", "Aaron Paul", "Anna Gunn", "Dean Norris", "Betsy Brandt"],
  "seasons": [
    { "seasonNumber": 1, "episodeCount": 7 },
    { "seasonNumber": 2, "episodeCount": 13 }
  ],
  "instructions": "Present the confirmed series to the user and call identify_episode with tvId."
}
```

### Disambiguation needed

```json
{
  "disambiguationNeeded": true,
  "instructions": "Ask the user one focused, natural question to narrow down the candidates. Never show a numbered list.",
  "candidates": [
    {
      "tvId": 1396,
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
  "error": "No TMDB results found for 'Zxqfnmvp'."
}
```

---

## `instructions` field

The `instructions` field is always present in the tool result. Key scenarios:

- **Confirmed series match** — present to user for confirmation, then call `identify_episode` with `tvId`.
- **Series disambiguation** — ask one focused question using enriched candidate data; never show a numbered list; retry with `tvId` once confirmed.
- **Confirmed episode** — present suggested filename, then call `rename_episode` if confirmed.
- **Rename success** — inform user of destination path and moved subtitles.
- **Error** — inform user and suggest next steps.

---

## Notes for Black Widow

- Mock `ITvShowIdentificationService`, `ITvEpisodeLookupService`, `ITvEpisodeMoveService`, and `ITmdbClient`.
- Test confirmed series match: enrichment is fetched, `disambiguationNeeded` is false, `instructions` present.
- Test disambiguation: `disambiguationNeeded` is true, candidates are enriched, `instructions` present, no numbered list instruction included.
- Test that `tvId` in parameters bypasses identification and returns confirmed match directly.
- Test confirmed episode: suggested filename returned, `instructions` present.
- Test rename success: destination path and subtitles in result, `instructions` present.
- Test business error: `isError` true, plain language message.
- E2E: disambiguation flow — agent receives candidates, user selects via natural question, retry with `tvId` succeeds.
- E2E: full happy path — identify series, identify episode, rename episode.

## Notes for Tony Stark

- Implement `TvEpisodeMcpTools` in `Scoutarr.Mcp` with all three tools.
- Add `GetTvShowEnrichmentAsync` to `ITmdbClient` and implement in `TmdbClient`.
- Enrich disambiguation candidates with `overview`, `originalLanguage`, `firstAirDate`, and `network` — one TMDB call per candidate, max 10.
- Extend the MCP server system prompt with the TV episode section defined above.
- `instructions` must always be present — never omit it even on error.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: MCP Server — TV episode tools
  As an AI agent
  I want to call identify_series, identify_episode, and rename_episode MCP tools
  So that I can guide the user conversationally through TV episode identification and renaming

  Scenario: identify_series — confirmed match
    Given a call to identify_series with filename "breaking.bad.s01e01.mkv"
    And Core returns a confirmed series match for "Breaking Bad"
    When the tool is called
    Then disambiguationNeeded is false
    And the result contains enriched series metadata
    And the instructions field is present and tells the agent to call identify_episode

  Scenario: identify_series — disambiguation needed
    Given a call to identify_series with filename "the.office.mkv"
    And Core returns disambiguation candidates
    When the tool is called
    Then disambiguationNeeded is true
    And each candidate is enriched with overview, originalLanguage, firstAirDate, and network
    And the instructions field tells the agent to ask one focused natural question

  Scenario: identify_series — tvId provided, identification skipped
    Given a call to identify_series with tvId 1396
    When the tool is called
    Then Core identification is skipped
    And the confirmed series match is returned directly

  Scenario: Disambiguation — agent asks contextual question, user selects, retry succeeds
    Given identify_series returns disambiguation candidates for "the office"
    And the agent asks "Is this the US version starring Steve Carell, or the UK original with Ricky Gervais?"
    And the user selects the US version
    When identify_series is called again with the confirmed tvId
    Then the series is confirmed without disambiguation

  Scenario: identify_episode — confirmed episode
    Given a confirmed series tvId 1396
    And a call to identify_episode with filename "breaking.bad.s01e01.mkv"
    When the tool is called
    Then the suggested filename is returned
    And the instructions field tells the agent to present the filename and offer to rename

  Scenario: rename_episode — success
    Given a confirmed series and episode
    When rename_episode is called
    Then the destination path is returned
    And any moved subtitle files are listed
    And the instructions field tells the agent to confirm the rename to the user

  Scenario: Business error — no TMDB results
    Given a call to identify_series with filename "Zxqfnmvp.mkv"
    And Core returns no TMDB results
    When the tool is called
    Then isError is true
    And the error message is in plain language
```

---

## Subtasks

- [ ] Add `GetTvShowEnrichmentAsync` to `ITmdbClient` and implement in `TmdbClient`
- [ ] Implement `identify_series` tool in `Scoutarr.Mcp`
- [ ] Implement `identify_episode` tool in `Scoutarr.Mcp`
- [ ] Implement `rename_episode` tool in `Scoutarr.Mcp`
- [ ] Extend MCP server system prompt with TV episode section
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Black Widow writes E2E tests in red
- [ ] Tony Stark implements all three tools
- [ ] Hawkeye reviews
