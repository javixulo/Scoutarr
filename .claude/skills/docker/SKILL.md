---
name: docker
description: Use this skill when working on the Dockerfile, docker-compose configuration, building the image, or running E2E tests against the container.
---

# Docker Skill

## Image structure

Scoutarr runs as a single container exposing both the REST API and the MCP Server on the same port.

Base image: `mcr.microsoft.com/dotnet/aspnet` (official Microsoft ASP.NET runtime image).

---

## Volume structure

```
/config     ← configuration and logs (mounted from host)
    logs/
        scoutarr.log
/media      ← media files to be renamed (mounted from host)
```

The container must not write anywhere outside these two volumes.

---

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `TMDB_API_KEY` | Yes | — | TMDB API key |
| `SCRAPING_LANGUAGE` | No | `en` | Language for TMDB metadata |
| `CONFIDENCE_THRESHOLD` | No | `0.80` | Minimum confidence for auto-accept (0.0–1.0) |
| `NAMING_FORMAT_MOVIE` | No | `{title} ({year})` | Output filename template for movies |
| `NAMING_FORMAT_EPISODE` | No | `{series} - S{season}E{episode} - {title}` | Output filename template for episodes |

---

## docker-compose.yml example

```yaml
services:
  scoutarr:
    image: scoutarr:latest
    container_name: scoutarr
    environment:
      - TMDB_API_KEY=your_key_here
      - SCRAPING_LANGUAGE=en
      - CONFIDENCE_THRESHOLD=0.80
      - NAMING_FORMAT_MOVIE={title} ({year})
      - NAMING_FORMAT_EPISODE={series} - S{season}E{episode} - {title}
    volumes:
      - ./config:/config
      - /your/media:/media
    ports:
      - "8080:8080"
    restart: unless-stopped
```

---

## MCP client configuration (Claude Desktop)

```json
{
  "mcpServers": {
    "scoutarr": {
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

---

## Building the image

```bash
docker build -t scoutarr:latest .
```

Always build from the root of the repository where the `Dockerfile` lives.

---

## Running E2E tests

E2E tests require a running container. The test setup must:

1. Build the image.
2. Start the container with a temporary config volume and a test media directory.
3. Run the tests against the live container.
4. Stop and remove the container after tests complete.

Use a dedicated `docker-compose.e2e.yml` for E2E test runs, separate from the production compose file.

---

## Logs

Logs are written to both stdout and `/config/logs/scoutarr.log`. To check logs from outside the container:

```bash
# Via Docker
docker logs scoutarr

# Via mounted volume
cat ./config/logs/scoutarr.log
```
