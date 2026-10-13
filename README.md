# repo_housekeeping

A project set up for AI-assisted development: shared rules, reusable skills, MCP servers and a task map that tells humans and agents what to work on next.

| Path | Purpose |
|---|---|
| `AGENTS.md` | Rules every AI assistant reads (`CLAUDE.md` points Claude Code at it) |
| `.agents/skills/` | Reusable "how to do X" guides, mirrored in `.claude/skills/` for Claude Code |
| `.vscode/mcp.json`, `.cursor/mcp.json`, `.mcp.json`, `.codex/config.toml` | The same MCP servers in each tool's format |
| `tasks/todo.md` | The task list; `tasks/task-map.*` is generated from it plus GitHub issues |
| `scripts/` | `sync-skills.sh` (install skills), `task_map.py` (build the map), `board_status.sh` (move a card on the Project board) |
| `.github/` | PR template, CODEOWNERS, Dependabot, secret scanning and the task-map workflow |
| `docs/ai-setup.md` | Per-tool setup and the independent-review prompt |
| `docs/ci.md` | Generic CI instructions: workflow skeleton, required checks, security, CD |

Start with [docs/ai-setup.md](./docs/ai-setup.md).

The skills come from open-source projects listed in [.agents/SOURCES.md](./.agents/SOURCES.md), each under its own licence (texts in `.agents/licenses/`).
