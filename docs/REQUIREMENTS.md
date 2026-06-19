# Scoutarr – Requirements

> For technical implementation decisions derived from these requirements, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Vision

Scoutarr is a tool that identifies media files (movies and TV shows) and renames them following naming conventions compatible with Jellyfin and similar media servers. It is designed to handle files that arrive outside of automated pipelines such as Radarr or Sonarr — downloads from direct download managers, files shared manually, or existing collections.

Scoutarr is inspired by [TinyMediaManager (TMM)](https://www.tinymediamanager.org), an open source media management tool that serves as a reference for scraping logic, naming conventions, and edge case handling. TMM source code is available at [github.com/tinymediamanager](https://github.com/tinymediamanager/tinymediamanager) and its documentation at [tinymediamanager.org/docs](https://www.tinymediamanager.org/docs/).

---

## Core Use Cases

- Rename a single movie file.
- Rename a single TV show episode file.
- Rename all episodes in a folder (full series or single season).
- Handle subsequent passes on a known series folder (new episodes arriving over time).

---

## Requirements Index

| Area | Document | Summary |
|---|---|---|
| **Identification** | [requirements/identification.md](requirements/identification.md) | TMDB scraping, filename parsing, episode heuristics, confidence, disambiguation |
| **File Handling** | [requirements/file-handling.md](requirements/file-handling.md) | Modes of operation, input types, folder structure, series state, metadata file |
| **Interfaces** | [requirements/interfaces.md](requirements/interfaces.md) | REST API and MCP Server behaviour, responses, error handling |
| **Configuration** | [requirements/configuration.md](requirements/configuration.md) | Environment variables, naming format templates |

---

## Logging

Scoutarr writes structured logs for all operations, including renames, errors, TMDB lookups, and disambiguation events. Logs must persist across container restarts.

Log entries must include:
- Timestamp.
- Severity level (INFO, WARN, ERROR).
- Operation being performed.
- File or folder affected.
- Outcome or error detail.

> For the technical implementation of logging (transports, file location, format), see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Deployment

- Distributed as a Docker image.
- Intended to be added to an existing `docker-compose.yml` alongside other media tools.
- Media folders are mounted as volumes.
- AI agent clients must be able to discover and connect to the MCP server once the container is running.

> For the technical implementation of deployment (volume structure, MCP transport, client configuration), see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Technology

- Language: C# / .NET (latest LTS).
- Cross-platform (Linux primary target).
- No graphical UI in MVP.

---

## Out of Scope (MVP)

- Graphical user interface.
- Monitoring folders for new files.
- Direct integration with Radarr, Sonarr, or other *arr applications.
- Music or other media types.
