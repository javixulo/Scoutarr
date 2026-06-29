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
    SeriesYear:    int       // Series start year — taken from TvShowIdentificationSuccess, never from episode air date
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

| Token | Source | Notes | Optional |
|---|---|---|---|
| `{series}` | TMDB series name | — | No |
| `{year}` | Series start year | Always the year the series first aired. Source: `TvShowIdentificationSuccess.Year`, **not** the episode air date. | No |
| `{season:00}` | Season number, zero-padded to 2 digits | — | No |
| `{episode:00}` | Episode number, zero-padded to 2 digits | — | No |
| `{title}` | TMDB episode title | — | No |
| `{resolution}` | Parsed from filename | — | Yes |

**Zero-padding rule:** `{season:00}` and `{episode:00}` are always zero-padded to at least 2 digits.

**Co-presence rule:** `{season:00}` and `{episode:00}` must both appear in the template.

---

## Test scenarios for `{year}`

Black Widow must include the following scenarios explicitly, to guard against confusion between series start year and episode air year:

- Breaking Bad S05E16 ("Felina", aired 2013): `{year}` must produce `2008` (series start year), not `2013`.
- Game of Thrones S08E06 (aired 2019): `{year}` must produce `2011` (series start year), not `2019`.
- A series with a single season where start year and episode air year coincide: confirm `{year}` still reads from `SeriesYear` field, not from air date.

---

## Subtasks

- [ ] Define `TvFormatterInput` record in `Scoutarr.Core` with `SeriesYear` field (not `Year`) to make the intent unambiguous at call sites
- [ ] Define `ITvFilenameFormatter` interface in `Scoutarr.Core`
- [ ] Evaluate whether separator omission and template parsing can be shared with `MovieFilenameFormatter`
- [ ] Black Widow writes tests in red (all scenarios above, including the `{year}` = series start year scenarios)
- [ ] Tony Stark implements `TvFilenameFormatter`
- [ ] Hawkeye reviews
