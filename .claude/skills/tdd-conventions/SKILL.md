---
name: tdd-conventions
description: Use this skill when writing, reviewing, or running tests in this project.
---

# TDD Conventions Skill

## The TDD cycle in this project

1. **Red** — Black Widow writes tests that fail because the implementation does not exist yet.
2. **Green** — Tony Stark writes the minimum code to make the tests pass.
3. **Refactor** — Tony Stark cleans up the code. Tests must remain green.

Tests are never written after the implementation. Tests define the expected behaviour.

---

## Test frameworks

- **xUnit** — test framework.
- **FluentAssertions** — assertion library. Always prefer over raw `Assert`.
- **NSubstitute** — mocking library. Use for all interface mocks.

---

## Test naming

Format: `{MethodName}_{Scenario}_{ExpectedResult}`

```csharp
// Good
ParseSxEyPattern_ValidFormat_ReturnsSeasonAndEpisode()
ParseSxEyPattern_NullInput_ThrowsArgumentException()
RenameMovie_FileNotFound_ReturnsFileNotFoundError()

// Bad
TestParser()
Test1()
ShouldWork()
```

---

## Task reference comments

Every test must include a comment referencing the originating feature and task it implements:

```csharp
// FEATURE-001-TASK-001: Filename parser — SxEy pattern detection
[Fact]
public void ParseSxEyPattern_ValidFormat_ReturnsSeasonAndEpisode() { ... }
```

---

## What to test

### Unit tests (Scoutarr.Core.Tests)
- All filename parsing heuristics, including edge cases.
- Confidence score calculation.
- Output filename formatting for all template tokens.
- Series metadata validation (episode out of range, malformed file, etc.).
- Domain error types and messages.

### Integration tests (Scoutarr.Api.Tests, Scoutarr.Mcp.Tests)
- Happy path: correct response shape on success.
- All error scenarios listed in `requirements/interfaces.md`.
- Correct HTTP status codes (REST).
- Correct `isError` behaviour (MCP).
- Request validation (missing fields, invalid formats).
- Always mock `ITmdbClient` and `IFileSystem`. Never call real services.

### E2E tests (Scoutarr.E2E.Tests)
- Run against the built Docker image, not source code.
- Use a real TMDB API key from environment.
- Use a temporary directory with real test media files.
- Cover: container starts, both interfaces reachable, rename succeeds end-to-end, logs written to volume.

---

## What NOT to test

- Private methods — test behaviour through public interfaces.
- Third-party library internals.
- Configuration reading — covered by E2E tests.
