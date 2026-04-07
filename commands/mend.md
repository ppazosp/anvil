---
description: Fix a bug issue with TDD — reproduce with failing test, fix, verify, close
argument-hint: Issue ID (e.g. P1-01, 003)
---

# Mend

Bug-fix workflow driven by an issue .md file. Reproduces the bug with a failing test first, then fixes it.

**Issue ID:** $ARGUMENTS

---

## Phase 0: Load Issue & Context

**Goal:** Get full bug context, check blockers, signal work has started

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
4. Read the issue body

5. **Status check:**
   - If `status` is `done` or `canceled`: STOP — "Issue already done/canceled."
   - If `status` is `in-progress`: warn — "Issue already in progress. Continue anyway?"

6. **Blocker check (if `blocked_by` is non-empty):**
   - For each blocker ID: find and read its .md file, parse frontmatter `status`
   - If ANY blocker is NOT `done`: **STOP** — "Blocker <ID> is still <status>. Cannot proceed."
   - If blockers are `done`: read each blocker's `## Completion Summary` section. Store as `BLOCKER_CONTEXT`.

7. **Load spec files if forged:** parse issue body for spec references, read all, store as `SPEC_CONTEXT`

8. **Update issue status:** edit frontmatter `status: in-progress`, `updated: <today>` and commit

**Present:** Issue title, description summary, priority. If forged: show spec summary. Confirm this is the right issue.

---

## Phase 1: Understand the Bug

**If `IS_FORGED`:**
1. Read spec files for expected behavior context
2. Present: "Expected behavior is X but the bug is Y. Confirm?"
3. If confirmed → proceed

**If NOT forged:**
1. Identify: What's broken (expected vs actual), where, when (reproduction conditions)
2. If vague: ask for clarification
3. **DO NOT PROCEED without clear understanding**

---

## Phase 2: Codebase Exploration

**Launch 1 explorer subagent (see `agents/explorer.md`):**
- "Trace the code path for [affected behavior]. Find the exact location where the bug likely originates."

**After agent returns:**
1. Read key files identified
2. Identify root cause area
3. **Assess scope:**
   - Small fix (1-2 files) → fix branch in current workspace
   - Larger refactor (3+ files) → create worktree

**If worktree needed:**
1. Check for `.worktrees/` directory (create if needed, verify gitignored)
2. Create worktree:
   ```bash
   git worktree add .worktrees/fix-<ISSUE_ID> -b fix/<ISSUE_BRANCH>-<ISSUE_ID>
   cd .worktrees/fix-<ISSUE_ID>
   ```
3. Install dependencies (auto-detect from project files)
4. Verify tests pass on clean baseline

**If small fix:**
```bash
git checkout -b fix/<ISSUE_BRANCH>-<ISSUE_ID>
```

---

## Phase 3: Reproduce with Failing Test

**CRITICAL: DO NOT modify implementation code yet.**

1. Write a test that captures the exact broken behavior:
   - Should pass when bug is fixed
   - Should fail RIGHT NOW for the right reason
   - Name clearly: `test_<what_should_work>` or `test_<bug_description>`

2. Run the test:
   ```bash
   pytest tests/path/test.py::test_name -v
   pnpm test -- --grep "test name"
   go test -run TestName -v
   ```

3. **Verify it fails for the right reason** — not a syntax error, import error, or unrelated failure. The failure should demonstrate the bug.

4. **If cannot reproduce with a test:** ask user for more details. DO NOT skip this phase.

---

## Phase 4: Fix

1. Make the **smallest change** that fixes the bug
2. Run the failing test — confirm it now passes
3. Run full test suite — confirm no regressions
4. If regressions: fix them
5. If 3+ fix attempts fail:
   - STOP random fixing
   - Read errors carefully, trace data flow
   - Form one hypothesis, test with the smallest possible change
   - If still failing after systematic investigation: question the approach, ask user

---

## Phase 5: Simplify & Review

1. **Dispatch the simplifier subagent** (see `agents/simplifier.md`) on modified files — clean up before review
2. **Dispatch the reviewer subagent** (see `agents/reviewer.md`) on modified files — check for remaining issues
3. Fix any HIGH/CRITICAL findings

---

## Phase 6: Verify

**VERIFICATION GATE:** Run full test suite, type check, lint. Show actual command output as evidence. No "should pass." No "looks correct."

**All checks must pass with shown output.**

---

## Phase 7: Commit + Complete

### 7a: Commit

```bash
git add <changed files>
git commit -m "fix[<module>]: <description of what was fixed>"
```

### 7b: Merge Decision

Ask user:
- **Merge to main** — squash merge, delete branch/worktree, push
- **Create PR** — push branch, open PR via `gh pr create`
- **Keep branch** — leave for manual review

Execute chosen option.

### 7c: Update Phase Spec (if forged)

If `IS_FORGED` and phase spec file exists:
1. Read the phase spec
2. Mark completed acceptance criteria as `[x]`
3. Update `Last updated` date
4. Commit: `feat[specs]: mark acceptance criteria for <ISSUE_ID>`

### 7d: Update Issue File

1. Edit frontmatter:
   - If merged: `status: done`
   - If PR or keep branch: `status: in-progress`
   - `updated: <today>`

2. Append `## Completion Summary`:

```markdown
## Completion Summary

**Commit:** `<short-hash>` — `<commit message>`

### What was broken
- <description of the bug>

### Root cause
- <what caused the bug>

### Test added
- `<test name>` — <what it verifies>

### Fix applied
- <description of the change>

### Files modified
- `<path>` — <description>
```

3. Commit: `feat[specs]: complete <ISSUE_ID>`

---

## Red Flags — STOP and ask user if:

- Bug cannot be reproduced after clarification
- Fix requires changing public API or data schema
- Fix introduces breaking changes
- Multiple unrelated bugs found during investigation
- Root cause is in a dependency or external system
- Issue is already done or canceled

---

## Agents Used

| Phase | Agents |
|-------|--------|
| 2 | Explorer subagent — `agents/explorer.md` |
| 5 | Simplifier + Reviewer subagents — `agents/simplifier.md`, `agents/reviewer.md` |
