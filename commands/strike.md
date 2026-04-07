---
description: Implement a feature issue end-to-end — read issue, implement, review, merge
argument-hint: Issue ID (e.g. P1-01, 003)
---

# Strike

End-to-end feature implementation driven by an issue .md file. Reads the issue, implements the feature autonomously, and updates the issue when done.

**Issue ID:** $ARGUMENTS

---

## Phase 0: Load Issue & Context

**Goal:** Get full issue context, check blockers, signal work has started

**Actions:**
1. If `$ARGUMENTS` is empty, ask the user for the issue ID
2. Find the issue file: glob for `docs/specs/*/issues/$ARGUMENTS-*.md` or `docs/specs/*/issues/*$ARGUMENTS*.md`
3. Read the issue file and parse YAML frontmatter:
   - `ISSUE_ID`: frontmatter `id`
   - `ISSUE_TITLE`: frontmatter `title`
   - `ISSUE_STATUS`: frontmatter `status`
   - `ISSUE_BRANCH`: frontmatter `branch`
   - `ISSUE_PRIORITY`: frontmatter `priority`
   - `ISSUE_BLOCKED_BY`: frontmatter `blocked_by`
   - `IS_FORGED`: whether the id matches `P<N>-<NN>` pattern
4. Read the issue body for: Objective, Requirements, Acceptance Criteria, Files, Context

5. **Status check:**
   - If `status` is `done` or `canceled`: STOP — "Issue already done/canceled."
   - If `status` is `in-progress`: warn — "Issue already in progress. Continue anyway?"

6. **Blocker check (if `blocked_by` is non-empty):**
   - For each blocker ID: find and read its .md file, parse frontmatter `status`
   - If ANY blocker is NOT `done`: **STOP** — "Blocker <ID> is still <status>. Cannot proceed."
   - If blockers are `done`: read each blocker's `## Completion Summary` section — extract what was built, files modified, decisions. Store as `BLOCKER_CONTEXT`.

7. **Load spec files (if forged issue):**
   - Parse issue body for spec references (paths like `docs/specs/<project>/phase-N-*.md` and `general.md`)
   - Read all referenced spec files
   - Store as `SPEC_CONTEXT`

8. **Update issue status:** edit frontmatter `status: in-progress`, `updated: <today>` and commit

**Output:** "Starting `ISSUE_ID`: `ISSUE_TITLE`" — then proceed immediately.

---

## Phase 1: Understand & Refine Requirements

**If `IS_FORGED` (created by `/forge`):**
1. Requirements already well-defined in `SPEC_CONTEXT` + issue body
2. Incorporate `BLOCKER_CONTEXT` (what blocking issues already built)
3. **Proceed directly to Phase 2** — specs are the source of truth
4. Only stop if something is genuinely unclear or contradictory

**If NOT forged (standalone issue):**
1. Use issue title + body as the feature request
2. If unclear or vague, ask the user to clarify:
   - What problem does this solve?
   - What should it do / not do?
   - Any constraints?
3. If clear enough, proceed without asking

---

## Phase 2: Codebase Exploration

**Read CLAUDE.md** for project context.

**Launch 2-3 explorer subagents IN PARALLEL (see `agents/explorer.md`):**
- "Find features similar to [feature] and trace their implementation"
- "Map the architecture and abstractions for [feature area]"
- "Identify patterns, testing approaches, and extension points relevant to [feature]"

**After agents return:**
1. Read all key files they identified
2. Summarize findings and patterns

---

## Phase 3: Architecture Design

**Launch 2-3 architect subagents IN PARALLEL (see `agents/architect.md`) with different focuses:**
- Minimal changes (smallest change, maximum reuse)
- Clean architecture (maintainability, elegant abstractions)
- Pragmatic balance (speed + quality)

**After agents return:**
1. Select best approach
2. Write implementation plan to `docs/plans/YYYY-MM-DD-<feature-name>.md`
3. Create task list from plan

---

## Phase 4: Setup Isolated Workspace

**Goal:** Isolated worktree for this feature

1. **Check for worktree directory:**
   - If `.worktrees/` exists: use it
   - If `worktrees/` exists: use it
   - If neither: create `.worktrees/`

2. **Verify gitignored:**
   ```bash
   git check-ignore -q .worktrees 2>/dev/null
   ```
   If NOT ignored: add to `.gitignore`, commit.

3. **Create worktree:**
   ```bash
   git worktree add .worktrees/feature-<ISSUE_ID> -b feature/<ISSUE_BRANCH>-<ISSUE_ID>
   cd .worktrees/feature-<ISSUE_ID>
   ```

4. **Install dependencies** (auto-detect from project files):
   - `package.json` → `pnpm install` (or npm/yarn)
   - `pyproject.toml` → `uv sync` (or poetry install)
   - `Cargo.toml` → `cargo build`
   - `go.mod` → `go mod download`

5. **Run tests** to verify clean baseline. If tests fail: report, ask whether to proceed.

---

## Phase 5: Autonomous Implementation

**CRITICAL: Fully autonomous. No stopping for feedback between tasks.**

**For each task from the plan:**

1. **Write the failing test first.**
   - One test for one behavior
   - Run it — verify it FAILS for the right reason (feature missing, not syntax error)
   - If it passes immediately: you're testing existing behavior, fix the test
   - If it errors (import, syntax): fix the error, re-run until it fails correctly

2. **Write minimal code to make the test pass.**
   - Simplest code that passes, nothing more
   - Don't add features, refactor other code, or "improve" beyond the test
   - Run test — verify it PASSES
   - Run full suite — verify no regressions

3. **If tests fail after implementation:**
   - Read the error message carefully
   - Reproduce consistently
   - Check recent changes, trace data flow backward through call stack
   - Form ONE hypothesis, test with the smallest possible change
   - If fix doesn't work: new hypothesis — don't pile fixes on top of each other
   - After 3+ failed fix attempts: STOP. Question the approach. Reconsider architecture. If still stuck, ask the user.

4. **Commit after each task:**
   ```bash
   git commit -m "feat[<module>]: <what was implemented>"
   ```

5. **Proceed to next task immediately**

---

## Phase 6: Code Simplification

**Goal:** Clean up before review

**Dispatch the simplifier subagent** (see `agents/simplifier.md`) on all modified files.

The simplifier refines code for clarity, consistency, and maintainability while preserving functionality. It focuses on the files changed in this branch.

After simplifier returns: review its changes, commit if changes were made.

---

## Phase 7: Code Review

**Goal:** Catch bugs, silent failures, and coverage gaps

**Dispatch the reviewer subagent** (see `agents/reviewer.md`) on all modified files (`git diff --name-only main...HEAD`).

The reviewer checks:
- Code quality, logic errors, security vulnerabilities
- Silent failures, swallowed errors, bad fallbacks
- Test coverage gaps for critical paths

**For each HIGH/CRITICAL issue the reviewer finds:**
1. Fix it
2. Re-run affected tests
3. Commit the fix

---

## Phase 8: Final Verification

**Goal:** Everything passes with evidence

1. Run full test suite — show actual output
2. Type check if applicable (`tsc --noEmit`, `mypy`, etc.)
3. Lint if applicable (`eslint`, `ruff`, etc.)

**VERIFICATION GATE:** You MUST run the verification command, read the full output, and confirm it passes BEFORE claiming success. No "should pass." No "looks correct." Show the actual command output as evidence.

**All checks must pass with shown output.**

---

## Phase 9: Complete — Merge & Update Issue

### 9a: Auto-Merge

1. Switch to main worktree
2. Squash merge: `git merge --squash feature-branch`
3. Commit with descriptive message
4. Push to remote: `git push`
5. Delete worktree and feature branch:
   ```bash
   git worktree remove .worktrees/feature-<ISSUE_ID>
   git branch -d feature/<ISSUE_BRANCH>-<ISSUE_ID>
   ```

**Capture the merge commit hash.**

### 9b: Update Phase Spec (if forged)

If `IS_FORGED` and phase spec file exists:
1. Read the phase spec
2. Mark completed acceptance criteria as `[x]`
3. Update the `Last updated` date
4. Commit: `feat[specs]: mark acceptance criteria for <ISSUE_ID>`

### 9c: Update Issue File

1. Edit frontmatter: `status: done`, `updated: <today>`
2. Append `## Completion Summary`:

```markdown
## Completion Summary

**Commit:** `<short-hash>` — `<commit message>`

### What was built
- <bullet points describing what was implemented>

### Files created/modified
- `<path>` — <description of change>

### Decisions
- <any deviations from spec with rationale>
```

3. Commit: `feat[specs]: complete <ISSUE_ID>`

### 9d: Present Summary

Show: what was built, architecture chosen, files modified, tests added, review findings, autonomous decisions, git log.

---

## Autonomous Decision Guidelines

| Situation | Default Decision |
|-----------|------------------|
| Naming | Follow existing codebase patterns |
| Error handling | Defensive, log errors, fail gracefully |
| Tests | Unit for functions, integration for features |
| Commits | Small, focused, conventional messages |
| Ambiguity | Simpler solution, document in commit |
| Blocked task | Try alternative, if stuck mark blocked and continue |
| Architecture | Follow architect subagent recommendations |
| Issue description vague (non-forged) | Ask user, don't assume scope |
| Issue description vague (forged) | Trust the spec, proceed autonomously |
| Merge | Always squash merge to main, push, clean up worktree |

---

## Red Flags — STOP and ask user if:

- Requirements genuinely unclear or contradictory
- Tests failing after 3+ systematic debug attempts
- Security vulnerability requires design change
- Breaking changes to public API
- Data migration affects production data
- Issue is already done or canceled

---

## Agents Used

| Phase | Agents |
|-------|--------|
| 2 | Explorer subagents (2-3 parallel) — `agents/explorer.md` |
| 3 | Architect subagents (2-3 parallel) — `agents/architect.md` |
| 6 | Simplifier subagent — `agents/simplifier.md` |
| 7 | Reviewer subagent — `agents/reviewer.md` |
