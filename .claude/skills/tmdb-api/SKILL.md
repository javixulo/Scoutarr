---
name: TMDB API
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

## Response structure — TV show search

```json
{
  "results": [
    {
      "id": 1396,
      "name": "Breaking Bad",
      "original_name": "Breaking Bad",
      "original_language": "en",
      "first_air_date": "2008-01-20",
      "overview": "...",
      "genre_ids": [18, 80],
      "popularity": 340.2
    }
  ],
  "total_results": 5
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

---

## Movie enrichment endpoint

Used during disambiguation only — when no candidate exceeds the confidence threshold.

```
GET /movie/{movie_id}?append_to_response=credits
```

Returns full movie details plus cast and crew in a single request.

### Relevant fields from enriched response

```json
{
  "id": 414906,
  "title": "The Batman",
  "release_date": "2022-03-01",
  "overview": "...",
  "genres": [
    { "id": 28, "name": "Action" }
  ],
  "belongs_to_collection": {
    "id": 263,
    "name": "The Dark Knight Collection"
  },
  "credits": {
    "cast": [
      { "name": "Robert Pattinson", "character": "Bruce Wayne / Batman", "order": 0 },
      { "name": "Zoë Kravitz", "character": "Selina Kyle / Catwoman", "order": 1 }
    ],
    "crew": [
      { "name": "Matt Reeves", "job": "Director" }
    ]
  }
}
```

### When to call this

Only when `MovieSearchResult` contains candidates all below the confidence threshold. Enrich the top N candidates (max 10) before returning the disambiguation result to the MCP layer.

### What to extract for disambiguation

| Field | Use |
|---|---|
| `credits.crew` where `job = "Director"` | "Is this the one directed by X?" |
| `credits.cast` top 3 by `order` | "Is this the one with X as Y?" |
| `belongs_to_collection.name` | "Is this part of the X collection?" |
| `genres` | "Is this a drama or an action film?" |
| `overview` | Context for the agent to reason about |

### Mocking in tests

Mock `ITmdbClient.GetMovieDetailsAsync(int movieId)` — never call the real API. Use realistic enriched response structures matching the shape above.

---

## TV show enrichment endpoint

Used during disambiguation only — when no TV show candidate exceeds the confidence threshold. Same pattern as movie enrichment.

```
GET /tv/{tv_id}?append_to_response=credits
```

Returns full TV show details plus aggregate cast and crew in a single request.

### Relevant fields from enriched response

```json
{
  "id": 1396,
  "name": "Breaking Bad",
  "original_name": "Breaking Bad",
  "original_language": "en",
  "first_air_date": "2008-01-20",
  "status": "Ended",
  "overview": "...",
  "genres": [
    { "id": 18, "name": "Drama" },
    { "id": 80, "name": "Crime" }
  ],
  "networks": [
    { "name": "AMC" }
  ],
  "number_of_seasons": 5,
  "credits": {
    "cast": [
      { "name": "Bryan Cranston", "character": "Walter White", "order": 0 },
      { "name": "Aaron Paul", "character": "Jesse Pinkman", "order": 1 }
    ],
    "crew": [
      { "name": "Vince Gilligan", "job": "Creator" }
    ]
  }
}
```

### When to call this

Only when `TvShowSearchResult` contains candidates all below the confidence threshold. Enrich the top N candidates (max 10) before returning the disambiguation result to the MCP layer.

### What to extract for disambiguation

| Field | Use |
|---|---|
| `credits.crew` where `job = "Creator"` | "Is this the one created by X?" |
| `credits.cast` top 3 by `order` | "Is this the one with X as Y?" |
| `networks` first entry | "Is this the AMC show?" |
| `genres` | "Is this a drama or a comedy?" |
| `status` | "Is this a show that has already ended?" |
| `number_of_seasons` | "Is this the one with 5 seasons?" |
| `overview` | Context for the agent to reason about |

### Mocking in tests

Mock `ITmdbClient.GetTvShowDetailsAsync(int tvId)` — never call the real API. Use realistic enriched response structures matching the shape above.
