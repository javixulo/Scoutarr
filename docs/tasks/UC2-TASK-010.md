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

## Enrichment — series identification

When `identify_series` returns a confirmed match, the MCP layer fetches:
- Creator, Cast (top 5), Network, Genres, Status, Seasons (with episode counts)

Season 0 (specials) is included in the `Seasons` list if present.

When disambiguation is needed, candidates are enriched with `overview` and `originalLanguage` only — no extra TMDB call.

---

## `instructions` field

The `instructions` field is always present in the tool result. Key scenarios:

- **Confirmed series match** — present to user for confirmation, then call `identify_episode` with `tvId`.
- **Series disambiguation** — ask user to select a candidate, then retry with `candidateIndex`.
- **Confirmed episode** — present suggested filename, then call `rename_episode` if confirmed.
- **Rename success** — inform user of destination path and moved subtitles.
- **Error** — inform user and suggest next steps.

---

## Subtasks

- [ ] Add `GetTvShowEnrichmentAsync` to `ITmdbClient` and implement in `TmdbClient`
- [ ] Implement `identify_series` tool in `Scoutarr.Mcp`
- [ ] Implement `identify_episode` tool in `Scoutarr.Mcp`
- [ ] Implement `rename_episode` tool in `Scoutarr.Mcp`
- [ ] Extend MCP server system prompt to cover TV episode flow
- [ ] Black Widow writes tests in red
- [ ] Black Widow writes E2E tests in red
- [ ] Tony Stark implements all three tools
- [ ] Hawkeye reviews
