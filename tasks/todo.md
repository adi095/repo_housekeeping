# Task list

> **Tracked in GitHub Issues** ([adi095/repo_housekeeping](https://github.com/adi095/repo_housekeeping/issues)). **The issues are the source of truth for status**; update this file only when scope changes.
> **What can I pick?** Open [task-map.html](./task-map.html) (auto-synced from the issues): ready tasks have every dependency closed. Keep the heading format and `**Deps:**` lines below intact; `scripts/task_map.py` parses them.
> Heading format: `### T<NN> · <title> (#<issue>) — <role> · <XS|S|M|L> · <Must|Should|Could>`.

---

## Week 1 · Foundations

### T01 · Describe the project and set the commands (#1) — Dev · XS · Must
- [ ] Fill in the Project and Commands sections of `AGENTS.md`
- [ ] Add a `LICENSE`

**Verify:** a new AI chat can state the project's purpose and how to run its tests · **Deps:** none

### T02 · Scaffold the application (#2) — Dev · S · Must
- [ ] Create the first runnable skeleton with a test
- [ ] Add a CI workflow for it

**Verify:** the test passes locally and in CI · **Deps:** T01

### T03 · Write the first user story and spec (#3) — Dev · S · Should
- [ ] Use the `spec-driven-development` skill to write `docs/spec.md`

**Verify:** the spec has objectives, scope and acceptance criteria · **Deps:** T01
