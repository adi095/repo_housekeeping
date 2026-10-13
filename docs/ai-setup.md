# AI tooling setup

How to connect an AI assistant to the shared skills and MCP servers. Rules for the assistants themselves are in [AGENTS.md](../AGENTS.md).

## Prerequisites
- **Node.js 22+** (all MCP servers run through `npx`; the `impeccable` scripts need it too)
- **git**

## What's in the repo
| Path | Purpose | Read by |
|---|---|---|
| `AGENTS.md` | Shared instructions | Copilot, Cursor, Codex; Claude via `CLAUDE.md` (`@AGENTS.md`) |
| `.agents/skills/` | Skills (see [.agents/SOURCES.md](../.agents/SOURCES.md)) | Copilot, Cursor, Codex |
| `.claude/skills/` | Symlinks to the same skills (+ impeccable's Claude build) | Claude Code |
| `.vscode/mcp.json` | MCP servers | GitHub Copilot in VS Code |
| `.cursor/mcp.json` | MCP servers | Cursor |
| `.mcp.json` | MCP servers (project scope) | Claude Code |
| `.codex/config.toml` | MCP servers (project scope, trusted projects only) | Codex |
| `scripts/sync-skills.sh` | Reinstalls skills from pinned commits | Everyone |

**Windows:** enable symlinks before cloning (`git config --global core.symlinks true` plus Developer Mode), or `.claude/skills` will be plain text files and won't work.

## Per-tool setup
### GitHub Copilot (VS Code)
1. Open the repo folder and trust it.
2. Open `.vscode/mcp.json` and click **Start** on each server. GitHub signs in with OAuth; no token needed.
3. Skills and `AGENTS.md` load automatically in agent mode.

### Cursor
1. Create a GitHub personal access token (fine-grained, this repo only: Issues, Pull requests, Contents read) and export it as `GITHUB_PERSONAL_ACCESS_TOKEN`.
2. Enable the servers under **Settings, MCP**.

### Claude Code
1. Export `GITHUB_PERSONAL_ACCESS_TOKEN` as above.
2. Start `claude` in the repo and approve the project MCP servers when prompted (`/mcp` shows status).

### Codex
1. Export `GITHUB_PERSONAL_ACCESS_TOKEN` as above.
2. Run `codex` in the repo and **trust the project** when asked. Check with `/mcp`.

### Known quirk
VS Code scans both `.agents/skills` and `.claude/skills`, so some versions list each skill twice. It's harmless.

## Independent review before every PR
Before opening a PR, a **separate agent with fresh context** (not the one that wrote the code) reviews the diff with the `code-review-and-quality` skill. A reviewer without the author's reasoning catches what the author can't see.

How to start one (commit your work and run `git fetch origin` first):

| Tool | Fresh-context reviewer |
|---|---|
| GitHub Copilot CLI | Ask it to run the prompt below in a **general-purpose sub-agent** |
| GitHub Copilot in VS Code | Turn on **Run Subagent** under *Configure Tools*, then write "Use a subagent to..." followed by the prompt below |
| Claude Code | Ask it to use a **subagent** with the prompt below |
| Cursor / Codex | Open a **new chat or session** on the same branch and paste the prompt |

Prompt to paste (fill in the task; don't add your own explanation of the design):
```text
Use the code-review-and-quality skill to review ONLY the changes in `git diff origin/main...HEAD`
for task T<NN> (issue #<n>). Don't review the rest of the codebase; read surrounding code only
where you need it to judge a changed line. Acceptance criteria: <copy from tasks/todo.md>.
If the diff touches auth, sessions, request input or text passed into an LLM prompt, also use the
security-and-hardening skill.
Review correctness, readability, architecture, security and performance of the changes; run
only the tests that cover them. Don't change, commit or push anything.
Label every finding Critical / Required / Consider / Nit / FYI with file:line, a concrete
failure scenario and a fix, and list missing tests. Be concise: no praise, no restating the diff.
```
Then **show the findings to the user, who decides** which to fix and which to reject (with a reason). Apply only the agreed fixes, don't commit or push without the user's go-ahead, and summarise the review in the PR's "Independent review" section.

## Optional: Context7 API key
Context7 works without a key but is rate-limited. For higher limits, get a free key at context7.com/dashboard and add `"--api-key", "<key>"` to the Context7 `args` **in your personal (user-level) config only**. Never commit it.

## Updating skills
Edit the pinned commit in `scripts/sync-skills.sh`, run it, review the diff, and update `.agents/SOURCES.md`.
