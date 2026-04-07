# Issue .md Schema Reference

## File Location

All issues live in: `docs/specs/<project>/issues/`

## File Naming

- Forged issues: `P<phase>-<NN>-<slug>.md` (e.g. `P1-01-docker-compose.md`)
- Standalone issues: `<NNN>-<slug>.md` (e.g. `001-fix-login.md`)

The `<slug>` is a kebab-case summary of the issue title.

## YAML Frontmatter

```yaml
---
id: P1-01                  # Canonical identifier, matches filename prefix
title: Short issue title    # Action verb + what
status: todo                # todo | in-progress | done | canceled
phase: 1                   # Phase number (null for standalone)
phase_name: Foundation      # Human-readable phase name (null for standalone)
branch: infra               # Parallel workstream name
position: 1                 # Position in branch sequence (1-indexed)
branch_size: 3              # Total issues in this branch
priority: 1                 # 1=critical, 2=high, 3=medium, 4=low
blocked_by: []              # List of issue IDs that must be done first
created: 2026-04-03         # ISO date
updated: 2026-04-03         # ISO date, updated on every state change
---
```

## Body Structure

### New Issue (created by `/forge` or manually)

```markdown
# <ID>: <Title>

## Objective
What this issue accomplishes — 1-2 sentences.

## Requirements
Detailed requirements, potentially with sub-sections.

## Acceptance Criteria
- [ ] Testable criterion 1
- [ ] Testable criterion 2

## Files Likely Affected
- `path/to/file` — what changes

## Context
Phase spec: `docs/specs/<project>/phase-N-<name>.md`
General spec: `docs/specs/<project>/general.md`
Branch: `<branch>` (position N of M)
```

### Completed Issue (after `/strike`, `/mend`, or `/quench`)

The `## Completion Summary` section is appended:

```markdown
## Completion Summary

**Commit:** `<short-hash>` — `<commit message>`

### What was built
- Bullet points

### Files created/modified
- `path/to/file` — description

### Decisions
- Deviations from spec with rationale
```

For bug fixes (`/mend`), the completion summary uses:

```markdown
## Completion Summary

**Commit:** `<short-hash>` — `<commit message>`

### What was broken
- Description of the bug

### Root cause
- What caused it

### Test added
- `test name` — what it verifies

### Fix applied
- Description of the change

### Files modified
- `path/to/file` — description
```

## ID Conventions

### Forged Issues
- Format: `P<phase>-<NN>`
- Phase is 1-indexed
- NN is 2-digit, sequential within the phase (01, 02, ..., 99)
- Examples: `P1-01`, `P2-03`, `P4-14`

### Standalone Issues
- Format: `<NNN>`
- 3-digit, globally sequential within the project
- To assign: scan `docs/specs/<project>/issues/` for highest existing number, increment
- Examples: `001`, `002`, `042`

## State Transitions

```
todo ──→ in-progress ──→ done
  │         │
  │         └──→ todo (abandoned/paused)
  │
  └──→ canceled
```

## Blocker Resolution

An issue can only start (`todo` → `in-progress`) when ALL issues listed in `blocked_by` have `status: done`.

`/strike` and `/mend` enforce this as a hard stop.
