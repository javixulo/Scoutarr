# UC5-TASK-002 — H3: Special episode identification by file duration

**Requirements:** [requirements/identification.md](../requirements/identification.md) — "Episode number heuristics"
**Dependencies:** UC2-TASK-005 (IEpisodeHeuristic, EpisodeParseContext, TvEpisodeNumberParser), UC3-TASK-002 (SeriesMetadata, ISeriesMetadataService)

---

## Context

Some TV series include specials — Season 0 episodes — that are distributed without a clear `SxxEyy` marker in the filename. H3 targets this case specifically by matching the file's actual duration against the `RuntimeMinutes` stored in `SeriesMetadata` for each Season 0 episode.

Examples of filenames this heuristic targets:

- `breaking.bad.special.mkv` — no episode marker, duration matches a known special
- `breaking.bad.behind.the.scenes.mkv` — title words present, duration confirms

H3 **only** applies to Season 0. If the filename contains an explicit season token for any season other than S00 (e.g. `S02`, `2x`, `Season 2`), H3 returns `null` immediately.

H3 relies on:
- `context.FileDuration` — the actual duration of the file, read upstream by the orchestrator using `MediaInfo.Wrapper.Core` before invoking the heuristic chain.
- `context.Metadata` — the series metadata file written by UC3-TASK-002, which contains `RuntimeMinutes` per episode.

If either is unavailable, H3 returns `null` and the chain continues.

This task implements `IEpisodeHeuristic` and registers it as H3 in the heuristic chain in `TvEpisodeNumberParser`.

---

## Resolution logic

### Step 1 — Guard checks

- If `context.Metadata` is null → return `null`.
- If `context.FileDuration` is null → return `null`.
- If `context.Metadata` has no Season 0 → return `null`.
- If `context.Filename` contains an explicit season token for a season other than S00 → return `null`.

Explicit season tokens that trigger the guard: `S01`–`S99`, `1x`–`99x`, `Season 1`–`Season 99` (case-insensitive). `S00` and `0x` and `Season 0` do **not** trigger the guard.

### Step 2 — Match by duration

For each episode in Season 0 of the metadata:
- Compute the tolerance window: `[RuntimeMinutes, RuntimeMinutes + 7]` minutes.
- The file duration (converted to minutes, rounded down) must fall within this window.
- Collect all episodes that match.

### Step 3 — Resolve

- If exactly one episode matches → proceed to Step 4 (title confirmation).
- If multiple episodes match → attempt title disambiguation (Step 4) across all candidates.
- If no episodes match → return `null`.

### Step 4 — Title confirmation / disambiguation

Extract candidate words from the filename by removing:
- Common noise tokens (resolution tags, release group suffixes, file extension, the series title tokens)
- The explicit season token if present (e.g. `S00`)

Match the remaining words against the episode title of each candidate using case-insensitive token matching. A match is counted if at least one meaningful word (> 3 characters) from the filename appears in the episode title.

- Single duration match + title confirms → return that episode.
- Single duration match + title does not match → return that episode anyway (duration alone is sufficient).
- Multiple duration matches + exactly one title matches → return that episode.
- Multiple duration matches + title matches none or more than one → return `null`.

---

## Library

File duration is read upstream by the orchestrator (not inside the heuristic itself) using:

```
MediaInfo.Wrapper.Core (NuGet: MediaInfo.Wrapper.Core)
```

Cross-platform (.NET 6+, Linux/Docker). On Linux, requires system packages:
```
apt-get install libzen0v5 libmms0 zlib1g libnghttp2-14 librtmp1 libcurl4
```

These must be included in the Docker image (see TASK-023).

Usage in the orchestrator:
```csharp
using var media = new MediaInfoWrapper(filePath);
var duration = media.Success ? TimeSpan.FromMilliseconds(media.Duration) : (TimeSpan?)null;
```

The heuristic itself receives `context.FileDuration` as a `TimeSpan?` and performs no I/O.

---

## Interface

H3 implements `IEpisodeHeuristic` from UC2-TASK-005:

```
IEpisodeHeuristic

TryParse(EpisodeParseContext context)
    → ParsedEpisodeNumber?
```

`TvEpisodeNumberParser` injects H3 after H2 in its ordered heuristic list.

---

## Notes for Black Widow

- All tests are pure — populate `EpisodeParseContext` directly with mock `SeriesMetadata` and a `TimeSpan?`. No filesystem or MediaInfo calls.
- Test null metadata → `null`.
- Test null file duration → `null`.
- Test metadata with no Season 0 → `null`.
- Test filename with explicit non-S00 season token → `null`.
- Test filename with explicit S00 token → does NOT block H3, continues normally.
- Test duration within tolerance window → match.
- Test duration exactly at upper bound of window (RuntimeMinutes + 7) → match.
- Test duration one minute above upper bound → no match.
- Test single duration match, no title words → returns that episode.
- Test single duration match, title confirms → returns that episode.
- Test multiple duration matches, title disambiguates → returns matching episode.
- Test multiple duration matches, title does not disambiguate → `null`.
- Test H3 is registered after H2 in `TvEpisodeNumberParser` — verify ordering in unit tests.

## Notes for Tony Stark

- Implement `SpecialEpisodeDurationHeuristic` in `Scoutarr.Core`.
- `TryParse` must be synchronous — no async, no I/O.
- Duration comparison: convert `context.FileDuration` to whole minutes (`(int)fileDuration.TotalMinutes`) before comparing against `RuntimeMinutes`.
- Tolerance is **upward only**: `fileDuration_minutes >= RuntimeMinutes && fileDuration_minutes <= RuntimeMinutes + 7`.
- Season token detection must be a regex applied to the filename before any other processing.
- Register H3 immediately after H2 in the DI registration of `TvEpisodeNumberParser`.
- Add `MediaInfo.Wrapper.Core` NuGet reference to `Scoutarr.Core`.
- Add required Linux system packages to the Dockerfile (coordinate with TASK-023).

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: H3 — Special episode identification by file duration

  This heuristic only applies to Season 0 (specials). It reads the file duration
  from EpisodeParseContext and matches it against RuntimeMinutes in SeriesMetadata
  with a tolerance of +7 minutes. Title words are used to disambiguate if needed.

  Scenario: Duration matches exactly one special, no title words
    Given the series has Season 0 with 3 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 90 minutes and title "Pilot Extended"
    And special S00E03 has runtime 22 minutes and title "Mini Episode"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is S00E01

  Scenario: Duration matches exactly one special, title confirms
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 90 minutes and title "Pilot Extended"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.behind.the.scenes.mkv"
    When H3 is applied
    Then the result is S00E01

  Scenario: Duration matches multiple specials, title disambiguates
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 43 minutes and title "Pilot Extended"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.pilot.extended.mkv"
    When H3 is applied
    Then the result is S00E02

  Scenario: Duration matches multiple specials, title does not disambiguate
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 43 minutes and title "Pilot Extended"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is null

  Scenario: Duration exceeds all specials beyond tolerance
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 22 minutes and title "Mini Episode"
    And the file duration is 60 minutes
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is null

  Scenario: Filename contains explicit non-S00 season token
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.s02.episode.special.mkv"
    When H3 is applied
    Then the result is null

  Scenario: Filename contains explicit S00 token — H3 is not blocked
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And special S00E02 has runtime 90 minutes and title "Pilot Extended"
    And the file duration is 47 minutes
    And the filename is "breaking.bad.s00.special.mkv"
    When H3 is applied
    Then the result is S00E01

  Scenario: No metadata available
    Given no metadata is available for the series
    And the file duration is 47 minutes
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is null

  Scenario: Metadata has no Season 0
    Given the series metadata has no Season 0
    And the file duration is 47 minutes
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is null

  Scenario: No file duration in context
    Given the series has Season 0 with 2 specials
    And special S00E01 has runtime 45 minutes and title "Behind the Scenes"
    And the file duration is not available
    And the filename is "breaking.bad.special.mkv"
    When H3 is applied
    Then the result is null
```

---

## Subtasks

- [ ] Add `MediaInfo.Wrapper.Core` NuGet reference to `Scoutarr.Core`
- [ ] Add MediaInfo Linux system packages to Dockerfile (coordinate with TASK-023)
- [ ] Implement `SpecialEpisodeDurationHeuristic` in `Scoutarr.Core`
- [ ] Register H3 after H2 in `TvEpisodeNumberParser` DI registration
- [ ] Black Widow writes tests in red (all scenarios above)
- [ ] Tony Stark implements `SpecialEpisodeDurationHeuristic`
- [ ] Hawkeye reviews
