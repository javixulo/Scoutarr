# TASK-000 — Project bootstrap — solution structure, configuration, logging

**Architecture:** [ARCHITECTURE.md](../ARCHITECTURE.md) — sections 1, 3, 4, 5
**Requirements:** [REQUIREMENTS.md](../REQUIREMENTS.md) — "Logging", "Deployment", "Technology"

---

## Context

This task creates the skeleton on which every other task builds. Nothing functional is implemented here — no parsing, no TMDB calls, no renaming. The goal is a solution that compiles, has the correct project layout, boots inside Docker, and gives every subsequent task a solid, consistent foundation to extend.

This task also produces the Dockerfile that TASK-001 (CI/CD pipeline) will consume to build and publish the image on every merge to main.

---

## Deliverables

### 1. Solution and project structure

```
Scoutarr.sln
    src/
        Scoutarr.Core        — business logic (no HTTP, no MCP, no filesystem)
        Scoutarr.Api         — ASP.NET Core REST API
        Scoutarr.Mcp         — MCP Server (HTTP/SSE)
    tests/
        Scoutarr.Core.Tests
        Scoutarr.Api.Tests
        Scoutarr.Mcp.Tests
        Scoutarr.E2E.Tests
```

Dependency rules (enforced by project references — never add a reference that violates these):
- `Scoutarr.Api` → `Scoutarr.Core`
- `Scoutarr.Mcp` → `Scoutarr.Core`
- `Scoutarr.Api` and `Scoutarr.Mcp` must never reference each other
- `Scoutarr.Core` must never reference `Api` or `Mcp`

### 2. NuGet packages

| Project | Package | Purpose |
|---|---|---|
| `Scoutarr.Core` | `TMDbLib` | TMDB HTTP client |
| `Scoutarr.Core` | `Microsoft.Extensions.DependencyInjection` | DI abstractions |
| `Scoutarr.Core` | `Microsoft.Extensions.Logging.Abstractions` | Logging abstractions |
| `Scoutarr.Core` | `MediaInfo.Wrapper.Core` | File duration reading (UC-05) |
| `Scoutarr.Api` | `Microsoft.AspNetCore` | REST API host |
| `Scoutarr.Mcp` | `Microsoft.AspNetCore` | HTTP/SSE host |
| `*.Tests` | `xunit`, `xunit.runner.visualstudio` | Test framework |
| `*.Tests` | `Moq` | Mocking |
| `*.Tests` | `Microsoft.NET.Test.Sdk` | Test runner |
| `Scoutarr.Api.Tests` | `Microsoft.AspNetCore.Mvc.Testing` | Integration test host |
| `Scoutarr.Mcp.Tests` | `Microsoft.AspNetCore.Mvc.Testing` | Integration test host |

### 3. Configuration

All configuration is read from environment variables. Define a `ScoutarrOptions` class in `Scoutarr.Core` bound via `IOptions<ScoutarrOptions>`:

| Variable | Property | Type | Default |
|---|---|---|---|
| `TMDB_API_KEY` | `TmdbApiKey` | `string` | — (required) |
| `SCRAPING_LANGUAGE` | `ScrapingLanguage` | `string` | `"en"` |
| `CONFIDENCE_THRESHOLD` | `ConfidenceThreshold` | `double` | `0.80` |
| `NAMING_FORMAT_MOVIE` | `NamingFormatMovie` | `string` | `"{title} ({year})"` |
| `NAMING_FORMAT_EPISODE` | `NamingFormatEpisode` | `string` | `"{series} - S{season:00}E{episode:00} - {title}"` |

Startup must fail fast with a clear error message if `TMDB_API_KEY` is missing or empty.

### 4. Logging

Logging is configured in both `Scoutarr.Api` and `Scoutarr.Mcp` hosts. Two sinks, always active simultaneously:

- **stdout** — structured JSON, Docker captures this stream via its logging drivers.
- **File** — written to `/config/logs/scoutarr.log` inside the container; `/config` is a mounted volume so logs persist across container restarts.

Minimum log fields on every entry: timestamp, severity level, message. Operations in Core will add context (file affected, operation, outcome) as structured properties.

Use `Microsoft.Extensions.Logging` abstractions in `Scoutarr.Core` — never depend on a specific logging library there. The hosts wire up the concrete provider.

### 5. Core abstractions

Define these in `Scoutarr.Core`. No implementation required beyond what is needed to compile and register in DI.

**`IFileSystem`** — filesystem abstraction; all filesystem operations in Core go through this interface so tests can mock them without touching disk:

```csharp
public interface IFileSystem
{
    bool FileExists(string path);
    bool DirectoryExists(string path);
    void CreateDirectory(string path);
    void MoveFile(string sourcePath, string destinationPath);
    void DeleteFile(string path);
    IEnumerable<string> EnumerateFiles(string path, string searchPattern, SearchOption searchOption);
    IEnumerable<string> EnumerateDirectories(string path);
    long GetFileSize(string path);
    string ReadAllText(string path);
    void WriteAllText(string path, string contents);
}
```

**Domain error base type** — all Core services return results or domain errors; the interfaces (Api, Mcp) translate these into their own formats:

```csharp
public abstract record DomainError(string Reason, string Detail);
```

### 6. API skeleton

`Scoutarr.Api` must:
- Start an ASP.NET Core host on port `8080`.
- Register a `/health` endpoint returning `200 OK` with body `{ "status": "ok" }` — used by Docker `HEALTHCHECK` and E2E tests.
- Register RFC 9457 Problem Details middleware for unhandled exceptions.
- Have no actual business endpoints yet — those come in UC-01 onwards.

### 7. MCP skeleton

`Scoutarr.Mcp` must:
- Start an HTTP/SSE host on port `8080` at path `/mcp`.
- Respond to MCP protocol handshake (capability negotiation) with an empty tool list.
- Have no actual tools yet — those come in UC-01 onwards.

### 8. Dockerfile and docker-compose.yml

> **Owner: Jarvis.** Tony Stark must have the solution compiling and the `/health` endpoint responding before Jarvis picks this up.

Jarvis delivers:

**Dockerfile** — multi-stage build:

```
Stage 1 — build
    Base image: mcr.microsoft.com/dotnet/sdk (latest LTS)
    Restore, build, publish Scoutarr.Api and Scoutarr.Mcp in Release configuration

Stage 2 — runtime
    Base image: mcr.microsoft.com/dotnet/aspnet (latest LTS)
    Install system packages required by MediaInfo.Wrapper.Core on Linux:
        libzen0v5 libmms0 zlib1g libnghttp2-14 librtmp1 libcurl4
    Create volume directories with correct permissions:
        /config/logs
        /input/movies
        /input/tv
        /media/movies
        /media/tv
    Copy published output from build stage
    EXPOSE 8080
    HEALTHCHECK: GET http://localhost:8080/health
    ENTRYPOINT: dotnet Scoutarr.Api.dll (or a wrapper script if both Api and Mcp run in the same container)
```

> Note for Jarvis: decide at implementation time whether Api and Mcp run as a single process sharing a host or as two separate processes in the same container. Document the decision in a comment in the Dockerfile. Either approach is acceptable for MVP.

**docker-compose.yml** — at repo root, matching the example in ARCHITECTURE.md section 4, with all five volume mounts and all environment variables.

### 9. Smoke tests

One smoke test per test project to verify the solution is wired correctly:

- `Scoutarr.Core.Tests` — `ScoutarrOptions` binds correctly from a dictionary of environment values; `TmdbApiKey` missing throws on validation.
- `Scoutarr.Api.Tests` — `GET /health` returns `200 OK` with `{ "status": "ok" }`.
- `Scoutarr.Mcp.Tests` — MCP handshake returns a valid capability response with an empty tool list.
- `Scoutarr.E2E.Tests` — container starts from the built image, `GET http://localhost:8080/health` returns `200 OK`, `/config/logs/scoutarr.log` exists in the mounted volume.

---

## Notes for Black Widow

- Write the smoke tests described in section 9 in red before Tony Stark touches any code.
- For `Api.Tests` and `Mcp.Tests`, use `WebApplicationFactory<T>` — do not spin up a real HTTP server.
- For `E2E.Tests`, the test must build the Docker image locally and start a container with `Process` or `Testcontainers` (preferred). The container must be stopped and removed in test teardown regardless of outcome.
- Do not test configuration binding beyond what is in section 9 — that is Tony Stark's responsibility to get right, not a test surface.

## Notes for Tony Stark

- Target the latest .NET LTS at time of implementation.
- Use `Microsoft.Extensions.Logging` in Core — no Serilog dependency in Core itself. Wire Serilog (or equivalent) only in the host projects.
- `IFileSystem` real implementation (`PhysicalFileSystem`) wraps `System.IO` — keep it thin, no logic.
- Responsibility ends when the solution compiles, all C# smoke tests are green, and the `/health` endpoint responds in-process. Jarvis takes it from there.

## Notes for Jarvis

- The Dockerfile must produce a working image with `docker build -t scoutarr .` from the repo root — no extra scripts required.
- Verify that `docker run --rm -p 8080:8080 -e TMDB_API_KEY=test scoutarr` responds to `GET /health` with `200 OK` before marking this task done.
- The E2E smoke test depends on the image existing locally — coordinate with Black Widow so the test uses `docker build` as part of its setup if needed.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: Project bootstrap
  As any agent picking up a UC-01+ task
  I want a working solution skeleton
  So that I can add features without setting up infrastructure

  Scenario: Solution builds cleanly
    Given the repository is cloned
    When `dotnet build Scoutarr.sln` is run
    Then the build succeeds with zero errors and zero warnings

  Scenario: All tests pass on a clean clone
    Given the repository is cloned
    When `dotnet test Scoutarr.sln` is run
    Then all smoke tests pass

  Scenario: Health endpoint responds
    Given the API host is running
    When GET /health is called
    Then the response status is 200 OK
    And the body is { "status": "ok" }

  Scenario: MCP handshake succeeds
    Given the MCP host is running
    When the MCP capability negotiation request is sent
    Then the response contains a valid capability object with an empty tool list

  Scenario: Missing TMDB_API_KEY fails fast
    Given the TMDB_API_KEY environment variable is not set
    When the application starts
    Then startup fails with a clear error message referencing TMDB_API_KEY

  Scenario: Logs are written to file on disk
    Given the container is running with /config mounted to a host directory
    When any request is handled
    Then a log entry appears in /config/logs/scoutarr.log on the host

  Scenario: Docker image builds from repo root
    Given the repository is cloned
    When `docker build -t scoutarr .` is run from the repo root
    Then the image builds successfully
    And `docker run --rm -p 8080:8080 -e TMDB_API_KEY=test scoutarr` responds to GET /health with 200 OK
```

---

## Subtasks

- [ ] Tony Stark creates the .NET solution and all projects with correct references
- [ ] Tony Stark adds all NuGet packages
- [ ] Tony Stark implements `ScoutarrOptions` with validation
- [ ] Tony Stark implements `IFileSystem` / `PhysicalFileSystem`
- [ ] Tony Stark implements `DomainError` base type
- [ ] Tony Stark wires logging (stdout + file) in both host projects
- [ ] Tony Stark scaffolds `/health` endpoint in `Scoutarr.Api`
- [ ] Tony Stark scaffolds MCP handshake in `Scoutarr.Mcp`
- [ ] Black Widow writes smoke tests in red
- [ ] Tony Stark makes smoke tests green
- [ ] Jarvis writes Dockerfile and docker-compose.yml
- [ ] Jarvis verifies `docker build` + `docker run /health` manually
- [ ] Hawkeye reviews
