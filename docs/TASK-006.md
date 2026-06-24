# TASK-006 — MCP — identify and rename movie

**Requirements:**
- [requirements/interfaces.md](requirements/interfaces.md) — "MCP Server"
- [requirements/identification.md](requirements/identification.md) — "Disambiguation flow"
- [ARCHITECTURE.md](ARCHITECTURE.md) — "MCP Server"
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

### Disambiguation needed — `isError: false`

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
    },
    {
      "tmdbId": 272,
      "title": "Batman Begins",
      "year": 2005,
      "overview": "...",
      "director": "Christopher Nolan",
      "topCast": ["Christian Bale", "Michael Caine", "Liam Neeson"],
      "collection": "The Dark Knight Collection",
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

## Disambiguation flow (technical)

```
1. Agent calls identify_movie("Batman.mkv")
2. Core returns MovieDisambiguationNeeded with N candidates
3. MCP layer enriches candidates via GET /movie/{id}?append_to_response=credits
4. MCP returns disambiguation result with instructions field
5. Agent reads instructions + candidates, asks user one contextual question
6. User answers → agent narrows candidates internally, asks next question if needed
   OR user provides extra data → agent filters candidates, calls tool again if one remains
7. Agent calls identify_movie("Batman.mkv", tmdbId: 414906) with confirmed ID
8. MCP returns success → agent confirms to user
```

If the user cancels, the agent reports cancellation — no files are modified, no further tool calls.

---

## Conversational flow reference

These are reference examples of how the agent should behave. They are not automated tests — they document expected agent behaviour guided by the system prompt and `instructions` field.

### Flow 1 — Agent asks about director, user confirms

```
User:   identify batman.mkv
Agent:  → calls identify_movie("batman.mkv")
        ← disambiguation result: Matt Reeves (2022), Christopher Nolan (2005, Dark Knight trilogy)...

Agent:  "I found several Batman films. Is this the 2022 version directed by Matt Reeves?"
User:   "Yes"
Agent:  → calls identify_movie("batman.mkv", tmdbId: 414906)
        ← success: "The Batman (2022).mkv"
Agent:  "Got it — this is The Batman (2022). The suggested filename is 'The Batman (2022).mkv'."
```

### Flow 2 — Agent asks about director, user says no, agent asks next question

```
User:   identify batman.mkv
Agent:  → calls identify_movie("batman.mkv")
        ← disambiguation result: Matt Reeves (2022), Christopher Nolan trilogy (2005/2008/2012)...

Agent:  "Is this the 2022 Batman directed by Matt Reeves?"
User:   "No"
Agent:  [rules out tmdbId 414906, looks at remaining candidates]
        "Is it from Christopher Nolan's Dark Knight trilogy? If so, which one —
        Batman Begins (2005), The Dark Knight (2008), or The Dark Knight Rises (2012)?"
User:   "The Dark Knight"
Agent:  → calls identify_movie("batman.mkv", tmdbId: 155)
        ← success
Agent:  "Perfect — The Dark Knight (2008). Suggested filename: 'The Dark Knight (2008).mkv'."
```

### Flow 3 — User volunteers extra information

```
User:   identify batman.mkv
Agent:  → calls identify_movie("batman.mkv")
        ← disambiguation result with multiple candidates

Agent:  "I found several Batman films. Is this the 2022 version directed by Matt Reeves?"
User:   "I don't know the director, but the main actor is Christian Bale"
Agent:  [filters candidates by cast — only Dark Knight trilogy matches]
        "That sounds like it could be from Christopher Nolan's Dark Knight trilogy.
        Is it Batman Begins, The Dark Knight, or The Dark Knight Rises?"
User:   "Batman Begins"
Agent:  → calls identify_movie("batman.mkv", tmdbId: 272)
        ← success
Agent:  "Batman Begins (2005) — suggested filename: 'Batman Begins (2005).mkv'."
```

### Flow 4 — User provides extra data but no candidate matches

```
User:   identify batman.mkv
Agent:  → calls identify_movie("batman.mkv")
        ← disambiguation result with multiple candidates

Agent:  "Is this the 2022 Batman directed by Matt Reeves?"
User:   "No, the main actor is Ben Affleck"
Agent:  [filters candidates by cast — no Ben Affleck in current candidate list]
        "None of the candidates I found feature Ben Affleck. Could you tell me
        the year it was released? That would help me search more precisely."
User:   "2016"
Agent:  → calls identify_movie("batman.mkv", year: 2016)
        ← success or new disambiguation
```

### Flow 5 — User cancels

```
User:   identify batman.mkv
Agent:  → calls identify_movie("batman.mkv")
        ← disambiguation result

Agent:  "I found several Batman films. Is this the 2022 version directed by Matt Reeves?"
User:   "Actually forget it"
Agent:  "No problem — no files have been modified."
        [no further tool calls]
```

---

## Notes for Black Widow

- Use a test MCP client or direct service invocation — no real TMDB, no real filesystem.
- Mock `IMovieIdentificationService` and `ITmdbClient`.
- Test that disambiguation result is returned as `isError: false` with enriched candidates and `instructions` field.
- Test that `tmdbId` bypasses disambiguation and returns success directly.
- Test that business errors are returned as `isError: true`.
- Do not test agent reasoning or conversational flows — those are covered by the reference examples above, not by automated tests.

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

  # ─────────────────────────────────────────
  # IDENTIFY — happy path
  # ─────────────────────────────────────────

  Scenario: High confidence match returns success result
    Given a call to identify_movie with filename "The.Batman.2022.1080p.mkv"
    And Core returns a successful identification with confidence 0.97
    When the tool is called
    Then the result is not an error
    And the result contains filename "The Batman (2022).mkv"
    And the result contains mode "identify"
    And the result contains match with tmdbId, title, year, and confidence

  # ─────────────────────────────────────────
  # IDENTIFY — disambiguation
  # ─────────────────────────────────────────

  Scenario: Low confidence returns enriched disambiguation result with instructions
    Given a call to identify_movie with filename "Batman.mkv"
    And Core returns MovieDisambiguationNeeded with 3 candidates
    And TMDB enrichment returns director, top cast, collection and genres for each candidate
    When the tool is called
    Then the result is not an error
    And the result contains disambiguationNeeded true
    And the result contains a non-empty instructions field
    And each candidate includes director, topCast, collection, and genres

  Scenario: Enriched candidate includes director name
    Given a call to identify_movie with filename "Batman.mkv"
    And Core returns MovieDisambiguationNeeded
    And one candidate's TMDB details show director "Christopher Nolan"
    When the tool is called
    Then that candidate's director field is "Christopher Nolan"

  Scenario: Enriched candidate includes top 3 cast members by order
    Given a call to identify_movie with filename "Batman.mkv"
    And Core returns MovieDisambiguationNeeded
    And one candidate's TMDB credits has 10 cast members
    When the tool is called
    Then that candidate's topCast contains exactly 3 names in order

  Scenario: Enriched candidate includes collection name when present
    Given a call to identify_movie with filename "Batman.mkv"
    And one candidate belongs to "The Dark Knight Collection"
    When the tool is called
    Then that candidate's collection field is "The Dark Knight Collection"

  Scenario: Enriched candidate has null collection when not part of a series
    Given a call to identify_movie with filename "Batman.mkv"
    And one candidate does not belong to any collection
    When the tool is called
    Then that candidate's collection field is null

  Scenario: tmdbId bypasses disambiguation and returns success
    Given a call to identify_movie with filename "Batman.mkv" and tmdbId 414906
    And Core bypasses search and returns a successful identification for tmdbId 414906
    When the tool is called
    Then the result is not an error
    And the result contains mode "identify"

  # ─────────────────────────────────────────
  # RENAME — happy path
  # ─────────────────────────────────────────

  Scenario: Rename applies rename on disk and returns success
    Given a call to rename_movie with filename "The.Batman.2022.1080p.mkv"
    And Core returns a successful identification
    And the filesystem rename succeeds
    When the tool is called
    Then the result is not an error
    And the result contains mode "rename"
    And the result contains filename "The Batman (2022).mkv"

  # ─────────────────────────────────────────
  # ERRORS — isError: true
  # ─────────────────────────────────────────

  Scenario: No TMDB results returns isError true
    Given a call to identify_movie with filename "Zxqfnmvp.mkv"
    And Core returns a no TMDB results error
    When the tool is called
    Then the result has isError true
    And the result contains a human-readable error message

  Scenario: TMDB unreachable returns isError true
    Given a call to identify_movie with filename "The.Batman.2022.mkv"
    And Core returns a TMDB unreachable error
    When the tool is called
    Then the result has isError true

  Scenario: File not found returns isError true
    Given a call to rename_movie with filename "notexist.mkv"
    And the file does not exist on disk
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
