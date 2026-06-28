# UC2-TASK-005 — TV episode number parser

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-004 (TvShowIdentificationSuccess)

---

## Context

Given a dirty TV episode filename, extract the season number and episode number.

The parser is designed as a chain of heuristics. Each heuristic implements a common interface and is tried in priority order — the first one that produces a result wins.

**This task implements Heuristic 1 only.** Heuristics 2 and beyond are out of scope here and are defined in UC-05.

---

## Output types

```
ParsedEpisodeNumber
{
    Season:   int
    Episode:  int
}
```

On failure: a domain error when no heuristic produces a result.

---

## Interface design

```
EpisodeParseContext
{
    Filename:     string
    Metadata:     SeriesMetadata?   // null if not yet available
    FileDuration: TimeSpan?         // null if not available or not yet read
}

IEpisodeHeuristic

TryParse(EpisodeParseContext context)
    → ParsedEpisodeNumber?

ITvEpisodeNumberParser

ParseAsync(EpisodeParseContext context, CancellationToken ct)
    → ParsedEpisodeNumber | TvEpisodeParseError
```

`TvEpisodeNumberParser` holds an ordered `IReadOnlyList<IEpisodeHeuristic>` injected via constructor.

Heuristics that do not need `Metadata` or `FileDuration` simply ignore those fields. This allows the chain to be extended with context-aware heuristics (e.g. duration-based, metadata-based) without modifying the interface.

---

## Heuristic 1 — Explicit SxEy pattern

Matches `S01E02`, `S1E2`, `s01e02`, `s1e2`, and similar variants. Case-insensitive.

Only uses `context.Filename`. Ignores `Metadata` and `FileDuration`.

---

## Subtasks

- [ ] Define `EpisodeParseContext` record in `Scoutarr.Core`
- [ ] Define `IEpisodeHeuristic` interface in `Scoutarr.Core`
- [ ] Define `ParsedEpisodeNumber` record in `Scoutarr.Core`
- [ ] Define `ITvEpisodeNumberParser` interface in `Scoutarr.Core`
- [ ] Implement `SxEyHeuristic`
- [ ] Implement `TvEpisodeNumberParser` with heuristic chain
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `SxEyHeuristic` and `TvEpisodeNumberParser`
- [ ] Hawkeye reviews
