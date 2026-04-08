---
description: Create a standalone issue — writes an issue .md file with proper frontmatter
argument-hint: Project name + title (e.g. myapp "Fix login redirect")
---

# Cast

Create a new standalone issue .md file. Use this for issues outside the `/forge` workflow — ad-hoc bugs, features, or tasks.

**Input:** $ARGUMENTS

---

## Steps

1. **Parse input:**
   - If `$ARGUMENTS` is empty: ask the user for project name, issue title, and kind (strike or mend)
   - Otherwise: extract project name (first word), title (quoted string), and optional `--kind` flag (defaults to `strike`)

2. **Determine the next ID:**
   - Glob for `docs/specs/<project>/issues/[0-9]*.md`
   - Find the highest existing 3-digit ID
   - Increment by 1 (or start at `001` if none exist)

3. **Gather details:**
   - Ask the user for a brief description of the issue
   - If `kind` is `mend`: ask what's broken, expected vs actual behavior
   - If `kind` is `strike`: ask what should be built, acceptance criteria
   - For each question, recommend an answer if you can infer from context

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
