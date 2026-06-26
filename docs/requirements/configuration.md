# Configuration

> Part of [Scoutarr Requirements](../REQUIREMENTS.md).

---

## Overview

- Single global configuration via environment variables (Docker-friendly).
- No configuration file — all parameters are passed as environment variables, following the convention of the *arr ecosystem.

---

## Parameters

| Parameter | Description |
|---|---|
| `TMDB_API_KEY` | TMDB API key (required). |
| `SCRAPING_LANGUAGE` | Language for TMDB metadata (e.g. `en`, `es`). |
| `CONFIDENCE_THRESHOLD` | Minimum confidence score for automatic acceptance (0.0–1.0). |
| `NAMING_FORMAT_MOVIE` | Output filename template for movies. |
| `NAMING_FORMAT_EPISODE` | Output filename template for TV episodes. |

---

## Naming format templates

Templates follow a simple `{token}` syntax. Inspired by TinyMediaManager, the following tokens are available:

**Movies:**

| Token | Description |
|---|---|
| `{title}` | Movie title |
| `{year}` | Release year |
| `{resolution}` | Video resolution (e.g. 1080p, 4K) — omitted if not detected |
| `{edition}` | Edition (e.g. Director's Cut) — omitted if not available |

**TV episodes:**

| Token | Description |
|---|---|
| `{series}` | Series title |
| `{year}` | Series start year |
| `{season}` | Season number (zero-padded, e.g. 01) |
| `{episode}` | Episode number (zero-padded, e.g. 02) |
| `{title}` | Episode title |
| `{resolution}` | Video resolution (e.g. 1080p, 4K) — omitted if not detected |

**Default formats:**
- Movie: `{title} ({year})`
- Episode: `{series} - S{season}E{episode} - {title}`

> For the technical implementation of environment variables and the docker-compose example, see [ARCHITECTURE.md](../ARCHITECTURE.md).
