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

The release group tag (e.g. the `-GROUP` suffix common in scene releases) is identified and ignored.

---

## Episode number heuristics (TV shows)

Extracting the season and episode number from a dirty filename is one of the most critical and error-prone steps. Scoutarr applies a set of heuristics in order of reliability:

**Heuristic 1 — Explicit SxEy pattern (highest confidence)**
Matches formats like `S01E02`, `S1E2`, `S01e02`, `s1e2`, etc. This is the most common and unambiguous format.

**Heuristic 2 — Compact numeric pattern with TMDB validation**
Matches formats like `102` or `1102` where the season and episode are concatenated. The correct split is determined by cross-referencing against TMDB data:
- If the series has 11+ seasons, `1102` could be season 11 episode 2.
- If season 1 has fewer than 102 episodes, `102` must be season 1 episode 2.
- If the split is still ambiguous after TMDB validation, it is treated as an error and reported to the user.

Additional heuristics will be added over time as new edge cases are identified.

---

## Confidence and automatic acceptance

- Each TMDB candidate is assigned a confidence score.
- If the top candidate exceeds the configured threshold, it is automatically accepted and the operation proceeds.
- If no candidate exceeds the threshold, the disambiguation flow is triggered.

---

## Disambiguation flow

The disambiguation behaviour differs between interfaces:

**MCP Server** — initiates a conversational disambiguation flow:
- The AI agent asks the user targeted questions to narrow down the candidates (e.g. year, original language, genre).
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
- **Genre** — the genre of the content (e.g. drama, animation, thriller).
- **candidateIndex** — 1-based index of a candidate from a previous disambiguation response. When provided, skips search and scoring and uses the selected candidate directly.

---

## No results

If TMDB returns zero results for a given query, the operation is aborted and an error is returned to the user. No disambiguation flow is initiated. The user should verify the filename or provide additional parameters and retry.
