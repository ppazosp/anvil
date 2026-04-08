---
description: Manually complete an issue — append completion summary, mark done
argument-hint: Issue ID (e.g. P1-001, 003)
---

# Quench

Mark an issue as done after manual implementation. Appends a completion summary to the issue .md file and updates its status.

Use this when you implemented something outside of `/strike` or `/mend` and want to close the issue.

**Issue ID:** $ARGUMENTS

---

## Steps

1. If `$ARGUMENTS` is empty: ask the user for the issue ID

2. **Find the issue file:** glob for `docs/specs/*/issues/$ARGUMENTS-*.md` or `docs/specs/*/issues/*$ARGUMENTS*.md`

3. **Read the issue file**, parse frontmatter and body

4. **Gather completion context:**
   - `git log --oneline` for recent commits related to this work
   - Identify files created/modified
   - Note any deviations from the spec

5. **Update frontmatter:**
   - `status: done`
   - `updated: <today>`

6. **Append `## Completion Summary`:**

   ```markdown
   ## Completion Summary

   **Commit:** `<short-hash>` — `<commit message>`

   ### What was built
   - <bullet points>

   ### Files created/modified
   - `<path>` — <description>

   ### Decisions
   - <any deviations from spec with rationale>
   ```

7. **Update phase spec (if forged issue):**
   - Read the referenced phase spec file
   - Mark completed acceptance criteria as `[x]`
   - Update `Last updated` date

8. **Commit:** `feat[specs]: complete <ISSUE_ID>`
