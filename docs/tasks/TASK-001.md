# TASK-001 — CI/CD pipeline (GitHub Actions + ghcr.io)

**Architecture:** [ARCHITECTURE.md](../ARCHITECTURE.md) — sections 4, 6
**Depends on:** [TASK-000](TASK-000.md) — solution must compile, tests must pass, Dockerfile must exist

---

## Context

This task automates the build, test, and publish lifecycle using GitHub Actions. The repository is public, so Actions minutes are unlimited.

Two workflows:
- **CI** — runs on every pull request; must pass before merging.
- **CD** — runs on every merge to `main`; builds the Docker image and pushes it to GitHub Container Registry (`ghcr.io`).

E2E tests run as part of CD, not CI, because they require a built Docker image.

---

## Deliverables

### 1. CI workflow — `.github/workflows/ci.yml`

Trigger: `pull_request` targeting `main`.

Steps:
1. Checkout
2. Setup .NET (latest LTS — match the version used in the Dockerfile)
3. `dotnet restore Scoutarr.sln`
4. `dotnet build Scoutarr.sln --no-restore --configuration Release`
5. `dotnet test Scoutarr.sln --no-build --configuration Release` — runs unit and integration tests only (`Scoutarr.Core.Tests`, `Scoutarr.Api.Tests`, `Scoutarr.Mcp.Tests`); excludes `Scoutarr.E2E.Tests`

The CI workflow must not push any image or artifact.

### 2. CD workflow — `.github/workflows/cd.yml`

Trigger: `push` to `main`.

Steps:
1. Checkout
2. Setup .NET (latest LTS)
3. `dotnet restore Scoutarr.sln`
4. `dotnet build Scoutarr.sln --no-restore --configuration Release`
5. `dotnet test Scoutarr.sln --no-build --configuration Release` — unit and integration tests only; fail fast if any test fails
6. Log in to `ghcr.io` using `GITHUB_TOKEN` (no extra secrets needed for public repos)
7. `docker build -t ghcr.io/<owner>/scoutarr:<tag> .` — tag with both the git SHA and `latest`
8. `docker push ghcr.io/<owner>/scoutarr:<sha>`
9. `docker push ghcr.io/<owner>/scoutarr:latest`
10. Run E2E tests (`Scoutarr.E2E.Tests`) against the locally built image — the image is available from step 7, no pull needed
11. If E2E tests fail, the CD job fails but the image has already been pushed; Jarvis must decide whether to retag or leave `latest` as-is. Document this decision in a comment in the workflow.

### 3. Branch protection

Document in a comment in `ci.yml` that the following branch protection rules should be enabled on `main` by the project owner (cannot be set via workflow files):
- Require status checks to pass before merging: `ci` workflow
- Require branches to be up to date before merging
- Do not allow bypassing the above settings

---

## Notes for Jarvis

- Use `actions/checkout@v4` and `actions/setup-dotnet@v4` (or latest stable at time of implementation).
- Pin all third-party actions to a specific SHA, not a mutable tag, to avoid supply chain issues.
- The `GITHUB_TOKEN` is automatically available in Actions — no manual secret setup needed for pushing to `ghcr.io` on a public repo.
- To exclude E2E tests from CI/unit runs, use `--filter 'FullyQualifiedName!~E2E'` or a test category attribute — coordinate with Black Widow on which approach is used in `Scoutarr.E2E.Tests`.
- The Docker image tag should be `ghcr.io/javixulo/scoutarr` (lowercase owner, lowercase repo).
- Verify the published image is publicly visible at `ghcr.io/javixulo/scoutarr:latest` after the first CD run.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: CI/CD pipeline
  As any agent or project owner
  I want automated build, test, and publish workflows
  So that every PR is validated and every merge to main produces a published image

  Scenario: CI passes on a clean PR
    Given a pull request targeting main
    When the CI workflow runs
    Then dotnet restore, build, and unit+integration tests all pass
    And no Docker image is built or pushed

  Scenario: CI fails if any unit or integration test fails
    Given a pull request targeting main
    And at least one unit or integration test is failing
    When the CI workflow runs
    Then the CI workflow exits with a failure status
    And the PR cannot be merged

  Scenario: CD publishes image on merge to main
    Given a merge to main with all tests passing
    When the CD workflow runs
    Then the Docker image is built
    And pushed to ghcr.io/javixulo/scoutarr with both the git SHA tag and latest
    And the image is publicly accessible

  Scenario: CD fails if E2E tests fail
    Given a merge to main
    And the E2E tests fail against the built image
    When the CD workflow runs
    Then the CD workflow exits with a failure status

  Scenario: CI workflow does not require any manual secrets
    Given the repository has no manually configured secrets
    When the CI workflow runs on a pull request
    Then it completes successfully using only GITHUB_TOKEN
```

---

## Subtasks

- [ ] Jarvis writes `.github/workflows/ci.yml`
- [ ] Jarvis writes `.github/workflows/cd.yml`
- [ ] Jarvis verifies CI passes on a test PR
- [ ] Jarvis verifies CD publishes image to ghcr.io on a merge to main
- [ ] Project owner enables branch protection rules on main
- [ ] Hawkeye reviews
