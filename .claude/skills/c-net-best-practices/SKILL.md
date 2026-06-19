---
name: c-net-best-practices
description: Use this skill when writing or reviewing any C# implementation code in this project.
---

# C# / .NET Best Practices Skill

## General principles

- Write the minimum code necessary to make tests pass. No speculative abstractions.
- Prefer explicit over implicit. Avoid magic strings, magic numbers, and implicit conversions.

---

## Async/await

- All I/O operations (file system, HTTP calls to TMDB) must be async.
- Method signatures: `Task<T>` for async methods that return a value, `Task` for void async.
- Never use `.Result` or `.Wait()` — always `await`.
- Use `CancellationToken` in all async public methods.

```csharp
// Correct
public async Task<RenameResult> RenameMovieAsync(string filePath, CancellationToken ct = default)

// Wrong
public RenameResult RenameMovie(string filePath)
{
    return _tmdbClient.SearchMovieAsync(title).Result;
}
```

---

## Error handling

- Domain errors are custom exception types defined in `Scoutarr.Core`.
- Never throw generic `Exception`. Use specific domain exceptions.
- Never swallow exceptions silently.
- Controllers and MCP tools catch domain exceptions and map them to their respective response formats.

```csharp
// Domain exceptions in Scoutarr.Core
public class FileNotFoundException : ScoutarrException { }
public class NoTmdbMatchException : ScoutarrException { }
public class InsufficientPermissionsException : ScoutarrException { }
```

---

## Immutability

- Prefer immutable models. Use `record` types for DTOs and domain models.

```csharp
public record RenameResult(
    string OriginalFilename,
    string NewFilename,
    TmdbMatch Match
);
```

---

## Task comments

Every implemented block must include a comment referencing the originating feature and task:

```csharp
// FEATURE-001-TASK-001: Filename parser — SxEy pattern detection
public SeasonEpisode? ParseSxEyPattern(string filename) { ... }
```
