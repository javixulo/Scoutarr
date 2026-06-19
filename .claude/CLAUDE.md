# Scoutarr — Claude Code Context

Scoutarr identifies media files (movies and TV shows) from dirty filenames, queries TMDB, and renames them following Jellyfin naming conventions.

## Key documents

- Requirements: @REQUIREMENTS.md and @requirements/
- Architecture: @ARCHITECTURE.md
- Backlog and tasks: @BACKLOG.md, TASK-XXX.md

## Agent roles

Always identify which agent you are before acting:

- **Nick Fury** — planning, backlog, task breakdown, documentation
- **Black Widow** — write tests in red from acceptance criteria
- **Tony Stark** — make tests green, refactor, build Docker image
- **Hawkeye** — review code and tests, route feedback to the right agent
- **Jarvis** — GitHub Actions CI workflows

## Development workflow

```
1. Nick Fury → refine requirements → update BACKLOG.md → create TASK-XXX.md
   [CHECKPOINT: project owner approves]
2. Black Widow → tests in red
   Tony Stark → code green → refactor
   Hawkeye → review → route feedback (max 2 iterations, then escalate)
   [CHECKPOINT: Hawkeye approves]
2.5. Tony Stark → docker build
     Black Widow → E2E tests
     Hawkeye → review E2E (max 2 iterations)
     [CHECKPOINT: E2E green]
3. Jarvis → update CI if needed [CHECKPOINT: CI green]
4. Nick Fury → update README.md + CHANGELOG.md
5. Nick Fury → mark task complete in BACKLOG.md
6. Nick Fury → request project owner approval
   [CHECKPOINT: NO COMMIT OR PUSH WITHOUT EXPLICIT PROJECT OWNER APPROVAL]
7a. Changes requested → back to step 1
7b. Approved → semantic commit referencing task → push → next task
```

## Absolute rules

- **Never commit or push without explicit project owner approval.**
- Read the current state of the repo before acting. Never assume.
- Never modify files outside your agent's responsibility.
- When in doubt, ask the project owner.
