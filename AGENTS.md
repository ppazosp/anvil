# Anvil — Full Reference

This document is the compiled reference for Anvil, a spec-driven project management system for solo developers using AI agents. It contains all commands, agent prompts, and the issue schema in a single document.

Agents and LLMs should follow the instructions in this document. Humans may also find it useful, but guidance here is optimized for automation.

---

# Issue .md Schema Reference

## File Location

All issues live in: `docs/specs/<project>/issues/`

## File Naming

- Forged issues: `P<phase>-<NN>-<slug>.md` (e.g. `P1-001-docker-compose.md`)
- Standalone issues: `<NNN>-<slug>.md` (e.g. `001-fix-login.md`)

The `<slug>` is a kebab-case summary of the issue title.

## YAML Frontmatter

```yaml
---
id: P1-001                  # Canonical identifier, matches filename prefix
title: Short issue title    # Action verb + what
status: todo                # todo | in-progress | done | canceled
phase: 1                   # Phase number (null for standalone)
heat: infra                 # Parallel workstream name (a "heat" in forging)
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
Heat: `<heat-name>`
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
- NN is 3-digit, sequential within the phase (001, 002, ..., 999)
- Examples: `P1-001`, `P2-003`, `P4-014`

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

---

# Command: /forge

Transform a project specification into perfectly defined issue .md files, organized by phases with parallelism optimization for multi-agent execution.

**Input:** Project name or project name + brief description

## Phase 0: Detect State & Load Context

**Goal:** Determine if this is new or a continuation, load all available context

1. **Parse input:** If empty, ask the user for project name + brief description. Otherwise extract project name (first word) and optional description.

2. **Re-entry check:** Look for `docs/specs/<project>/phases.md`. If it exists, count `[x]` (done) vs `[ ]` (pending) phases. If completed phases exist, present status and ask to continue or redo. Also check for interrupted runs (spec files exist but phase unchecked).

3. **Load project context:** Read `CLAUDE.md` for team table, stack, conventions. Read `docs/specs/<project>/general.md` if it exists.

4. **Codebase exploration (if code exists — skip for greenfield):** Launch 2-3 explorer subagents IN PARALLEL (see `agents/explorer.md`):
   - "Map overall architecture: services, modules, boundaries, entry points"
   - "Identify patterns, abstractions, conventions, and extension points"
   - "Document data model, API surface, external integrations, and tech debt"

**If re-entering and general spec + phases already exist, skip directly to the pending phase.**

## Phase 1: General Specification

**Goal:** Extract a complete, unambiguous project specification through relentless interrogation

**CRITICAL: Be aggressive. Do NOT accept vague answers. Push on every assumption.**

1. Start with whatever description the user provided
2. Interview relentlessly — walk down each branch of the decision tree, one question at a time. For each question, provide your recommended answer. If a question can be answered by exploring the codebase, explore instead of asking. Resolve each branch completely before moving to the next.
3. Cover: Purpose, Core features, Data model, Integrations, User flows, Constraints, Non-functional requirements
4. When ALL branches are resolved, present the complete spec summary
5. Get explicit user approval

**Output:** Write to `docs/specs/<project>/general.md` with sections: Overview, Core Features, Out of Scope, Data Model, Integrations, User Flows, Constraints & Non-Functional Requirements, Key Decisions.

**Commit:** `feat[specs]: add general specification for <project>`

## Phase 2: Phase Division

**Goal:** Break the project into ordered, deliverable phases

Propose phases based on technical dependencies, deliverable value, complexity balance, and natural code boundaries. Present with dependencies. Interrogate the user on phase boundaries. Get approval.

**Output:** Write to `docs/specs/<project>/phases.md` with checkbox list of phases, dependencies, and estimated issue counts.

**Commit:** `feat[specs]: add phase breakdown for <project>`

## Phase 3: Phase Deep-Dive

**Goal:** Detailed specification + acceptance criteria for the current phase

**Process ONE phase at a time.** Interview aggressively about requirements, edge cases, acceptance criteria. Define testable, specific, complete acceptance criteria. Get user approval.

**Output:** Write to `docs/specs/<project>/phase-N-<name>.md` with Objective, Requirements (R1, R2...), Acceptance Criteria, Technical Notes, Decisions.

**Commit:** `feat[specs]: add phase N specification for <project>`

## Phase 4: Issue Decomposition & Parallelism

**Goal:** Break the phase into issues optimized for parallel agent execution

1. Identify independent workstreams (heats) by code area/domain
2. Define each issue with title, ACs, files affected, priority, heat name
3. Granularity check: each issue must be executable by a single `/strike` run, touch 1-8 files, have testable ACs, be implementable without human decisions mid-run
4. Present parallelism diagram BEFORE creating anything
5. Get explicit user approval

**DO NOT create issue files until user confirms.**

## Phase 5: Create Issue Files

For each issue (in dependency order, blockers first), write to `docs/specs/<project>/issues/P<N>-<NNN>-<slug>.md` with YAML frontmatter (id, title, status: todo, phase, heat, priority, blocked_by, created, updated) and body (Objective, Requirements, Acceptance Criteria, Files Likely Affected, Context).

Update `phases.md` marking phase `[x]` with issue IDs. Commit. Present summary with `/strike` launch commands.

## Phase 6: Next Phase or Finish

Ask to continue with next phase or stop. If all phases complete, present full project map.

## Red Flags — STOP and ask user if:
- Phase spec contradicts general spec
- A single heat has 5+ sequential issues
- Cross-heat dependencies are unavoidable
- Phase has >10 issues or 1-2 issues

## Autonomous Decision Guidelines
- Branch naming: domain names (`data-model`, `auth`, `api`, `ui`, `config`)
- Issue priority: blockers higher, leaf issues lower
- Phase size: 3-8 issues. Branch size: 1-4 issues.
- Acceptance criteria: 2-5 per issue, always testable

---

# Command: /strike

End-to-end feature implementation driven by an issue .md file. Reads the issue, implements the feature autonomously, and updates the issue when done.

**Input:** Issue ID (e.g. P1-001, 003)

## Phase 0: Load Issue & Context

1. Find the issue file: glob for `docs/specs/*/issues/$ID-*.md`
2. Parse YAML frontmatter for: id, title, status, heat, priority, blocked_by. Detect `IS_FORGED` (id matches `P<N>-<NNN>` pattern).
3. **Status check:** Stop if done/canceled. Warn if in-progress.
4. **Blocker check:** For each blocker, read its .md file. If ANY not `done`: STOP. If done: read their Completion Summary sections as `BLOCKER_CONTEXT`.
5. **Load spec files if forged:** Parse issue body for spec path references, read all.
6. **Update issue status:** Set frontmatter `status: in-progress`, commit.

## Phase 1: Understand & Refine Requirements

If forged: specs are source of truth, proceed directly. If standalone and vague: ask the user to clarify.

## Phase 2: Codebase Exploration

Read CLAUDE.md. Launch 2-3 explorer subagents IN PARALLEL (`agents/explorer.md`) to find similar features, map architecture, identify patterns.

## Phase 3: Architecture Design

Launch 2-3 architect subagents IN PARALLEL (`agents/architect.md`) with different focuses (minimal, clean, pragmatic). Select best approach. Write implementation plan.

## Phase 4: Setup Isolated Workspace

Check for `.worktrees/` or `worktrees/`. Verify gitignored. Create worktree:
```bash
git worktree add .worktrees/strike-<ISSUE_ID> -b strike/<ISSUE_HEAT>-<ISSUE_ID>
```
Install dependencies (auto-detect). Run tests for clean baseline.

## Phase 5: Autonomous Implementation

**Fully autonomous. No stopping for feedback between tasks.**

For each task:
1. **Write the failing test first.** Run it, verify it FAILS for the right reason.
2. **Write minimal code to pass.** Run test, verify it PASSES. Run full suite, verify no regressions.
3. **If tests fail:** Read error carefully. Trace data flow. Form ONE hypothesis, test minimally. After 3+ failures: question the approach, ask user.
4. **Commit after each task.**

## Phase 6: Code Simplification

Dispatch the simplifier subagent (`agents/simplifier.md`) on all modified files. Review changes, commit.

## Phase 7: Code Review

Dispatch the reviewer subagent (`agents/reviewer.md`) on all modified files. Fix HIGH/CRITICAL issues found. Re-run tests. Commit fixes.

## Phase 8: Final Verification

Run full test suite, type check, lint. **VERIFICATION GATE:** Show actual command output as evidence. No "should pass." All checks must pass with shown output.

## Phase 9: Complete — Merge & Update Issue

**9a: Auto-Merge.** Squash merge to main. Push. Delete worktree and strike branch.

**9b: Update Phase Spec (if forged).** Mark completed acceptance criteria as `[x]`.

**9c: Update Issue File.** Set frontmatter `status: done`. Append `## Completion Summary` with: What was built, Files created/modified, Decisions. Commit.

## Autonomous Decision Guidelines
- Naming: follow codebase patterns. Tests: unit for functions, integration for features.
- Ambiguity: simpler solution, document in commit. Blocked: try alternative, then ask user.
- Forged issues: trust the spec. Non-forged: ask if unclear.
- Merge: always squash merge to main.

## Red Flags — STOP and ask user if:
- Requirements unclear or contradictory
- Tests failing after 3+ attempts
- Security vulnerability requires design change
- Breaking changes to public API

---

# Command: /mend

Bug-fix workflow driven by an issue .md file. Reproduces the bug with a failing test first, then fixes it.

**Input:** Issue ID (e.g. P1-001, 003)

## Phase 0: Load Issue & Context

Same as `/strike` Phase 0 — find issue, parse frontmatter, check status/blockers, load specs, set in-progress.

## Phase 1: Understand the Bug

If forged: read specs, present expected vs actual, confirm. If standalone: identify what's broken, where, when. **DO NOT PROCEED without clear understanding.**

## Phase 2: Codebase Exploration

Launch 1 explorer subagent (`agents/explorer.md`) to trace the affected code path. Assess scope: small fix (1-2 files, mend branch) vs larger refactor (3+ files, worktree).

## Phase 3: Reproduce with Failing Test

**CRITICAL: DO NOT modify implementation code yet.** Write a test that fails RIGHT NOW for the right reason. Run it, verify the failure demonstrates the bug. If cannot reproduce: ask user for more details.

## Phase 4: Fix

Smallest change to make the failing test pass. Run full suite. If 3+ fix attempts fail: stop, trace data flow, form hypotheses systematically.

## Phase 5: Simplify & Review

Dispatch simplifier subagent (`agents/simplifier.md`), then reviewer subagent (`agents/reviewer.md`). Fix HIGH/CRITICAL findings.

## Phase 6: Verify

**VERIFICATION GATE:** Run tests, type check, lint. Show actual output. All must pass.

## Phase 7: Commit + Complete

Commit with `fix[<module>]: <description>`. Ask user: merge to main, create PR, or keep mend branch. Update phase spec ACs if forged. Update issue file: set status, append Completion Summary (What was broken, Root cause, Test added, Fix applied, Files modified).

---

# Command: /inspect

Show the current state of a forged project: phase progress, heat diagrams, and what to launch next.

**Input:** Project name

## Step 1: Load Project Data

Read `docs/specs/<project>/phases.md`. Glob for `docs/specs/<project>/issues/*.md`. Parse YAML frontmatter from each issue file.

## Step 2: Build Status Map

Classify issues: done, in-progress, ready (todo + all blockers done), blocked (todo + unfinished blockers), canceled. Calculate per-phase progress.

## Step 3: Display Dashboard

```
Phase 1: <name>  ████████████░░░░ 75% (3/4 done)
──────────────────────────────────────────────────
  data-model
  ✅ P1-001  Create user schema
  ✅ P1-002  Add relations and indexes
  auth
  ✅ P1-003  Setup auth middleware
  config
  🔄 P1-004  Environment and config setup
```

Legend: ✅ done, 🔄 in-progress, ⏳ ready, 🔒 blocked, ← READY = launchable

## Step 4: Next Actions

List ready-to-launch issues with `/strike` commands. List waiting issues with what's blocking them.

---

# Command: /quench

Mark an issue as done after manual implementation. Appends a completion summary and updates status.

**Input:** Issue ID

1. Find and read the issue file
2. Gather completion context from `git log`
3. Update frontmatter: `status: done`, `updated: <today>`
4. Append `## Completion Summary` with: What was built, Files created/modified, Decisions
5. Update phase spec ACs if forged
6. Commit: `feat[specs]: complete <ISSUE_ID>`

---

# Agent: Explorer

Expert code analyst. Traces execution paths, maps architecture layers, documents patterns and dependencies.

**Analysis approach:**
1. Feature Discovery — find entry points, core files, boundaries
2. Code Flow Tracing — follow call chains, trace data transformations, document side effects
3. Architecture Analysis — map abstraction layers, identify patterns, document interfaces
4. Implementation Details — algorithms, error handling, performance, tech debt

**Output:** Entry points with file:line references, execution flow, component responsibilities, architecture insights, dependency map, essential files list.

---

# Agent: Architect

Senior software architect. Designs feature architectures with decisive choices, produces actionable blueprints.

**Process:**
1. Codebase Pattern Analysis — extract patterns, conventions, tech stack, CLAUDE.md guidelines
2. Architecture Design — pick one approach, commit to it, ensure integration
3. Complete Blueprint — every file to create/modify, component design, data flow, build sequence

**Output:** Patterns found, architecture decision with rationale, component design, implementation map, data flow, build sequence checklist, critical details.

---

# Agent: Simplifier

Code simplification specialist. Refines recently modified code for clarity, consistency, and maintainability while preserving exact functionality.

**Rules:**
1. Preserve functionality — never change what code does
2. Apply project standards from CLAUDE.md
3. Enhance clarity — reduce complexity, eliminate redundancy, improve naming, avoid nested ternaries
4. Maintain balance — don't over-simplify, don't create clever solutions, don't prioritize fewer lines over readability
5. Scope — only recently modified code unless told otherwise

---

# Agent: Reviewer

Expert code reviewer covering three domains with confidence-based filtering. Only reports issues with confidence >= 80%.

**Domain 1: Code Quality & Security** — logic errors, null handling, race conditions, SQL injection, XSS, exposed secrets, OWASP top 10, guideline violations, duplication, performance.

**Domain 2: Error Handling Integrity** — empty catch blocks (forbidden), catch-and-continue without feedback, silent optional chaining, broad exception catching, fallback logic masking problems, retry exhaustion without user notice.

**Domain 3: Test Coverage** — untested error paths, missing edge cases, uncovered business logic, absent negative tests. Only report gaps rated >= 7 (could cause user-facing errors or worse).

**Output:** Review summary with critical issues, important issues, test coverage gaps, silent failure risks, and positive observations. Each with file:line, confidence score, and concrete fix suggestion.
