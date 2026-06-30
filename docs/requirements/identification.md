# Identification

> Part of [Scoutarr Requirements](../REQUIREMENTS.md).

---

## Metadata source

- Primary metadata source: TMDB.
- TMDB is queried using the title and year extracted from the filename, plus any optional parameters provided by the caller.

---

## Information extracted from the filename

Before querying TMDB, Scoutarr parses the filename to extract:
- **Title** — the main name of the movie or series, required for the TMDB search.
- **Year** — used to narrow the search when present.
- **Resolution** — (e.g. 720p, 1080p, 4K) stored and available for use in the output filename template.
- **Edition** (movies only) — e.g. `Director's Cut`, `Extended`, `Unrated`, `Theatrical`. TMDB does not expose edition/alternate-version data on a single movie entry (different cuts of the same film are not modelled as distinct, queryable releases), so edition is detected entirely from filename keywords, the same approach used by TinyMediaManager. Stored and available for use in the output filename template; `null` if no recognised edition keyword is present.

The release group tag (e.g. the `-GROUP` suffix common in scene releases) is identified and ignored.

---

## Episode number heuristics (TV shows)

Extracting the season and episode number from a dirty filename is one of the most critical and error-prone steps. Scoutarr applies a chain of heuristics in order of reliability. Each heuristic is tried in turn against an `EpisodeParseContext` (filename, optional series metadata, optional file duration); the first one that produces a result wins.

**H1 — Explicit SxEy pattern (highest confidence)**
Matches formats like `S01E02`, `S1E2`, `S01e02`, `s1e2`, etc. This is the most common and unambiguous format. Uses the filename only.

**H2 — Absolute episode number**
Matches a bare number in the filename (e.g. `breaking.bad.13.mkv`), without relying on a live TMDB call. The number is resolved against the series metadata file (see [file-handling.md](file-handling.md) — "Series state"), which stores a precalculated `AbsoluteEpisodeNumber` for every episode. Two interpretations are considered — the number as an absolute episode position, or as a compact `SxEy` encoding (e.g. `213` → S02E13) — and disambiguated using episode title words from the filename when both are plausible. Requires the series metadata file; if it is not yet available, H2 returns no result and the chain continues.

**H3 — Special episode identification by file duration**
Applies only to Season 0 (specials). Matches the actual duration of the file (read via `MediaInfo.Wrapper.Core`) against `RuntimeMinutes` stored per episode in the series metadata file, with a tolerance window. Disambiguates using episode title words when more than one special matches. Requires both the series metadata file and a readable file duration.

**H4 — Episode identification by air date**
Matches a broadcast date encoded in the filename (e.g. `2013.09.29`, `2013-09-29`, `20130929`) against the `AirDate` stored per episode in the series metadata file. Common for talk shows, daily/news programmes, and sports broadcasts. Disambiguates using episode title words when more than one episode aired on the same date. Requires the series metadata file.

**H5 — Episode identification by title match**
Matches words extracted from the filename against episode titles stored in the series metadata file, requiring all significant title words to be present (retried without articles/short prepositions if the strict match fails). Applies across all seasons, including Season 0. Requires the series metadata file.

H2 through H5 all depend on the series metadata file produced for the series folder (see [file-handling.md](file-handling.md) — "Series state"); when no metadata file is available yet for a given operation, they return no result and the chain falls through. Additional heuristics may be added over time as new edge cases are identified.

---

## Confidence and automatic acceptance

- Each TMDB candidate is assigned a confidence score.
- If the top candidate exceeds the configured threshold, it is automatically accepted and the operation proceeds.
- If no candidate exceeds the threshold, the disambiguation flow is triggered.

---

## Disambiguation flow

The disambiguation behaviour differs between interfaces:

**MCP Server** — initiates a conversational disambiguation flow:
- The AI agent asks the user targeted questions to narrow down the candidates (e.g. year, original language).
- The process continues iteratively until a candidate exceeds the confidence threshold.
- The user can cancel the operation at any point.
- If the user cancels, the operation is aborted and no files are modified.

**REST API** — cannot support interactive disambiguation. When no candidate exceeds the confidence threshold, the API returns the top 10 candidates ranked by confidence score, along with their TMDB metadata (title, year, overview, original language), so the caller can present them to the user and retry.

To select a specific candidate from a previous response, the caller repeats the same request and includes the `candidateIndex` parameter — a 1-based index into the candidate list returned by the previous call. For example, if the previous response returned 7 candidates and the user selects the third one, the caller retries with `candidateIndex: 3`. When `candidateIndex` is provided, the confidence threshold is bypassed and that candidate is used directly.

---

## Optional parameters

The caller may provide the following optional parameters at request time to assist identification:
- **Year** — the release year of the movie or series.
- **Original language** — the original language of the content (e.g. `en`, `es`, `ja`).
- **candidateIndex** — 1-based index of a candidate from a previous disambiguation response. When provided, skips search and scoring and uses the selected candidate directly.

---

## No results

If TMDB returns zero results for a given query, the operation is aborted and an error is returned to the user. No disambiguation flow is initiated. The user should verify the filename or provide additional parameters and retry.
