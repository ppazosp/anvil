# Anvil — Full Reference

This document is the compiled reference for Anvil, a spec-driven project management system for solo developers using AI agents. It contains all commands, agent prompts, and the issue schema in a single document.

Agents and LLMs should follow the instructions in this document. Humans may also find it useful, but guidance here is optimized for automation.

---

# Issue .md Schema

All issues live in `docs/specs/<project>/issues/`. Forged issues use `P<phase>-<NNN>-<slug>.md`, standalone issues use `<NNN>-<slug>.md`.

## YAML Frontmatter

```yaml
---
id: P1-001                 # Canonical identifier, matches filename prefix
title: Short issue title    # Action verb + what
status: todo                # todo | in-progress | done | canceled
kind: strike                # strike (feature) | mend (bug fix)
phase: 1                   # Phase number (null for standalone)
heat: infra                 # Parallel workstream name (null for standalone)
priority: 1                 # 1=critical, 2=high, 3=medium, 4=low
blocked_by: []              # List of issue IDs that must be done first
created: 2026-04-03         # ISO date
updated: 2026-04-03         # ISO date, updated on every state change
---
```

## Issue Body

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

## Completion Summary (appended after work is done)

For features (`/strike`):
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

For bugs (`/mend`):
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

- **Forged:** `P<phase>-<NNN>` — 3-digit, sequential within phase (P1-001, P2-003)
- **Standalone:** `<NNN>` — 3-digit, globally sequential (001, 002, 042)

## State Transitions

```
todo → in-progress → done
  │       │
  │       └→ todo (abandoned)
  └→ canceled
```

Blocker rule: an issue can only start when ALL `blocked_by` issues have `status: done`. `/strike` and `/mend` enforce this as a hard stop.

---

# Command: /cast

Create a new standalone issue .md file. Use this for issues outside the `/forge` workflow — ad-hoc bugs, features, or tasks.

**Input:** Project name + title (e.g. `myapp "Fix login redirect"`)

## Steps

1. **Parse input:** Extract project name (first word), title (quoted string), and optional `--kind` flag (defaults to `strike`). If empty, ask the user.

2. **Determine the next ID:** Glob for `docs/specs/<project>/issues/[0-9]*.md`, find the highest 3-digit ID, increment by 1 (or start at `001`).

3. **Gather details:**
   - Ask the user for a brief description
   - If `kind` is `mend`: ask what's broken, expected vs actual behavior
   - If `kind` is `strike`: ask what should be built, acceptance criteria
   - Recommend answers when you can infer from context

4. **Write the issue file** to `docs/specs/<project>/issues/<NNN>-<slug>.md`:

   ```markdown
   ---
   id: <NNN>
   title: <title>
   status: todo
   phase: null
   heat: null
   kind: <strike|mend>
   priority: <1-4>
   blocked_by: []
   created: <YYYY-MM-DD>
   updated: <YYYY-MM-DD>
   ---

   # <NNN>: <Title>

   ## Objective
   <what this issue accomplishes — 1-2 sentences>

   ## Requirements
   <specific requirements>

   ## Acceptance Criteria
   - [ ] <criterion 1>
   - [ ] <criterion 2>

   ## Files Likely Affected
   - `<path/to/file>` — <what changes>
   ```

5. **Commit:** `feat[specs]: cast issue <NNN> for <project>`

6. **Present:**
   ```
   Cast <NNN>: <title>
   Kind: <strike|mend>
   Path: docs/specs/<project>/issues/<NNN>-<slug>.md

   Run: /<kind> <NNN>
   ```

---

# Command: /forge

Transform a project specification into perfectly defined issue .md files, organized by phases with parallelism optimization for multi-agent execution.

**Input:** Project name or project name + brief description

## Phase 0: Detect State & Load Context

**Goal:** Determine if this is new or a continuation, load all available context

**Actions:**

1. **Parse input:** If empty, ask the user for project name + brief description. Otherwise extract project name (first word) and optional description.

2. **Re-entry check:**
   - Look for `docs/specs/<project>/phases.md`
   - If it exists, read it and count `[x]` (done) vs `[ ]` (pending) phases
   - If completed phases exist: present status, ask to continue or redo
   - Check for interrupted runs (spec files exist but phase unchecked)

3. **Load project context:** Read `CLAUDE.md` for team table, stack, conventions. Read `docs/specs/<project>/general.md` if it exists.

4. **Codebase exploration (if code exists — skip for greenfield):**
   - Launch 2-3 **explorer** subagents IN PARALLEL (see `agents/explorer.md`):
     - "Map overall architecture: services, modules, boundaries, entry points"
     - "Identify patterns, abstractions, conventions, and extension points"
     - "Document data model, API surface, external integrations, and tech debt"
   - Synthesize findings — these inform all subsequent phases

**If re-entering and general spec + phases already exist, skip directly to the pending phase.**

## Phase 1: General Specification

**Goal:** Extract a complete, unambiguous project specification through relentless interrogation

**CRITICAL: Be aggressive. Do NOT accept vague answers. Push on every assumption.**

1. Start with whatever description the user provided
2. Interview relentlessly — walk down each branch of the decision tree, one question at a time. For each question, **provide your recommended answer**. If a question can be answered by exploring the codebase, **explore instead of asking**. Resolve each branch completely before moving to the next.
3. **Cover:** Purpose, Core features, Data model, Integrations, User flows, Constraints, Non-functional requirements
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

**Process ONE phase at a time.** Interview aggressively about requirements, edge cases, acceptance criteria. Acceptance criteria must be testable, specific, and complete. Get user approval.

**Output:** Write to `docs/specs/<project>/phase-N-<name>.md` with Objective, Requirements (R1, R2...), Acceptance Criteria, Technical Notes, Decisions.

**Commit:** `feat[specs]: add phase N specification for <project>`

## Phase 4: Issue Decomposition & Parallelism

**Goal:** Break the phase into issues optimized for parallel agent execution

1. **Identify independent workstreams (heats)** by code area/domain. Issues in the same heat share files → sequential. Different heats → parallel.
2. **Define each issue** with title, ACs, files affected, priority, heat name
3. **Granularity check:** each issue must be executable by a single `/strike` run, touch 1-8 files, have testable ACs, be implementable without human decisions mid-run
4. **Granularity red lines:** too big → split. Too small → merge. Too coupled → redesign heats.
5. **Present parallelism diagram** BEFORE creating anything
6. **Get explicit user approval**

**DO NOT create issue files until user confirms.**

## Phase 5: Create Issue Files

For each issue (in dependency order, blockers first), write to `docs/specs/<project>/issues/P<N>-<NNN>-<slug>.md` with YAML frontmatter (id, title, status: todo, kind: strike, phase, heat, priority, blocked_by, created, updated) and body (Objective, Requirements, Acceptance Criteria, Files Likely Affected, Context).

Update `phases.md` marking phase `[x]` with issue IDs and heats. Commit. Present summary with `/strike` launch commands.

## Phase 6: Next Phase or Finish

Ask to continue with next phase or stop. If all phases complete, present full project map.

## Red Flags — STOP and ask user if:
- Phase spec contradicts general spec
- A single heat has 5+ sequential issues
- Cross-heat dependencies are unavoidable
- Phase has >10 issues or 1-2 issues

## Autonomous Decision Guidelines
- Heat naming: domain names (`data-model`, `auth`, `api`, `ui`, `config`)
- Issue priority: blockers higher, leaf issues lower
- Phase size: 3-8 issues. Heat size: 1-4 issues.
- Acceptance criteria: 2-5 per issue, always testable

---

# Command: /strike

End-to-end feature implementation driven by an issue .md file. Reads the issue, implements the feature autonomously, and updates the issue when done.

**Input:** Issue ID (e.g. P1-001, 003)

## Phase 0: Load Issue & Context

1. Find the issue file: glob for `docs/specs/*/issues/$ID-*.md`
2. Parse YAML frontmatter for: id, title, status, heat, kind, priority, blocked_by. Detect `IS_FORGED` (id matches `P<N>-<NNN>` pattern).
3. Read the issue body for: Objective, Requirements, Acceptance Criteria, Files, Context
4. **Status check:** Stop if done/canceled. Warn if in-progress.
5. **Blocker check:** For each blocker, read its .md file. If ANY not `done`: **STOP**. If done: read their `## Completion Summary` sections as `BLOCKER_CONTEXT`.
6. **Load spec files if forged:** Parse issue body for spec path references, read all. Store as `SPEC_CONTEXT`.
7. **Update issue status:** Set frontmatter `status: in-progress`, `updated: <today>`, commit.

**Output:** "Starting `ISSUE_ID`: `ISSUE_TITLE`" — then proceed immediately.

## Phase 1: Understand & Refine Requirements

**If `IS_FORGED`:** Specs are source of truth. Incorporate `BLOCKER_CONTEXT`. Proceed directly to Phase 2. Only stop if genuinely unclear or contradictory.

**If NOT forged:** Use issue title + body. If unclear, ask the user to clarify: What problem? What should it do/not do? Constraints? If clear enough, proceed without asking.

## Phase 2: Codebase Exploration

Read CLAUDE.md. Launch 2-3 **explorer** subagents IN PARALLEL (see `agents/explorer.md`):
- "Find features similar to [feature] and trace their implementation"
- "Map the architecture and abstractions for [feature area]"
- "Identify patterns, testing approaches, and extension points relevant to [feature]"

After agents return: read all key files identified, summarize findings.

## Phase 3: Architecture Design

Launch 2-3 **architect** subagents IN PARALLEL (see `agents/architect.md`) with different focuses:
- Minimal changes (smallest change, maximum reuse)
- Clean architecture (maintainability, elegant abstractions)
- Pragmatic balance (speed + quality)

Select best approach. Write implementation plan to `docs/plans/YYYY-MM-DD-<feature-name>.md`. Create task list.

## Phase 4: Setup Isolated Workspace

1. Check for `.worktrees/` or `worktrees/`. If neither: create `.worktrees/`.
2. Verify gitignored. If not: add to `.gitignore`, commit.
3. Create worktree:
   ```bash
   git worktree add .worktrees/strike-<ISSUE_ID> -b strike/<ISSUE_HEAT>-<ISSUE_ID>
   cd .worktrees/strike-<ISSUE_ID>
   ```
4. Install dependencies (auto-detect: pnpm/npm/yarn/uv/cargo).
5. Run tests for clean baseline. If tests fail: report, ask whether to proceed.

## Phase 5: Autonomous Implementation

**CRITICAL: Fully autonomous. No stopping for feedback between tasks.**

For each task from the plan:

1. **Write the failing test first.** One test for one behavior. Run it — verify it FAILS for the right reason (feature missing, not syntax error). If it passes immediately: fix the test. If it errors: fix the error, re-run until it fails correctly.

2. **Write minimal code to make the test pass.** Simplest code that passes, nothing more. Don't add features, refactor other code, or "improve" beyond the test. Run test — verify it PASSES. Run full suite — verify no regressions.

3. **If tests fail after implementation:** Read the error carefully. Reproduce consistently. Check recent changes, trace data flow backward through call stack. Form ONE hypothesis, test with the smallest possible change. If fix doesn't work: new hypothesis — don't pile fixes. After 3+ failed attempts: STOP. Question the approach. Reconsider architecture. If still stuck, ask the user.

4. **Commit after each task:** `feat[<module>]: <what was implemented>`

5. **Proceed to next task immediately.**

## Phase 6: Code Simplification

Dispatch the **simplifier** subagent (see `agents/simplifier.md`) on all modified files. The simplifier refines code for clarity, consistency, and maintainability while preserving functionality. Review changes, commit if changes were made.

## Phase 7: Code Review

Dispatch the **reviewer** subagent (see `agents/reviewer.md`) on all modified files (`git diff --name-only main...HEAD`). The reviewer checks code quality, logic errors, security vulnerabilities, silent failures, swallowed errors, bad fallbacks, and test coverage gaps.

For each HIGH/CRITICAL issue found: fix it, re-run affected tests, commit the fix.

## Phase 8: Final Verification

Run full test suite, type check, lint. **VERIFICATION GATE:** You MUST run the verification command, read the full output, and confirm it passes BEFORE claiming success. No "should pass." No "looks correct." Show the actual command output as evidence. All checks must pass with shown output.

## Phase 9: Complete — Merge & Update Issue

**9a: Auto-Merge.** Switch to main worktree. Squash merge: `git merge --squash strike-branch`. Commit with descriptive message. Push. Delete worktree and strike branch.

**9b: Update Phase Spec (if forged).** Mark completed acceptance criteria as `[x]`. Update `Last updated` date. Commit.

**9c: Update Issue File.** Set frontmatter `status: done`, `updated: <today>`. Append `## Completion Summary` with: What was built, Files created/modified, Decisions. Commit: `feat[specs]: complete <ISSUE_ID>`.

**9d: Present Summary.** Show: what was built, architecture chosen, files modified, tests added, review findings, autonomous decisions, git log.

## Autonomous Decision Guidelines
- Naming: follow codebase patterns. Tests: unit for functions, integration for features.
- Ambiguity: simpler solution, document in commit. Blocked: try alternative, then ask user.
- Forged issues: trust the spec. Non-forged: ask if unclear.
- Merge: always squash merge to main.

## Red Flags — STOP and ask user if:
- Requirements unclear or contradictory
- Tests failing after 3+ systematic debug attempts
- Security vulnerability requires design change
- Breaking changes to public API
- Data migration affects production data
- Issue is already done or canceled

---

# Command: /mend

Bug-fix workflow driven by an issue .md file. Reproduces the bug with a failing test first, then fixes it.

**Input:** Issue ID (e.g. P1-001, 003)

## Phase 0: Load Issue & Context

Same as `/strike` Phase 0 — find issue, parse frontmatter, check status/blockers, load specs, set in-progress.

## Phase 1: Understand the Bug

**If `IS_FORGED`:** Read specs for expected behavior context. Present: "Expected behavior is X but the bug is Y. Confirm?" If confirmed → proceed.

**If NOT forged:** Identify what's broken (expected vs actual), where, when (reproduction conditions). If vague: ask for clarification. **DO NOT PROCEED without clear understanding.**

## Phase 2: Codebase Exploration

Launch 1 **explorer** subagent (see `agents/explorer.md`):
- "Trace the code path for [affected behavior]. Find the exact location where the bug likely originates."

Assess scope: small fix (1-2 files, mend branch) vs larger refactor (3+ files, worktree).

**If worktree needed:**
1. Check for `.worktrees/` (create if needed, verify gitignored)
2. `git worktree add .worktrees/mend-<ISSUE_ID> -b mend/<ISSUE_HEAT>-<ISSUE_ID>`
3. Install dependencies, verify tests pass

**If small fix:** `git checkout -b mend/<ISSUE_HEAT>-<ISSUE_ID>`

## Phase 3: Reproduce with Failing Test

**CRITICAL: DO NOT modify implementation code yet.**

1. Write a test that captures the exact broken behavior. Should pass when fixed, should fail RIGHT NOW for the right reason.
2. Run the test. Verify it fails for the right reason — not a syntax error, import error, or unrelated failure.
3. **If cannot reproduce:** ask user for more details. DO NOT skip this phase.

## Phase 4: Fix

1. Smallest change that fixes the bug
2. Run the failing test — confirm it passes
3. Run full suite — confirm no regressions
4. If 3+ fix attempts fail: stop, trace data flow, form hypotheses systematically, ask user if still stuck

## Phase 5: Simplify & Review

1. Dispatch the **simplifier** subagent (see `agents/simplifier.md`) on modified files — clean up before review
2. Dispatch the **reviewer** subagent (see `agents/reviewer.md`) on modified files — check for remaining issues
3. Fix any HIGH/CRITICAL findings

## Phase 6: Verify

**VERIFICATION GATE:** Run full test suite, type check, lint. Show actual command output as evidence. No "should pass." All checks must pass with shown output.

## Phase 7: Commit + Complete

**7a: Commit.** `fix[<module>]: <description of what was fixed>`

**7b: Merge Decision.** Ask user: merge to main (squash merge, delete branch/worktree, push), create PR (push, `gh pr create`), or keep mend branch.

**7c: Update Phase Spec (if forged).** Mark completed ACs as `[x]`, update `Last updated`.

**7d: Update Issue File.** Set frontmatter status (`done` if merged, `in-progress` if PR/keep). Append `## Completion Summary` with: What was broken, Root cause, Test added, Fix applied, Files modified. Commit: `feat[specs]: complete <ISSUE_ID>`.

## Red Flags — STOP and ask user if:
- Bug cannot be reproduced after clarification
- Fix requires changing public API or data schema
- Fix introduces breaking changes
- Multiple unrelated bugs found
- Root cause is in a dependency or external system
- Issue is already done or canceled

---

# Command: /inspect

Show the current state of a project: phase progress, issue status by heat, and what to launch next.

**Input:** Project name

## Step 1: Load Project Data

1. If empty: ask the user for the project name
2. Read `docs/specs/<project>/phases.md` for phase breakdown and issue IDs
3. Glob for `docs/specs/<project>/issues/*.md`
4. For each issue file: parse YAML frontmatter for `id`, `title`, `status`, `phase`, `heat`, `kind`, `blocked_by`

## Step 2: Build Status Map

Classify each issue: `done`, `in-progress`, `ready` (todo + all blockers done), `blocked` (todo + unfinished blockers), `canceled`. Calculate per-phase progress.

## Step 3: Display Dashboard

**Compact horizontal format — one line per heat, issues chained with arrows:**

```
myapp — Phase 1: Foundation  ████████░░ 3/4

  data-model   ✅ P1-001 → ✅ P1-002
  auth         ✅ P1-003
  config       🔄 P1-004

myapp — Phase 2: API  ░░░░░░░░░░ 0/5

  api          ⏳ P2-001 → 🔒 P2-002 → 🔒 P2-003
  ui           ⏳ P2-004 → 🔒 P2-005

⚡ /strike P2-001  /strike P2-004
```

Icons: `✅` done, `🔄` in-progress, `⏳` ready, `🔒` blocked, `❌` canceled

Ready-to-launch line uses `/strike` or `/mend` based on the issue's `kind` field. Omit completed phases unless `--all` flag.

---

# Command: /quench

Mark an issue as done after manual implementation. Appends a completion summary and updates status.

**Input:** Issue ID

1. Find and read the issue file
2. Gather completion context from `git log`
3. Update frontmatter: `status: done`, `updated: <today>`
4. Append `## Completion Summary` (format depends on `kind` field — feature or bug)
5. Update phase spec ACs if forged
6. Commit: `feat[specs]: complete <ISSUE_ID>`

---

# Agent: Explorer

Expert code analyst. Traces execution paths, maps architecture layers, documents patterns and dependencies.

**Analysis approach:**

1. **Feature Discovery** — find entry points (APIs, UI components, CLI commands), locate core files, map boundaries and configuration
2. **Code Flow Tracing** — follow call chains from entry to output, trace data transformations, identify dependencies, document side effects
3. **Architecture Analysis** — map abstraction layers (presentation → business logic → data), identify design patterns, document interfaces, note cross-cutting concerns (auth, logging, caching)
4. **Implementation Details** — key algorithms and data structures, error handling and edge cases, performance considerations, technical debt

**Output:** Entry points with file:line references, step-by-step execution flow, component responsibilities, architecture insights, dependency map, essential files list. Always include specific file paths and line numbers.

---

# Agent: Architect

Senior software architect. Designs feature architectures with decisive choices, produces actionable blueprints.

**Process:**

1. **Codebase Pattern Analysis** — extract patterns, conventions, tech stack, CLAUDE.md guidelines. Find similar features.
2. **Architecture Design** — pick one approach, commit to it. Ensure integration with existing code. Design for testability, performance, maintainability.
3. **Complete Blueprint** — every file to create/modify, component responsibilities, integration points, data flow, build sequence.

**Output:** Patterns found with file:line references, architecture decision with rationale, component design, implementation map, data flow, build sequence checklist, critical details (error handling, state, testing, performance, security). Make confident choices rather than presenting options.

---

# Agent: Simplifier

Code simplification specialist. Refines recently modified code for clarity, consistency, and maintainability while preserving exact functionality.

**Rules:**

1. **Preserve functionality** — never change what code does, only how it does it
2. **Apply project standards** from CLAUDE.md — import conventions, function style, type annotations, framework patterns, error handling, naming
3. **Enhance clarity** — reduce complexity and nesting, eliminate redundancy, improve naming, consolidate related logic, remove obvious comments, avoid nested ternaries, choose clarity over brevity
4. **Maintain balance** — don't over-simplify, don't create clever solutions, don't combine too many concerns, don't remove helpful abstractions, don't prioritize fewer lines over readability
5. **Scope** — only recently modified code unless told otherwise

**Process:** Identify modified files (`git diff --name-only main...HEAD`), analyze, apply standards, verify functionality preserved, commit: `refactor: simplify implementation`

---

# Agent: Reviewer

Expert code reviewer covering three domains with confidence-based filtering. **Only reports issues with confidence >= 80%.**

**Domain 1: Code Quality & Security**
- Logic errors, null/undefined handling, race conditions
- SQL injection, XSS, exposed secrets, OWASP top 10
- Project guideline violations (from CLAUDE.md)
- Code duplication, performance problems
- Missing critical error handling, accessibility issues

**Domain 2: Error Handling Integrity**
- Empty catch blocks (absolutely forbidden)
- Catch blocks that only log and continue without user feedback
- Returning null/undefined/default on error without logging
- Optional chaining (`?.`) silently skipping operations
- Broad exception catching hiding unrelated errors
- Fallback logic masking problems
- Retry exhaustion without user notice

For each error handler: Is the error logged with context? Does the user get actionable feedback? Does the catch only catch expected types? Should it propagate instead?

**Domain 3: Test Coverage**
- Untested error handling paths
- Missing edge case coverage
- Uncovered critical business logic
- Absent negative test cases
- Missing async/concurrent tests

Rate gaps 1-10 (9-10: data loss/security, 7-8: user-facing errors, 5-6: confusion, 1-4: optional). Only report gaps rated >= 7.

**Confidence scoring:** Rate every issue 0-100. Only report >= 80. Quality over quantity.

**Output format:** Review summary with critical issues, important issues, test coverage gaps, silent failure risks, positive observations. Each with file:line, confidence score, and concrete fix suggestion.
