---
description: Spec → phases → issue .md files, optimized for parallel agent execution
argument-hint: Project name (e.g. "myapp") or project name + brief description
---

# Forge

Transform a project specification into perfectly defined issue .md files, organized by phases with parallelism optimization for multi-agent execution.

**Input:** $ARGUMENTS

---

## Phase 0: Detect State & Load Context

**Goal:** Determine if this is new or a continuation, load all available context

**Actions:**

1. **Parse input:**
   - If `$ARGUMENTS` is empty: ask the user for project name + brief description
   - Otherwise: extract project name (first word) and optional description (rest)

2. **Re-entry check:**
   - Look for `docs/specs/<project>/phases.md`
   - If it exists, read it and count `[x]` (done) vs `[ ]` (pending) phases
   - If completed phases exist:
     - Present status: "Phases 1-2 forged. Phase 3 pending."
     - Ask: "Continue with Phase N?" or "Redo a specific phase?"
     - If continuing: skip to **Phase 3** for that phase
   - Also check if phase spec files exist but phase is unchecked (interrupted run) — ask: "Found spec for Phase N but issues not created. Use existing spec or redo?"

3. **Load project context:**
   - Read `CLAUDE.md` for: team table, stack, conventions
   - Read `docs/specs/<project>/general.md` if it exists (re-entry)

4. **Codebase exploration (if code exists — skip for greenfield):**
   - Launch 2-3 **explorer** subagents IN PARALLEL (see `agents/explorer.md`):
     - "Map overall architecture: services, modules, boundaries, entry points"
     - "Identify patterns, abstractions, conventions, and extension points"
     - "Document data model, API surface, external integrations, and tech debt"
   - Synthesize findings — these inform all subsequent phases

**If re-entering and general spec + phases already exist, skip directly to the pending phase.**

---

## Phase 1: General Specification

**Goal:** Extract a complete, unambiguous project specification through relentless interrogation

**CRITICAL: Be aggressive. Do NOT accept vague answers. Push on every assumption.**

**Process:**

1. Start with whatever description the user provided
2. Interview relentlessly about every aspect of the project:
   - Walk down each branch of the decision tree, one question at a time
   - For each question, **provide your recommended answer** based on codebase exploration and domain knowledge
   - If a question can be answered by exploring the codebase, **explore instead of asking**
   - Resolve each branch completely before moving to the next

3. **Cover these branches (skip what's already clear):**
   - **Purpose:** What problem does this solve? For whom? Why now?
   - **Core features:** What are the main capabilities? What's explicitly OUT of scope?
   - **Data model:** Key entities, relationships, constraints
   - **Integrations:** External systems, APIs, data sources
   - **User flows:** Main user journeys from start to finish
   - **Constraints:** Performance, security, compliance, tech limitations
   - **Non-functional:** Scalability, accessibility, monitoring, error handling strategy

4. When ALL branches are resolved, present the complete spec summary
5. Get explicit user approval: "Is this spec complete and accurate?"

**Output:** Write to `docs/specs/<project>/general.md`:

```markdown
# <Project Name> — General Specification

> Status: current | Created: YYYY-MM-DD | Last updated: YYYY-MM-DD

## Overview
[What it is, who it's for, why it exists — 1-2 paragraphs]

## Core Features
[Bulleted list of main capabilities with brief descriptions]

## Out of Scope
[Explicitly excluded to prevent scope creep]

## Data Model
[Key entities, relationships, constraints]

## Integrations
[External systems, APIs, services, data flows]

## User Flows
[Main journeys described step by step]

## Constraints & Non-Functional Requirements
[Performance, security, scalability, accessibility]

## Key Decisions
[Decisions made during interrogation with rationale]
```

**Commit:** `feat[specs]: add general specification for <project>`

---

## Phase 2: Phase Division

**Goal:** Break the project into ordered, deliverable phases

**Actions:**

1. **If user already has phases:** listen, validate, challenge if needed
2. **If not:** propose phases based on:
   - Technical dependencies (what must exist before what can be built)
   - Deliverable value (each phase produces something testable/demoable)
   - Complexity balance (no phase 10x bigger than others)
   - Natural code boundaries (backend foundation → API → frontend)

3. **Present phase breakdown with dependencies:**

```
Phase 1: <name> — <description>
  Depends on: nothing

Phase 2: <name> — <description>
  Depends on: Phase 1 (needs X from it)

Phase 3: <name> — <description>
  Depends on: Phase 1 (not Phase 2 — can run in parallel with Phase 2)
```

4. **Interrogate:** Challenge the user on phase boundaries. Are phases too big? Too coupled? Can any run in parallel at the phase level?

5. Get user approval

**Output:** Write to `docs/specs/<project>/phases.md`:

```markdown
# <Project Name> — Phase Breakdown

> Status: current | Created: YYYY-MM-DD | Last updated: YYYY-MM-DD
> References: [general.md](general.md)

## Phases

- [ ] **Phase 1: <name>** — <description>
  - Dependencies: none
  - Estimated issues: ~N

- [ ] **Phase 2: <name>** — <description>
  - Dependencies: Phase 1
  - Estimated issues: ~N

- [ ] **Phase 3: <name>** — <description>
  - Dependencies: Phase 1
  - Estimated issues: ~N
```

**Commit:** `feat[specs]: add phase breakdown for <project>`

---

## Phase 3: Phase Deep-Dive

**Goal:** Detailed specification + acceptance criteria for the current phase

**CRITICAL: Process ONE phase at a time. After creating issues, ask "next phase?"**

**Announce:** "Forging Phase N: <name>"

**Process:**

1. Read the general spec for context
2. Interview aggressively about THIS phase only:
   - Exact requirements and expected behaviors
   - Edge cases and error scenarios
   - What "done" looks like — specific, testable criteria
   - Which parts of the codebase will be affected (reference exploration results)
   - For each question, provide your recommended answer
   - Explore codebase instead of asking when possible
   - One question at a time, resolve completely

3. **Define acceptance criteria that are:**
   - Testable (can write a test or manually verify)
   - Specific (no "should work correctly")
   - Complete (cover happy path + key error paths)

4. Present phase spec summary, get user approval

**Output:** Write to `docs/specs/<project>/phase-N-<name>.md`:

```markdown
# Phase N: <Name>

> Status: current | Created: YYYY-MM-DD | Last updated: YYYY-MM-DD
> References: [general.md](general.md) | [phases.md](phases.md)

## Objective
[What this phase delivers — 1-2 sentences]

## Requirements

### R1: <requirement name>
[Description, expected behavior, edge cases]

### R2: <requirement name>
[Description, expected behavior, edge cases]

## Acceptance Criteria
- [ ] AC1: <specific testable criterion>
- [ ] AC2: <specific testable criterion>

## Technical Notes
[Patterns to follow, files affected, implementation hints from codebase exploration]

## Decisions
[Phase-specific decisions with rationale]
```

**Commit:** `feat[specs]: add phase N specification for <project>`

---

## Phase 4: Issue Decomposition & Parallelism

**Goal:** Break the phase into issues optimized for parallel agent execution

**Actions:**

1. **Identify independent workstreams (heats):**
   - Group work by code area / domain (e.g. `data-model`, `auth`, `api`, `ui`)
   - Issues in the same heat share files/state → must be sequential
   - Issues in different heats touch different code → can run in parallel
   - Minimize cross-heat dependencies

2. **Define each issue:**
   - **Title:** action verb + what (e.g. "Create user schema with roles and permissions")
   - **Acceptance criteria:** inherited from phase spec, scoped to this issue
   - **Files likely affected:** specific paths from codebase exploration
   - **Priority:** inferred from dependency order (blocking issues = higher priority)
   - **Heat name:** domain name (e.g. `data-model`, `auth`, `api`)

3. **Granularity check — each issue MUST be:**
   - Executable by a single `/strike` run
   - Touches 1-8 files typically
   - Has clear, testable acceptance criteria
   - Implementable without human decisions mid-run
   - Independent enough that an agent can work on it without context from other in-progress issues

4. **Granularity red lines:**
   - **Too big → split:** "Build the auth system" → schema, middleware, login endpoint, registration, token refresh
   - **Too small → merge:** "Add field X" + "Add field Y" + "Add field Z" → "Create schema with fields X, Y, Z"
   - **Too coupled → redesign heats:** if issue A needs output of issue B in a different heat, they belong in the same heat

5. **Present parallelism diagram BEFORE creating anything:**

```
Phase N: <Name>
├── <label> (N issues, sequential)
│   ├── #1: <title> ──blocks──▶ #2
│   └── #2: <title>
├── <label> (N issues, sequential)
│   ├── #3: <title> ──blocks──▶ #4
│   └── #4: <title>
└── <label> (N issues, independent)
    └── #5: <title>

Max parallelism: N agents
   Agent 1: <label> (#1 → #2)
   Agent 2: <label> (#3 → #4)
   Agent 3: <label> (#5)
```

6. **Get explicit user approval** on the issue breakdown, heat assignment, and parallelism plan

**DO NOT create issue files until user confirms.**

---

## Phase 5: Create Issue Files

**Goal:** Write issue .md files with YAML frontmatter

**Actions:**

1. **For each issue (in dependency order, blockers first):**

   Write to `docs/specs/<project>/issues/P<N>-<NNN>-<slug>.md`:

   ```markdown
   ---
   id: P<N>-<NNN>
   title: <title>
   status: todo
   kind: strike
   phase: <N>
   heat: <heat-name>
   priority: <1-4>
   blocked_by: [<list of IDs>]
   created: <YYYY-MM-DD>
   updated: <YYYY-MM-DD>
   ---

   # P<N>-<NNN>: <Title>

   ## Objective
   <what this issue accomplishes — 1-2 sentences>

   ## Requirements
   <specific requirements from phase spec relevant to this issue>

   ## Acceptance Criteria
   - [ ] <criterion 1>
   - [ ] <criterion 2>
   - [ ] <criterion 3>

   ## Files Likely Affected
   - `<path/to/file>` — <what changes>

   ## Context
   Phase spec: `docs/specs/<project>/phase-N-<name>.md`
   General spec: `docs/specs/<project>/general.md`
   Heat: `<heat-name>`
   ```

2. **Update `phases.md`:** mark this phase `[x]` and annotate with issue IDs:

   ```markdown
   - [x] **Phase 1: <name>** — <description>
     - Issues: P1-001, P1-002, P1-003, P1-004
     - Heats: data-model (P1-001→P1-002), auth (P1-003), config (P1-004)
   ```

3. **Commit:** `feat[specs]: forge phase N issues for <project>`

4. **Present summary with IDs:**

```
Phase N: <Name> — FORGED

├── <heat> (sequential)
│   ├── P1-001: <title> ──blocks──▶ P1-002
│   └── P1-002: <title>
├── <heat> (sequential)
│   └── P1-003: <title>
└── <heat> (independent)
    └── P1-004: <title>

Ready to launch 3 agents in parallel:
   /strike P1-001  (then P1-002 after)
   /strike P1-003
   /strike P1-004
```

---

## Phase 6: Next Phase or Finish

**If more phases remain:**
- Ask: "Ready to forge Phase N+1: <name>? Or stop here and continue later with `/forge <project>`?"
- If yes → go to **Phase 3**
- If no → stop. User can re-enter later.

**If all phases complete:**
- Present full project map with all phases, heats, and issue counts
- Update `phases.md` with final status
- Commit: `feat[specs]: complete project forge for <project>`

---

## Red Flags — STOP and ask user if:

- Phase spec contradicts general spec
- A single heat has 5+ sequential issues (split into sub-heats)
- Cross-heat dependencies are unavoidable (redesign heats)
- Issue requires architectural decisions not in specs (go back to spec phase)
- Phase has >10 issues (consider splitting the phase)
- Phase has 1-2 issues (consider merging with adjacent phase)

---

## Autonomous Decision Guidelines

| Situation | Default Decision |
|-----------|------------------|
| Heat naming | Domain names: `data-model`, `auth`, `api`, `ui`, `config` |
| Issue priority | Blockers get higher priority, leaf issues get lower |
| Phase size | Target 3-8 issues per phase |
| Heat size | Target 1-4 issues per heat |
| Acceptance criteria | 2-5 per issue, always testable |
| Spec format | Follow the templates in this command exactly |
| Commit messages | `feat[specs]: <action> for <project>` |

---

## Agents Used

| Phase | Agents |
|-------|--------|
| 0 | Explorer subagents (2-3 parallel) — `agents/explorer.md` |
| 1-4 | None (interrogation + analysis) |
| 5 | None (file writes only) |
