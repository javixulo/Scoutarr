# Scoutarr – Architecture

> Technical decisions derived from [REQUIREMENTS.md](REQUIREMENTS.md).

---

## Stack

- **Language:** C# / .NET (latest LTS)
- **Runtime:** Linux (Docker), cross-platform
- **TMDB client:** [TMDbLib](https://github.com/LordMike/TMDbLib)

---

## Project structure

See @skills/project-structure/SKILL.md for the full layout and dependency rules.

Key decisions:
- `Scoutarr.Core` has no knowledge of HTTP or MCP — pure business logic only.
- `Scoutarr.Api` and `Scoutarr.Mcp` are thin layers that delegate to Core via dependency injection. They never depend on each other.
- Core defines domain errors. Each interface translates them to its own format.
- Interfaces validate request shape. Core validates business rules.

---

## Interfaces

**REST API** — follows RFC 9457 for error responses. HTTP status codes: `200`, `400`, `403`, `404`, `409`, `422`, `500`.

**MCP Server** — HTTP/SSE transport. Business errors use `isError: true`. Protocol errors use MCP protocol errors.

See @skills/tmdb-api/SKILL.md for response shapes and examples.

---

## Deployment and logging

See @skills/docker/SKILL.md for docker-compose, volumes, environment variables, and logging configuration.
