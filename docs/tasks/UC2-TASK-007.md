# UC2-TASK-007 — Output filename formatter (TV episode)

**Requirements:** [requirements/configuration.md](../requirements/configuration.md) — "Naming format templates"
**Dependencies:** UC2-TASK-006 (TvEpisodeLookupSuccess)

---

## Context

Given a confirmed TV episode result and a naming template, produce a clean output filename stem (without extension). The template uses `{token}` syntax. The optional `{resolution}` token is omitted — including any immediately preceding separator — when the value is not available.

This task is the TV equivalent of `IMovieFilenameFormatter` from TASK-003. The separator omission rule and template mechanics are identical — only the tokens differ.

---

## Input

```
TvFormatterInput
{
    SeriesName:    string
    Season:        int
    Episode:       int
    EpisodeTitle:  string
    Resolution:    string?
    Template:      string
}
```

## Output

```
string  // formatted filename stem, no extension
```

---

## Token reference

| Token | Source | Optional |
|---|---|---|
| `{series}` | TMDB series name | No |
| `{season:00}` | Season number, zero-padded to 2 digits | No |
| `{episode:00}` | Episode number, zero-padded to 2 digits | No |
| `{title}` | TMDB episode title | No |
| `{resolution}` | Parsed from filename | Yes |

**Zero-padding rule:** `{season:00}` and `{episode:00}` are always zero-padded to at least 2 digits.

**Co-presence rule:** `{season:00}` and `{episode:00}` must both appear in the template.

---

## Subtasks

- [ ] Define `TvFormatterInput` record in `Scoutarr.Core`
- [ ] Define `ITvFilenameFormatter` interface in `Scoutarr.Core`
- [ ] Evaluate whether separator omission and template parsing can be shared with `MovieFilenameFormatter`
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `TvFilenameFormatter`
- [ ] Hawkeye reviews
