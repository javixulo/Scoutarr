# TASK-006 — MCP — identify and rename movie

**Requirements:**
- [requirements/interfaces.md](../requirements/interfaces.md) — "MCP Server"
- [requirements/identification.md](../requirements/identification.md) — "Disambiguation flow"
- [ARCHITECTURE.md](../ARCHITECTURE.md) — "MCP Server"
- [skills/tmdb-api/SKILL.md](skills/tmdb-api/SKILL.md) — "Movie enrichment endpoint"

**Dependencies:** TASK-004 (Core orchestration)

---

## Context

The MCP server exposes two tools: `identify_movie` and `rename_movie`. These are thin wrappers over `IMovieIdentificationService`, just like the REST API — but the disambiguation flow is fundamentally different.

When no candidate exceeds the confidence threshold, the MCP layer enriches the candidates with additional TMDB data (director, top cast, collection, genres) and returns a structured disambiguation result with an `instructions` field that guides the agent on how to proceed.

The agent's conversational behaviour is shaped by two mechanisms:
1. **MCP server system prompt** — loaded once when the agent connects; sets overall behaviour.
2. **`instructions` field in the tool result** — returned per-call; guides the agent for that specific situation.

The Core does not know about enrichment. Enrichment happens in the MCP layer, after Core returns `MovieDisambiguationNeeded`.

---

## Tools

### `identify_movie`
Returns a suggested filename without touching the filesystem.

### `rename_movie`
Applies the rename to disk. Internally calls identify first.

---

## Tool input schema (both tools)

```json
{
  "filename": "Batman.mkv",
  "tmdbId": null,
  "year": null,
  "originalLanguage": null,
  "genre": null
}
```

- `filename` — required.
- `tmdbId` — optional; when provided, bypasses search and scoring entirely. This is how the agent resolves disambiguation after the user confirms a candidate.
- `year`, `originalLanguage`, `genre` — optional hints passed to Core.

---

## MCP server system prompt

The following system prompt is registered on the MCP server and loaded by the agent on connection:

```
You are a media file identification assistant. You help users identify and rename movie and TV show files using TMDB metadata.

When you receive a disambiguation result (disambiguationNeeded: true):
- Never present a numbered list of candidates to the user.
- Ask one focused, natural question at a time using the enriched candidate data.
- Use director, cast, collection, and genre information to phrase contextual questions.
- If the user provides a new piece of information (actor, director, year), use it to narrow the candidates yourself before asking another question.
- If the user's answer rules out all remaining candidates, tell them no match was found and ask if they want to try with different information.
- If the user explicitly cancels, confirm the cancellation and do not modify any files.

When you receive a successful identification:
- Confirm the match naturally: title, year, and what will happen (identify vs rename).

When you receive an error:
- Explain what went wrong in plain language and suggest what the user can do next.
```

---

## Response shapes

### Success

```json
{
  "originalFilename": "Batman.mkv",
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

### Disambiguation needed

```json
{
  "disambiguationNeeded": true,
  "originalFilename": "Batman.mkv",
  "instructions": "Ask the user one focused question to narrow down the candidates. Use director, cast, collection, or genre. Do not list all candidates at once.",
  "candidates": [
    {
      "tmdbId": 414906,
      "title": "The Batman",
      "year": 2022,
      "overview": "...",
      "director": "Matt Reeves",
      "topCast": ["Robert Pattinson", "Zoë Kravitz", "Jeffrey Wright"],
      "collection": null,
      "genres": ["Action", "Crime", "Drama"]
    }
  ]
}
```

### Business error — `isError: true`

```json
{
  "isError": true,
  "error": "No TMDB results found for 'Zxqfnmvp'."
}
```

---

## Notes for Black Widow

- Use a test MCP client or direct service invocation — no real TMDB, no real filesystem.
- Mock `IMovieIdentificationService` and `ITmdbClient`.
- Test that disambiguation result is returned as `isError: false` with enriched candidates and `instructions` field.
- Test that `tmdbId` bypasses disambiguation and returns success directly.
- Test that business errors are returned as `isError: true`.

## Notes for Tony Stark

- Implement `MovieMcpTools` in `Scoutarr.Mcp`.
- Register the system prompt on the MCP server at startup.
- Enrichment: call `ITmdbClient.GetMovieDetailsAsync` for each candidate when `MovieDisambiguationNeeded` is returned. Max 10 candidates.
- Extract director (crew where `job = "Director"`), top 3 cast by `order`, `belongs_to_collection.name`, and genre names.
- Always include the `instructions` field in the disambiguation result.
- Add `GetMovieDetailsAsync(int movieId)` to `ITmdbClient` if not already defined by TASK-002.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: MCP — identify and rename movie
  As an AI agent
  I want to call identify_movie and rename_movie MCP tools
  So that I can help the user identify and rename their movie files conversationally

  Scenario: High confidence match returns success result
    Given a call to identify_movie with filename "The.Batman.2022.1080p.mkv"
    And Core returns a successful identification with confidence 0.97
    When the tool is called
    Then the result is not an error
    And the result contains filename "The Batman (2022).mkv"
    And the result contains mode "identify"
    And the result contains match with tmdbId, title, year, and confidence

  Scenario: Low confidence returns enriched disambiguation result with instructions
    Given a call to identify_movie with filename "Batman.mkv"
    And Core returns MovieDisambiguationNeeded with 3 candidates
    And TMDB enrichment returns director, top cast, collection and genres for each candidate
    When the tool is called
    Then the result is not an error
    And the result contains disambiguationNeeded true
    And the result contains a non-empty instructions field
    And each candidate includes director, topCast, collection, and genres

  Scenario: tmdbId bypasses disambiguation and returns success
    Given a call to identify_movie with filename "Batman.mkv" and tmdbId 414906
    And Core bypasses search and returns a successful identification for tmdbId 414906
    When the tool is called
    Then the result is not an error
    And the result contains mode "identify"

  Scenario: No TMDB results returns isError true
    Given a call to identify_movie with filename "Zxqfnmvp.mkv"
    And Core returns a no TMDB results error
    When the tool is called
    Then the result has isError true
    And the result contains a human-readable error message
```

---

## Subtasks

- [ ] Add `GetMovieDetailsAsync(int movieId)` to `ITmdbClient` in `Scoutarr.Core` (if not done in TASK-002)
- [ ] Define `EnrichedMovieCandidate` record in `Scoutarr.Mcp`
- [ ] Register MCP server system prompt at startup
- [ ] Implement `MovieMcpTools` in `Scoutarr.Mcp` with `identify_movie` and `rename_movie`
- [ ] Implement enrichment logic in MCP layer
- [ ] Black Widow writes tests in red (all Gherkin scenarios above)
- [ ] Tony Stark implements tools, enrichment, and system prompt registration
- [ ] Hawkeye reviews
