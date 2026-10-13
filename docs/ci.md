# CI instructions

How continuous integration works in this repo, and how to add or change it. Applies to any stack: replace the example commands with your own.

## Principles
1. **CI runs the same commands you run locally.** Put them behind one entry point (`make ci`, `npm run ci` or a script) so a green local run predicts a green GitHub run.
2. **Fail fast and visibly.** One job per concern (lint, test, build), so a red check names the cause.
3. **Least privilege.** Workflows get `contents: read` unless a job needs more (grant extra permissions on that job only), and secrets are never printed or committed.
4. **Reproducible.** Install from the lockfile (`npm ci`, `pnpm install --frozen-lockfile`, `uv sync --frozen`) so CI fails if the lockfile is stale. The lockfile must be committed.
5. **Nothing merges red.** `main` only receives squash-merged PRs whose required checks pass.

## What runs where
| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| `ci` | `.github/workflows/ci.yml` (add it, see below) | PR, push to `main`, manual | Lint, type-check, test, build |
| `security` | `.github/workflows/security.yml` | PR, push to `main`, manual | Secret scan (gitleaks) always; dependency review and CodeQL on public repos only |
| `task-map` | `.github/workflows/task-map.yml` | Issue changes, edits to `tasks/todo.md` or `scripts/task_map.py`, daily, manual | Regenerates `tasks/task-map.*` and commits it to `main` |

Dependency review needs the dependency graph: for a new repo run `gh api -X PUT repos/<owner>/<repo>/vulnerability-alerts` (or enable it under Settings, Advanced Security).

## Local gate before every PR
Define one command that mirrors CI and run it before pushing. Recipe lines in a Makefile **must be indented with a tab, not spaces**. The placeholders fail on purpose so an unfilled gate can't look green.

```makefile
# Makefile
.PHONY: ci lint test build
ci: lint test build

lint:
	@echo "TODO: set the lint command" && exit 1
test:
	@echo "TODO: set the test command" && exit 1
build:
	@echo "TODO: set the build command" && exit 1
```

Replace each recipe with your real command, for example `npm run lint`, `uv run pytest -q` or `uv build`. Use the same commands in CI (below), so after you add the Makefile `make ci` matches GitHub.

Optional local tools (install them if you want the extra checks before pushing):
- `actionlint` validates workflow files.
- `gitleaks detect --source . --redact` scans for secrets.
- `pre-commit install` runs the hooks in `.pre-commit-config.yaml` on each commit.

## Adding the main CI workflow
Create `.github/workflows/ci.yml`. Pin every action to a full commit SHA with the version in a comment (see Security hygiene); the SHAs below are the ones this repo already uses. Swap the setup steps for your stack.

```yaml
name: ci

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Cancel superseded PR runs, but let runs on main finish.
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  lint:
    name: ci / lint
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      # setup + install steps for your stack
      - run: make lint

  test:
    name: ci / test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      # setup + install steps for your stack
      - run: make test

  build:
    name: ci / build
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      # setup + install steps for your stack
      - run: make build
```

Rules for every job:
- Set `timeout-minutes`.
- Use dependency caching offered by the setup action (`cache: npm`, `enable-cache: true`).
- Upload test reports as artifacts only on failure, as a step placed after the test step.

```yaml
      - name: Upload report on failure
        if: failure()
        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
        with:
          name: test-report
          path: report.xml
          if-no-files-found: ignore
```

### Stack setup steps
Put these between `checkout` and the `make` step.

**Node (npm):**
```yaml
      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        with:
          node-version: "22"
          cache: npm
      - run: npm ci
```

**Node (pnpm):** same as above with `cache: pnpm` on `setup-node`, preceded by `- run: corepack enable` and followed by `- run: pnpm install --frozen-lockfile`. Pin the version in `package.json` (`"packageManager": "pnpm@<version>"`).

**Python (uv):** needs a committed `uv.lock`.
```yaml
      - uses: astral-sh/setup-uv@c18668ad3cf93ea998bef934396af7bb5c839dc7 # v10.2.0
        with:
          enable-cache: true
          python-version: "3.12"
      - run: uv sync --frozen
```

Typical commands behind `make`: `npm run lint`, `npm test`, `npm run build`; or `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy .`, `uv run pytest -q`.

### Jobs that need a database
Add a disposable container to the job and pass its URL through the environment. `localhost` works because the job runs directly on the runner (inside a `container:` job the host would be `db`).

```yaml
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 20
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
    steps:
      # checkout, setup, install, then your test command
```

Use throwaway credentials only; never production ones.

### Starting before the code exists
Add a `detect` job that reports whether the package exists, and make the other jobs depend on it:

```yaml
  detect:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    outputs:
      present: ${{ steps.check.outputs.present }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - id: check
        run: |
          if [ -f package.json ]; then echo "present=true" >> "$GITHUB_OUTPUT"
          else echo "present=false" >> "$GITHUB_OUTPUT"; fi

  lint:
    needs: detect
    if: needs.detect.outputs.present == 'true'
    # ...
```

Without the `outputs:` mapping on `detect`, the value is empty and every job is skipped forever. A skipped job counts as passing for required checks, so remove `detect` and the `if:` lines once the code exists, otherwise a wrong detection lets a PR merge unchecked.

## Making checks required
1. GitHub, Settings, Rules, Rulesets, New ruleset, New branch ruleset; target the default branch.
2. Enable **Require a pull request before merging** (required approvals: 1) and **Require status checks to pass**, then add the job names (for example `ci / lint`, `ci / test`, `ci / build`, `security / gitleaks`). A check only appears in the picker after the workflow has run once.
3. Enable **Block force pushes** and keep deletion restricted.

Private repos on a free plan don't enforce rulesets, so the rule is on us: never push to `main`.

## Security hygiene
- **Secrets:** store them in Settings, Secrets and variables, Actions. Reference them as `${{ secrets.NAME }}`; never echo them.
- **Untrusted input:** never put attacker-controlled text (`github.event.pull_request.title`, `github.head_ref`, issue and comment bodies) directly into `run:`. Pass it through `env:`:
  ```yaml
  # Vulnerable: the title is pasted into the shell script
  - run: echo "${{ github.event.pull_request.title }}"
  # Safe: the shell reads it as data
  - run: echo "$TITLE"
    env:
      TITLE: ${{ github.event.pull_request.title }}
  ```
- **Pinning:** pin all third-party actions to a full commit SHA with the version in a comment, and let Dependabot update them (`.github/dependabot.yml`). Do the same for first-party `actions/*` to keep one policy.
- **Fork PRs:** `pull_request` runs from forks get a read-only token and no secrets, which is what you want. `pull_request_target` runs with secrets, so never check out or run the PR's code under it. In Settings, Actions, General, require approval before workflows run for outside contributors.
- **Job permissions:** OIDC deploys and some publishing steps need `id-token: write` or similar. Add that to the one job that needs it, not at the top of the file.

## Keeping dependencies current
`.github/dependabot.yml` opens weekly PRs. Append an ecosystem block under `updates:` when the project gets one:

```yaml
  - package-ecosystem: npm      # npm, pnpm and yarn all use "npm"; also: uv, pip, docker
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

The `uv` ecosystem expects a `uv.lock` in `directory`. Use `groups:` (as this repo does for CodeQL) when several packages must update together.

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
