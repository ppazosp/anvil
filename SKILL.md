---
name: anvil
description: Spec-driven project management for solo developers using AI agents. Use when the user asks to plan a project, decompose specs into issues, implement features autonomously, fix bugs with TDD, track issue status, or complete issues. Triggers on /forge, /strike, /mend, /inspect, /quench, "forge issues", "create issues from spec", "implement issue", "fix bug", "project status", "complete issue", "issue dashboard", "phase breakdown", "parallelism", "spec-driven", "issue .md files", "YAML frontmatter issues".
license: MIT
metadata:
  author: ppazosp
  version: "1.0.0"
---

# Anvil

Spec-driven project management for solo developers using AI agents. All state lives in markdown files with YAML frontmatter, tracked by git. No external services.

Forge specs into issues. Strike them into code.

## Commands

| Command | File | What it does |
|---------|------|-------------|
| `/cast <project> <title>` | `commands/cast.md` | Create a standalone issue .md file |
| `/forge <project>` | `commands/forge.md` | Spec interrogation → phase breakdown → issue .md files with parallelism diagrams |
| `/strike <issue-id>` | `commands/strike.md` | Full autonomous implementation: explore → plan → TDD → simplify → review → merge |
| `/mend <issue-id>` | `commands/mend.md` | TDD bug fix: reproduce with failing test → fix → verify → close |
| `/inspect <project>` | `commands/inspect.md` | Compact status dashboard with progress bars and ready-to-launch commands |
| `/quench <issue-id>` | `commands/quench.md` | Manually complete an issue after doing the work yourself |

## Agents

Subagent prompts dispatched during `/strike` and `/mend`. On platforms without subagent support, read the prompt and follow inline.

| Agent | File | Role |
|-------|------|------|
| Explorer | `agents/explorer.md` | Traces code paths, maps architecture, documents patterns |
| Architect | `agents/architect.md` | Designs feature architecture, produces implementation blueprints |
| Simplifier | `agents/simplifier.md` | Refines code for clarity and consistency |
| Reviewer | `agents/reviewer.md` | Reviews for bugs, silent failures, security, test coverage gaps |

## Issue Schema

Issues are `.md` files in `docs/specs/<project>/issues/` with YAML frontmatter:

```yaml
---
id: P1-001             # P<phase>-<NNN> for forged, <NNN> for standalone
title: Create user schema
status: todo           # todo | in-progress | done | canceled
kind: strike           # strike (feature) | mend (bug fix)
phase: 1
heat: data-model       # parallel workstream
priority: 1            # 1=critical → 4=low
blocked_by: []
created: 2026-04-03
updated: 2026-04-03
---
```

Full schema: `docs/issue-schema.md`

## Workflow

```
/cast    → create standalone issue .md file
/forge   → spec → phases → issue .md files (with parallelism diagrams)
/inspect → parse issue .md files → compact status dashboard → ready-to-launch list
/strike  → read issue → explore → plan → TDD → simplify → review → merge → update .md
/mend    → read issue → explore → failing test → fix → simplify → review → verify → update .md
/quench  → append completion summary → mark done
```

## Full Reference

For the complete compiled reference (all commands, agents, and schema), see `AGENTS.md`.
