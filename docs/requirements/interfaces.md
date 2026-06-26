# Interfaces

> Part of [Scoutarr Requirements](../REQUIREMENTS.md).

---

## Overview

Scoutarr exposes two interfaces that cover the same core functionality:

- **REST API** — for integration with external tools, scripts, and direct HTTP calls.
- **MCP Server** — for integration with AI agents (Claude, Cursor, etc.).

---

## Responses and notifications

Both interfaces must return structured, informative responses for every operation — whether successful or failed. Responses are never silent.

### Success responses must include

- The original filename (or folder name).
- The suggested filename (or folder name) — in Rename mode this is the actual new name on disk; in Identify mode it is the suggested name only.
- The mode used (identify or rename).
- The TMDB match used (title, year, ID).
- Confidence score of the match.


### Error responses must include

- A clear, human-readable description of what went wrong.
- The specific file or folder that caused the error.
- Actionable information where possible (e.g. what the user can do to resolve it).

### Errors that must be explicitly handled and reported

- File or folder not found.
- Insufficient write permissions on the target path.
- A file with the target name already exists and cannot be overwritten.
- No TMDB match found above the confidence threshold.
- TMDB API unreachable or returning errors.
- Episode number does not exist in the known series data (e.g. S01E22 when season 1 only has 21 episodes).
- Series metadata file is present but malformed or unreadable.

> For the technical implementation of response formats (RFC 9457 for REST, `isError` for MCP), see [ARCHITECTURE.md](../ARCHITECTURE.md).

---

## MCP Server

The intended interaction model is conversational: an AI agent calls Scoutarr on behalf of the user, receives a structured response, and communicates the outcome naturally. For example:

- On success: the agent informs the user of the new filename and the matched title.
- On error: the agent explains what went wrong and asks the user how to proceed.
- On low confidence: the agent initiates the disambiguation flow with the user.

---

## REST API

The REST API is designed for direct HTTP calls from scripts, tools, or other services that do not use an AI agent. It does not support interactive disambiguation — when confidence is below the threshold, it returns the top 10 candidates ranked by confidence score so the caller can present them to the user and retry.
