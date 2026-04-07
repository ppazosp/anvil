# Anvil

Spec-driven project management for solo developers using AI agents. All state lives in markdown files tracked by git — no external services, no accounts, no API keys.

Forge specs into issues. Strike them into code.

**Works with:** Claude Code, Codex, GitHub Copilot CLI, OpenCode, or any AI coding agent that supports markdown instructions and subagents.

## Setup

### Claude Code

```bash
claude plugin add /path/to/anvil
# Commands available as /forge, /strike, /mend, /inspect, /quench
```

### Codex / Copilot CLI / OpenCode / Other Agents

Anvil's commands and agents are plain markdown files — any agent that can read instructions from `.md` files can use them.

1. Copy this repo into your project or reference it:
   ```bash
   git clone https://github.com/ppazosp/anvil.git .anvil
   ```

2. Add to your agent's instruction file (e.g. `AGENTS.md`, `codex.md`, `.github/copilot-instructions.md`):
   ```markdown
   ## Project Management

   This project uses Anvil for spec-driven issue management.
   - Commands: see `.anvil/commands/` — forge.md, strike.md, mend.md, inspect.md, quench.md
   - Agents: see `.anvil/agents/` — explorer.md, architect.md, simplifier.md, reviewer.md
   - Issue schema: see `.anvil/docs/issue-schema.md`

   When asked to /forge, /strike, /mend, /inspect, or /quench, read and follow
   the corresponding command file in `.anvil/commands/`.
   ```

3. Invoke commands by asking the agent:
   ```
   "Run /forge for myapp"
   "Run /strike P1-01"
   "Show me /inspect for myapp"
   ```

The commands use generic language — "ask the user", "launch a subagent", "run tests" — that any agent can interpret. Agent prompts in `agents/` can be used as system prompts for subagent dispatch on any platform.

## Quick Start

```bash
# 1. Forge: interrogate → spec → phases → issues
/forge myapp

# 2. Inspect: see what's ready to build
/inspect myapp

# 3. Strike: implement a feature end-to-end
/strike P1-01

# 4. Mend: fix a bug with TDD
/mend 003

# 5. Quench: manually close an issue
/quench P1-02
```

## Commands

| Command | What it does |
|---------|-------------|
| `/forge <project>` | Spec interrogation → phase breakdown → issue .md files with parallelism diagrams |
| `/inspect <project>` | Status dashboard with progress bars, branch trees, and ready-to-launch list |
| `/strike <issue-id>` | Full autonomous implementation: explore → plan → TDD → simplify → review → merge |
| `/mend <issue-id>` | TDD bug fix: reproduce with failing test → fix → verify → close |
| `/quench <issue-id>` | Manually complete an issue after doing the work yourself |

## Agents

Anvil includes 4 subagent prompts in `agents/`. These are dispatched during `/strike` and `/mend` workflows.

| Agent | Role | File |
|-------|------|------|
| Explorer | Traces code paths, maps architecture, documents patterns | `agents/explorer.md` |
| Architect | Designs feature architecture, produces implementation blueprints | `agents/architect.md` |
| Simplifier | Refines code for clarity and consistency, preserves functionality | `agents/simplifier.md` |
| Reviewer | Reviews for bugs, silent failures, security issues, and test coverage gaps | `agents/reviewer.md` |

On platforms that support subagents (Claude Code, Codex), these are dispatched automatically. On platforms without subagent support, the main agent reads the prompt file and follows its instructions inline.

## How It Works

### Issue files

Issues are markdown files with YAML frontmatter in `docs/specs/<project>/issues/`:

```yaml
---
id: P1-01
title: Create user schema
status: todo
phase: 1
branch: data-model
position: 1
branch_size: 2
priority: 1
blocked_by: []
---
```

See [docs/issue-schema.md](docs/issue-schema.md) for the full schema.

### Workflow

```
/forge
  ├── General spec (interrogation)
  ├── Phase breakdown (dependencies, parallelism)
  ├── Phase deep-dive (requirements, ACs)
  └── Issue .md files (YAML frontmatter + body)

/inspect
  └── Parses all issue .md files → dashboard

/strike or /mend
  ├── Read issue .md + check blockers
  ├── Explore codebase (explorer subagent)
  ├── Design architecture (architect subagent)
  ├── Setup worktree
  ├── TDD implementation
  ├── Simplify (simplifier subagent)
  ├── Review (reviewer subagent)
  ├── Verify (evidence-based)
  └── Merge + update issue .md

/quench
  └── Append completion summary + mark done
```

### Parallelism

Forge organizes issues into branches (workstreams). Issues in different branches touch different code and can run in parallel via separate worktrees:

```
Phase 1: Foundation
├── data-model (sequential)
│   ├── P1-01: Create schema ──▶ P1-02: Add indexes
├── auth (independent)
│   └── P1-03: Auth middleware
└── config (independent)
    └── P1-04: Environment setup

Max parallelism: 3 agents
```

## Attribution

Anvil includes code adapted from:

- [feature-dev](https://github.com/anthropics/claude-code) — Copyright Anthropic, Apache 2.0
- [pr-review-toolkit](https://github.com/anthropics/claude-code) — Copyright Anthropic, Apache 2.0
- [code-simplifier](https://github.com/anthropics/claude-code) — Copyright Anthropic, Apache 2.0
- [superpowers](https://github.com/obra/superpowers) — Copyright Jesse Vincent 2025, MIT

## License

MIT
