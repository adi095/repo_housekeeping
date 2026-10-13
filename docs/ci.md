# CI instructions

How continuous integration works in this repo, and how to add or change it. Applies to any stack: replace the example commands with your own.

## Principles
1. **CI runs the same commands you run locally.** Put them behind one entry point (`make ci`, `npm run ci` or a script) so a green local run predicts a green GitHub run.
2. **Fail fast and visibly.** One job per concern (lint, test, build), so a red check names the cause.
3. **Least privilege.** Workflows get `contents: read` unless a job needs more, and secrets are never printed or committed.
4. **Reproducible.** Install from the lockfile (`npm ci`, `pnpm install --frozen-lockfile`, `uv sync --frozen`) so CI fails if the lockfile is stale.
5. **Nothing merges red.** `main` only receives squash-merged PRs whose required checks pass.

## What runs where
| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| `ci` | `.github/workflows/ci.yml` (add it, see below) | PR, push to `main`, manual | Lint, type-check, test, build |
| `security` | `.github/workflows/security.yml` | PR, push to `main`, manual | Secret scan (gitleaks), dependency review and CodeQL (public repos only) |
| `task-map` | `.github/workflows/task-map.yml` | Issue changes, daily, manual | Regenerates `tasks/task-map.*` and commits it to `main` |

## Local gate before every PR
Define one command that mirrors CI, and run it before pushing.

```makefile
# Makefile
.PHONY: ci lint test build
ci: lint test build

lint:
	# e.g. npm run lint | uv run ruff check . && uv run mypy .
test:
	# e.g. npm test | uv run pytest -q
build:
	# e.g. npm run build | uv build
```

Optional local tools (each is skipped by CI if absent locally):
- `actionlint` validates workflow files.
- `gitleaks detect --source . --redact` scans for secrets.
- `pre-commit install` runs the hooks in `.pre-commit-config.yaml` on each commit.

## Adding the main CI workflow
Create `.github/workflows/ci.yml`. This is the skeleton; swap the steps for your stack.

```yaml
name: ci

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  lint:
    name: ci / lint
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      # setup + install steps for your stack
      - run: make lint

  test:
    name: ci / test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      # setup + install steps for your stack
      - run: make test

  build:
    name: ci / build
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      # setup + install steps for your stack
      - run: make build
```

Rules for every job:
- Set `timeout-minutes`.
- Keep `concurrency` with `cancel-in-progress: true` so a new push cancels the stale run.
- Use dependency caching offered by the setup action (`cache: npm`, `enable-cache: true`).
- Upload test reports as artifacts only `if: failure()`.

### Stack snippets
| Stack | Setup and install | Typical checks |
|---|---|---|
| Node | `actions/setup-node` with `cache: npm`, then `npm ci` | `npm run lint`, `npm test`, `npm run build` |
| pnpm | `corepack enable`, `pnpm install --frozen-lockfile` | `pnpm run lint`, `pnpm run typecheck`, `pnpm test` |
| Python (uv) | `astral-sh/setup-uv`, `uv sync --frozen` | `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy .`, `uv run pytest -q` |

### Jobs that need a database
Run a disposable container next to the job and pass its URL through the environment:

```yaml
    services:
      db:
        image: postgres:16
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: app
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app -d app"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://app:app@localhost:5432/app
```

Use throwaway credentials only; never production ones.

### Starting before the code exists
Add a `detect` job that checks for the package file and writes `present=true|false` to `$GITHUB_OUTPUT`, then give the other jobs `needs: detect` and `if: needs.detect.outputs.present == 'true'`. The workflow stays green until the code arrives.

## Making checks required
1. GitHub, Settings, Rules, New branch ruleset, target `main`.
2. Require a pull request with 1 approval, and require status checks: pick the job names (for example `ci / lint`, `ci / test`, `ci / build`, `security / gitleaks`).
3. Block force pushes and deletion.

Private repos on a free plan don't enforce rulesets, so the rule is on us: never push to `main`.

## Security hygiene
- **Secrets:** store them in Settings, Secrets and variables, Actions. Reference them as `${{ secrets.NAME }}`; never echo them.
- **Untrusted input:** don't put `${{ github.event.* }}` text (titles, branch names) directly into `run:`; pass it through `env:` first.
- **Pinning:** pin third-party actions to a full commit SHA with a version comment, and let Dependabot update them (`.github/dependabot.yml`). First-party `actions/*` may use a version tag.
- **Fork PRs:** don't give them secrets, and avoid `pull_request_target` unless you know the risks.

## Keeping dependencies current
`.github/dependabot.yml` opens weekly PRs. Add an ecosystem block when the project gets one:

```yaml
  - package-ecosystem: npm      # or uv, pip, docker
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

## Adding CD (deploy)
Keep it separate from CI and gated on it.
- **Hosted platforms** (Vercel, Netlify, Render): connect the repo; no workflow is needed.
- **Your own workflow:** trigger on `push` to `main`, use `environment: production` (optionally with required reviewers), and read credentials from secrets or cloud OIDC.
- Always have a rollback plan before the first deploy (see the `shipping-and-launch` skill).

## When a check fails
1. Open the failing job and read the first error, not the last.
2. Reproduce locally with the same command (`make ci`).
3. Fix the cause. Never disable a check, skip a test, add a lint suppression or lower a threshold just to get green; if a check is wrong, change it in its own PR and explain why.
4. Re-run only after pushing a fix; a retry that passes without a change means the test is flaky, so open an issue.

## For AI assistants
- Before opening a PR, run the local gate and report the result.
- Don't edit `.github/workflows/` casually: validate with `actionlint` if available, keep changes minimal and explain them in the PR.
- Never commit `tasks/task-map.*`; the `task-map` workflow owns them.
