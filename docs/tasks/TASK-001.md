# TASK-001 — CI/CD pipeline (GitHub Actions + ghcr.io)

**Architecture:** [ARCHITECTURE.md](../ARCHITECTURE.md) — sections 4, 6
**Depends on:** [TASK-000](TASK-000.md) — solution must compile, tests must pass, Dockerfile must exist

---

## Context

This task automates the build, test, and publish lifecycle using GitHub Actions. The repository is public, so Actions minutes are unlimited.

Three workflows following the branching strategy defined in ARCHITECTURE.md section 4:
- **CI** — runs on every pull request targeting `dev`; must pass before merging.
- **CD edge** — runs on every merge to `dev`; builds and publishes the `edge` image for local pre-release testing.
- **CD release** — runs when a `v*` tag is pushed to `main`; builds and publishes the versioned release image.

E2E tests run as part of CD, not CI, because they require a built Docker image.

---

## Deliverables

### 1. CI workflow — `.github/workflows/ci.yml`

Trigger: `pull_request` targeting `dev`.

Steps:
1. Checkout
2. Setup .NET (latest LTS — match the version used in the Dockerfile)
3. `dotnet restore Scoutarr.sln`
4. `dotnet build Scoutarr.sln --no-restore --configuration Release`
5. `dotnet test Scoutarr.sln --no-build --configuration Release` — runs unit and integration tests only (`Scoutarr.Core.Tests`, `Scoutarr.Api.Tests`, `Scoutarr.Mcp.Tests`); excludes `Scoutarr.E2E.Tests`

The CI workflow must not push any image or artifact.

### 2. CD edge workflow — `.github/workflows/cd-edge.yml`

Trigger: `push` to `dev`.

Steps:
1. Checkout
2. Setup .NET (latest LTS)
3. `dotnet restore Scoutarr.sln`
4. `dotnet build Scoutarr.sln --no-restore --configuration Release`
5. `dotnet test Scoutarr.sln --no-build --configuration Release` — unit and integration tests only; fail fast if any test fails
6. Log in to `ghcr.io` using `GITHUB_TOKEN`
7. `docker build -t ghcr.io/javixulo/scoutarr:edge -t ghcr.io/javixulo/scoutarr:<sha> .`
8. `docker push ghcr.io/javixulo/scoutarr:edge`
9. `docker push ghcr.io/javixulo/scoutarr:<sha>`
10. Run E2E tests (`Scoutarr.E2E.Tests`) against the locally built image

### 3. CD release workflow — `.github/workflows/cd-release.yml`

Trigger: `push` of a tag matching `v*` (e.g. `v0.1.0`).

Steps:
1. Checkout
2. Setup .NET (latest LTS)
3. `dotnet restore Scoutarr.sln`
4. `dotnet build Scoutarr.sln --no-restore --configuration Release`
5. `dotnet test Scoutarr.sln --no-build --configuration Release` — unit and integration tests only; fail fast if any test fails
6. Log in to `ghcr.io` using `GITHUB_TOKEN`
7. `docker build -t ghcr.io/javixulo/scoutarr:<tag> -t ghcr.io/javixulo/scoutarr:latest .` — `<tag>` is the pushed git tag (e.g. `v0.1.0`)
8. `docker push ghcr.io/javixulo/scoutarr:<tag>`
9. `docker push ghcr.io/javixulo/scoutarr:latest`
10. Run E2E tests (`Scoutarr.E2E.Tests`) against the locally built image
11. If E2E tests fail, the workflow fails but the image has already been pushed. Document in a comment in the workflow how to handle a bad release (e.g. retag `latest` to the previous SHA manually).

### 4. Branch protection

Document in a comment in `ci.yml` that the following branch protection rules should be enabled by the project owner (cannot be set via workflow files):
- On `dev`: require CI to pass before merging any PR
- On `main`: allow only project owner to push or merge; no direct pushes
- Require branches to be up to date before merging

---

## Notes for Jarvis

- Use `actions/checkout@v4` and `actions/setup-dotnet@v4` (or latest stable at time of implementation).
- Pin all third-party actions to a specific SHA, not a mutable tag, to avoid supply chain issues.
- The `GITHUB_TOKEN` is automatically available in Actions — no manual secret setup needed for pushing to `ghcr.io` on a public repo.
- To exclude E2E tests from CI runs, use `--filter 'FullyQualifiedName!~E2E'` or a test category attribute — coordinate with Black Widow on which approach is used in `Scoutarr.E2E.Tests`.
- The Docker image name is `ghcr.io/javixulo/scoutarr` (lowercase owner, lowercase repo).
- The E2E tests in CD need a `TMDB_API_KEY` — add it as a repository secret named `TMDB_API_KEY_TEST` and pass it to the test run via environment variable.
- Verify the `edge` image is publicly visible at `ghcr.io/javixulo/scoutarr:edge` after the first CD edge run.

---

## Acceptance criteria (Gherkin)

```gherkin
Feature: CI/CD pipeline
  As any agent or project owner
  I want automated build, test, and publish workflows
  So that every PR is validated and every merge produces the correct image

  Scenario: CI passes on a clean PR to dev
    Given a pull request targeting dev
    When the CI workflow runs
    Then dotnet restore, build, and unit+integration tests all pass
    And no Docker image is built or pushed

  Scenario: CI fails if any unit or integration test fails
    Given a pull request targeting dev
    And at least one unit or integration test is failing
    When the CI workflow runs
    Then the CI workflow exits with a failure status
    And the PR cannot be merged

  Scenario: CD edge publishes edge image on merge to dev
    Given a merge to dev with all tests passing
    When the CD edge workflow runs
    Then the Docker image is built
    And pushed to ghcr.io/javixulo/scoutarr:edge
    And pushed to ghcr.io/javixulo/scoutarr:<sha>
    And both tags are publicly accessible

  Scenario: CD release publishes versioned image on v* tag
    Given a v* tag pushed to main
    When the CD release workflow runs
    Then the Docker image is built
    And pushed to ghcr.io/javixulo/scoutarr:<tag>
    And pushed to ghcr.io/javixulo/scoutarr:latest
    And both tags are publicly accessible

  Scenario: CD fails if E2E tests fail
    Given a merge to dev or a v* tag push
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
- [ ] Jarvis writes `.github/workflows/cd-edge.yml`
- [ ] Jarvis writes `.github/workflows/cd-release.yml`
- [ ] Project owner adds `TMDB_API_KEY_TEST` as a repository secret
- [ ] Jarvis verifies CI passes on a test PR to dev
- [ ] Jarvis verifies CD edge publishes `edge` image on merge to dev
- [ ] Jarvis verifies CD release publishes versioned image on a `v*` tag
- [ ] Project owner enables branch protection rules on `dev` and `main`
- [ ] Hawkeye reviews
