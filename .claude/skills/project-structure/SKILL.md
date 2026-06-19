---
name: project-structure
description: Use this skill when adding new files, projects, services, or controllers, or when you need to understand where something lives in the solution.
---

# Project Structure Skill

## Solution layout

```
Scoutarr.sln
    src/
        Scoutarr.Core/           ← business logic only
        Scoutarr.Api/            ← REST API (ASP.NET Core)
        Scoutarr.Mcp/            ← MCP Server (HTTP/SSE)
    tests/
        Scoutarr.Core.Tests/     ← unit tests
        Scoutarr.Api.Tests/      ← integration tests for REST
        Scoutarr.Mcp.Tests/      ← integration tests for MCP
        Scoutarr.E2E.Tests/      ← end-to-end tests against Docker
```

---

## Dependency rules

- `Scoutarr.Core` has **no** dependency on `Scoutarr.Api` or `Scoutarr.Mcp`.
- `Scoutarr.Api` and `Scoutarr.Mcp` depend on `Scoutarr.Core`.
- `Scoutarr.Api` and `Scoutarr.Mcp` never depend on each other.
- Test projects depend only on the project they test, plus mocking libraries.

---

## What lives where

### Scoutarr.Core
- Domain models (e.g. `RenameResult`, `MediaFile`, `SeriesMetadata`)
- Domain errors (e.g. `FileNotFoundException`, `NoTmdbMatchException`)
- Service interfaces (e.g. `IRenameService`, `ITmdbClient`)
- Service implementations
- Filename parser and heuristics
- Confidence score calculator
- Series metadata reader/writer

### Scoutarr.Api
- Controllers (one per resource: `/rename`, `/identify`)
- Request/response DTOs
- RFC 9457 error mapping from domain errors
- Dependency injection setup

### Scoutarr.Mcp
- MCP tool definitions
- MCP response mapping from domain results
- `isError` mapping from domain errors
- Dependency injection setup

---

## Naming conventions

- Services: `{Name}Service.cs` (e.g. `RenameService.cs`)
- Interfaces: `I{Name}.cs` (e.g. `IRenameService.cs`)
- Controllers: `{Name}Controller.cs`
- DTOs: `{Name}Request.cs`, `{Name}Response.cs`
- Tests: `{ClassName}Tests.cs`

---

## Adding a new service

1. Define the interface in `Scoutarr.Core`.
2. Implement it in `Scoutarr.Core`.
3. Register it in the DI container in both `Scoutarr.Api` and `Scoutarr.Mcp`.
4. Write unit tests in `Scoutarr.Core.Tests`.
