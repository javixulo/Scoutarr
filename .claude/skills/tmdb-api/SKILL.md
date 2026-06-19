---
name: tmdb-api
description: Use this skill when working on any task that involves calling the TMDB API, writing mocks for TMDB responses, or handling TMDB data structures.
---

# TMDB API Skill

## Base URL

```
https://api.themoviedb.org/3
```

## Authentication

All requests require the API key as a query parameter:
```
?api_key={TMDB_API_KEY}
```

The API key is provided via the `TMDB_API_KEY` environment variable. Never hardcode it.

---

## Relevant endpoints

### Search movie
```
GET /search/movie?query={title}&year={year}&language={language}
```

### Search TV show
```
GET /search/tv?query={title}&first_air_date_year={year}&language={language}
```

### Get TV show details (seasons and episodes)
```
GET /tv/{tv_id}
GET /tv/{tv_id}/season/{season_number}
```

---

## Response structure — Movie search

```json
{
  "results": [
    {
      "id": 414906,
      "title": "The Batman",
      "original_title": "The Batman",
      "original_language": "en",
      "release_date": "2022-03-01",
      "overview": "...",
      "genre_ids": [28, 80, 18],
      "popularity": 120.5
    }
  ],
  "total_results": 12
}
```

## Response structure — TV show details

```json
{
  "id": 1396,
  "name": "Breaking Bad",
  "original_language": "en",
  "first_air_date": "2008-01-20",
  "status": "Ended",
  "number_of_seasons": 5,
  "seasons": [
    {
      "season_number": 1,
      "episode_count": 7,
      "air_date": "2008-01-20"
    }
  ]
}
```

---

## Rate limits

- 50 requests per 10 seconds.
- All TMDB calls must go through a single client service (`TmdbClient` in `Scoutarr.Core`) — never call the API directly from controllers or MCP tools.

---

## Confidence score

TMDB does not return a confidence score. Scoutarr computes it internally based on:
- Title similarity between the filename and the TMDB result.
- Year match (if year is present in the filename).
- Popularity score from TMDB as a tiebreaker.

---

## Mocking in tests

In integration and unit tests, never call the real TMDB API. Always mock `ITmdbClient`. Use realistic response structures matching the shapes above.
