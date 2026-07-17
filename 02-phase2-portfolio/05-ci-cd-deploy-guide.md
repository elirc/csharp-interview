# CI/CD + Deploy Guide

Goal: PRs run build + tests automatically; main deploys to a public URL. One day for CI, one for deploy.

## Part 1 — GitHub Actions CI

`.github/workflows/ci.yml` in the Bookable repo:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: bookable_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 5s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"
      - run: dotnet restore
      - run: dotnet build --no-restore -warnaserror
      - run: dotnet test --no-build --logger "trx"
        env:
          ConnectionStrings__Default: "Host=localhost;Database=bookable_test;Username=postgres;Password=testpass"
```

Notes worth understanding (not just pasting):
- **`services:`** spins up a throwaway Postgres next to the runner — your integration tests run against a real database in CI, same as locally. Interview line: *"my integration tests run against real Postgres in CI, not a mock."*
- **`ConnectionStrings__Default`** — double underscore maps env vars into .NET config hierarchy (concept #17 in action).
- **`-warnaserror`** — cheap rigor signal; fix warnings as they appear rather than letting them rot.
- Then: repo Settings → Branches → protect `main`, require the check to pass.

**Debugging CI is a skill.** Your first run will probably fail (connection string, migration not applied, timezone of the runner). Read the logs yourself before asking Claude — CI-log literacy comes up in "how do you debug a failing pipeline" questions.

Apply migrations in tests via `WebApplicationFactory` startup (`db.Database.Migrate()` in a test fixture) — simplest correct option at this scale.

## Part 2 — Deploy

**Recommended: Railway** (fastest path, free tier, Postgres add-on) or **Fly.io**. Azure App Service if you specifically target Azure-shop jobs — heavier setup, resume-relevant name.

Railway path:
1. Dockerfile for the API (multi-stage: `sdk` image builds, `aspnet` runtime image runs — understand *why* multi-stage: build tools don't ship to prod, image ~10x smaller). Ask Claude to review yours, not write it: attempt-first applies to Dockerfiles too.
2. Railway: new project → deploy from GitHub repo → add Postgres plugin → set env vars (`ConnectionStrings__Default` from the plugin's URL, `Jwt__Key` as a generated secret).
3. Migrations on startup for this project (`app.Services` scope → `Migrate()`), with the honest caveat you should be able to *say*: fine for a single-instance demo; with multiple instances you'd migrate as a deploy step instead — two instances racing migrations is a real failure mode.
4. Serve the React build from wwwroot (simplest) or as a separate static deploy. wwwroot keeps one URL, one deploy — take it.
5. Seed demo data + demo credentials (`demo@bookable.dev` / documented password) behind an idempotent seeder.

Smoke test from a different device. Put the URL in the GitHub repo description and your resume.

## What this buys you in interviews

- "Do you have CI/CD experience?" → *"Yes — GitHub Actions on my booking project: every PR builds and runs unit + integration tests against a Postgres service container; main auto-deploys to Railway. Want to see the pipeline?"*
- The pipeline file itself becomes a talking point: warnaserror, service containers, env-var config, why migrations run where they run. Ten minutes of material from 40 lines of YAML.
