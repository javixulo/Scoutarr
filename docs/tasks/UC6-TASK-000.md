# UC6-TASK-000 — Resolve the series for a remap operation

**Requirements:** [requirements/file-handling.md](../requirements/file-handling.md) — "Remapping working file"
**Dependencies:** UC2-TASK-004 (TV show identification service, Core)

---

## Context

Before validating any remap destination, Scoutarr needs to know which series a given file belongs to. This task does not introduce new identification logic — it applies the TV series identification flow already built and tested for UC-02, unchanged, to the remap entry point.

- If the series folder contains a metadata file, its stored TMDB ID is used directly — no new TMDB search is performed.
- If it does not, the existing identification flow (`ITvShowIdentificationService`, UC2-TASK-004) is invoked, using the same fallback chain and disambiguation behaviour it already has (folder name first, filename if the folder name doesn't yield a confident match, conversational/candidate-list disambiguation if still ambiguous, error if it cannot be resolved at all).

No new acceptance criteria are defined here — the identification behaviour itself is already covered by the UC-02 task suite. This task only covers wiring the remap entry point to call that existing service correctly, with the metadata-file shortcut in front of it.

---

## Notes for Black Widow

- No new test scenarios for the identification behaviour itself (already covered by UC-02 tests).
- Add tests confirming: when a metadata file is present, `ITvShowIdentificationService` is never called; when absent, it is called with the folder path as input.

## Notes for Tony Stark

- This is thin wiring, not a new service. Reuse `ITvShowIdentificationService` as-is.
- Read the metadata file (if present) via the existing series metadata read logic (UC3-TASK-002) to short-circuit identification.

---

## Subtasks

- [ ] Wire remap entry point to check for an existing metadata file first
- [ ] Fall back to `ITvShowIdentificationService` when no metadata file is present
- [ ] Black Widow writes tests for the wiring described above
- [ ] Tony Stark implements the wiring
- [ ] Hawkeye reviews
