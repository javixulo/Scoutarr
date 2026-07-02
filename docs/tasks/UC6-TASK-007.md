# UC6-TASK-007 — MCP — identify and rename remap

**Requirements:**
- [requirements/interfaces.md](../requirements/interfaces.md) — "MCP Server"
- [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
- [ARCHITECTURE.md](../ARCHITECTURE.md) — "MCP Server"

**Dependencies:** UC6-TASK-003 (Identify), UC6-TASK-005 (Rename), UC6-TASK-006 (REST API — shared underlying logic)

---

## Context

The MCP server exposes two tools: `identify_remap` and `rename_remap`. Unlike UC1-TASK-006 (movie identification), there is no TMDB candidate ambiguity here — the destination is always provided explicitly by the user. The ambiguity this task handles is a **title conflict**: the destination given doesn't match the title currently encoded in the filename. The agent's role is to explain that mismatch conversationally and ask the user whether to proceed (`force`) or provide a different destination — never to force it silently.

Both tools are thin wrappers over the same UC-06 Identify/Rename logic already used by the REST controller (UC6-TASK-006) — no business logic is duplicated here.

`force` is only accepted by `identify_remap`. `rename_remap` never accepts it.

---

## Tools

### `identify_remap`
Computes the proposed mapping without touching the media file.

### `rename_remap`
Applies an already-`resolved` (or `noop`) mapping to disk. Internally calls identify first if no entry exists yet.

---

## Tool input schemas

### `identify_remap`

```json
{
  "path": "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv",
  "destination": { "season": 1, "episode": 45 },
  "force": false
}
```

### `rename_remap`

```json
{
  "path": "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv",
  "destination": { "season": 1, "episode": 45 }
}
```

---

## MCP server system prompt

```
You are a media file remapping assistant. You help users move an already-organised
TV episode file to a different season/episode when TMDB has restructured a series.

The user must always tell you the exact destination season/episode — you never guess it.

When you receive a conflict result (status: "conflict"):
- Explain plainly that the title currently in the filename doesn't match the title of
  the destination episode in TMDB.
- Show both titles so the user can judge for themselves.
- Ask whether they want to proceed anyway (force) or provide a different destination.
- Never force the destination without an explicit confirmation from the user in this turn.

When you receive a noop result (status: "noop"):
- Tell the user the file is already at the requested destination — there is nothing to do.

When you receive a resolved or applied result:
- Confirm plainly what happened: the file's previous location and its new one.

When you receive an error:
- Explain what went wrong in plain language (e.g. the destination doesn't exist in TMDB,
  or the file's current season/episode couldn't be determined) and suggest what the user
  can do next.
```

---

## Response shapes

### Resolved / applied / noop

```json
{
  "originalPath": "Season 03/Breaking Bad - S03E10 - The Beginning.mkv",
  "proposedPath": "Season 01/Breaking Bad - S01E45 - The Beginning.mkv",
  "mode": "identify",
  "status": "resolved",
  "origin": { "season": 3, "episode": 10 },
  "destination": { "season": 1, "episode": 45 },
  "titleCheck": { "result": "match", "currentTitle": "The Beginning", "destinationTitle": "The Beginning" }
}
```

For `noop`, `titleCheck` is omitted and `proposedPath` equals `originalPath`.

### Conflict (`isError: false`)

```json
{
  "status": "conflict",
  "instructions": "Explain the title mismatch to the user and ask whether to force this destination or provide a different one.",
  "originalPath": "Season 03/Breaking Bad - S03E10 - Problem Dog.mkv",
  "proposedPath": "Season 01/Breaking Bad - S01E48 - Hermanos.mkv",
  "destination": { "season": 1, "episode": 48 },
  "titleCheck": { "result": "mismatch", "currentTitle": "Problem Dog", "destinationTitle": "Hermanos" }
}
```

### Business error (`isError: true`)

```json
{
  "isError": true,
  "error": "Season 9 does not exist for this series."
}
```

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: MCP — identify and rename remap
  As an AI agent
  I want to call identify_remap and rename_remap MCP tools
  So that I can help the user remap a file conversationally, always confirming conflicts explicitly

  Scenario: Resolved destination returns success result
    Given a call to identify_remap with path "Season 03/Breaking Bad - S03E10 - The Beginning.mkv" and destination S01E45
    And Core returns status "resolved"
    When the tool is called
    Then the result is not an error
    And the result contains status "resolved"

  Scenario: Noop destination returns success result
    Given a call to identify_remap with a destination equal to the file's current origin
    When the tool is called
    Then the result is not an error
    And the result contains status "noop"

  Scenario: Title conflict returns a non-error result with instructions
    Given a call to identify_remap with a destination whose title does not match the current filename's title
    And Core returns status "conflict"
    When the tool is called
    Then the result is not an error
    And the result contains a non-empty instructions field
    And the result contains both the current and destination titles

  Scenario: Force with the same destination resolves the conflict
    Given a prior conflict entry exists for this file with destination S01E45
    And a call to identify_remap with the same path, the same destination S01E45, and force true
    When the tool is called
    Then the result contains status "resolved"

  Scenario: Force with no matching prior conflict returns isError true
    Given no entry exists yet for this file
    And a call to identify_remap with destination S01E45 and force true
    When the tool is called
    Then the result has isError true

  Scenario: Destination does not exist in TMDB returns isError true
    Given a call to identify_remap with a destination season/episode that does not exist
    When the tool is called
    Then the result has isError true

  Scenario: rename_remap rejects an unconfirmed conflict
    Given a conflict entry exists for this file with destination S01E45
    And a call to rename_remap with the same path and destination
    When the tool is called
    Then the result has isError true
    And no file is moved

  Scenario: rename_remap applies a resolved entry
    Given a resolved entry exists for this file with destination S01E45
    And a call to rename_remap with the same path and destination
    When the tool is called
    Then the result is not an error
    And the result contains mode "rename" and status "applied"

  Scenario: rename_remap applies a noop entry
    Given a noop entry exists for this file
    And a call to rename_remap with the same path and destination
    When the tool is called
    Then the result is not an error
    And the result contains status "noop"
    And no file is moved
```

---

## Notes for Black Widow

- Mock the UC-06 Identify/Rename logic.
- Test that a `conflict` result is `isError: false` (not a hard error — it's a normal, expected outcome the agent must handle conversationally) and includes `instructions`.
- Test that `force` is honoured only when it matches the existing conflicted destination (same rule as UC6-TASK-003/006).
- Test that `rename_remap`'s tool input schema has no `force` field at all.

## Notes for Tony Stark

- Implement `RemapMcpTools` in `Scoutarr.Mcp`, thin wrapper over the same UC-06 Identify/Rename logic used by the REST controller (UC6-TASK-006) — no duplicated business logic between REST and MCP layers.
- Register the system prompt at startup, alongside the existing ones.
- Always include `instructions` on `conflict`, mirroring the enrichment pattern from UC1-TASK-006, but without TMDB candidate enrichment — there's nothing to enrich here since the destination is user-provided.

---

## Subtasks

- [ ] Register MCP server system prompt for remapping at startup
- [ ] Implement `RemapMcpTools` in `Scoutarr.Mcp` with `identify_remap` and `rename_remap` (note: `rename_remap` schema has no `force` field)
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements tools and system prompt registration
- [ ] Hawkeye reviews
