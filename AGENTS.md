# AGENTS.md

Instructions for every AI coding assistant on this repo (GitHub Copilot, Cursor, Codex, and Claude Code via `CLAUDE.md`). Keep this file short; put detail in skills or docs.

## Project
<!-- One paragraph: what this project is, the stack, and its current status. -->
repo_housekeeping: describe the project here.

## Commands
<!-- Replace with the real commands once the project has code. -->
- Install: `...`
- Test: `...`
- Lint: `...`

## Non-negotiables
1. **No secrets in the repo.** Use `.env` (git-ignored) and `.env.example`.
2. **Tests with every change.**
3. **Never push to `main`.** Branch, open a PR, merge only when CI is green.
4. Keep PRs small (about 100 to 300 lines), one concern each, with Conventional Commit titles (`feat:`, `fix:`, `docs:`).

## Picking a task
`tasks/todo.md` lists the tasks; `tasks/task-map.json` (for agents) and `tasks/task-map.html` (for people) show which are done, picked, ready or blocked. The `task-map` workflow regenerates the map from GitHub issues, so pull `main` first.
- Only pick a **ready** task (every dependency closed). Never start a **blocked** one.
- When asked "what should I pick?", answer from `suggestions[<role>]` in the JSON, then show that task's `waiting_on` and `unlocks`.
- **Every task, in this order:**
  1. **Claim:** `scripts/board_status.sh <issue> "In progress"` (sets the board Status and assigns you). Needs `gh auth login -s project`.
  2. **Branch and build:** `git checkout -b <type>/<tNN>-<slug>`, implement with tests.
  3. **Never commit `tasks/task-map.*` in a PR.** The workflow owns those files and a manual commit conflicts with it.
  4. **Independent review before the PR:** commit, `git fetch origin`, then have a **separate agent with fresh context** (a sub-agent or a new session, never the one that wrote the code) review only `git diff origin/main...HEAD` with the `code-review-and-quality` skill (plus `security-and-hardening` for auth, sessions, user input or text sent to an LLM). Give it only the prompt in `docs/ai-setup.md`, not your explanation of the design. **Show the findings to the user and let them decide** which to fix and which to reject (with a reason). Don't apply, commit or push review fixes without their go-ahead. Skip the review only for trivial changes (typo, link, comment).
  5. **PR:** use the PR template, a Conventional Commit title, `Closes #<n>` in the body and a short summary of the review, then `scripts/board_status.sh <issue> "In review"`.
  6. Merging closes the issue, which marks it done on the map and the board.

## Skills (in `.agents/skills`, mirrored in `.claude/skills`)
Load the matching skill before starting; `using-agent-skills` picks one if unsure.

| Task | Skill |
|---|---|
| Clarify or refine a feature | `interview-me`, `idea-refine`, `spec-driven-development` |
| Break work into tasks | `planning-and-task-breakdown` |
| Implement | `incremental-implementation`, `test-driven-development`, `source-driven-development` |
| API or module boundaries | `api-and-interface-design` |
| Database schema and migrations | `supabase-postgres-best-practices`, `supabase` |
| UI screens and design | `impeccable`, `frontend-ui-engineering` |
| UI review / Next.js performance | `web-design-guidelines`, `vercel-react-best-practices` |
| Debugging | `debugging-and-error-recovery`, `browser-testing-with-devtools` |
| Review before merge | `code-review-and-quality`, `security-and-hardening`, `code-simplification` |
| Decisions and docs | `documentation-and-adrs` (ADRs go in `docs/adr/`) |
| Git and CI | `git-workflow-and-versioning`, `ci-cd-and-automation` |

Update skills only with `scripts/sync-skills.sh`; never edit them in place.

## MCP servers
GitHub (issues, PRs, board), Context7 (current library docs; use it before writing framework code), Playwright (browser tests) and Chrome DevTools (debugging and performance). Setup is in `docs/ai-setup.md`.

## Definition of done
- [ ] Meets the task's acceptance criteria
- [ ] Lint, type-check and tests pass locally and in CI
- [ ] Reviewed by a separate agent before the PR; Critical and Required findings fixed or rejected with a reason
- [ ] No secrets; licences respected
- [ ] Docs updated if behaviour changed
