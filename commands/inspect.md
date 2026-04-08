---
description: Project status dashboard — phase progress, heat diagrams, next launchable issues
argument-hint: Project name (e.g. "myapp")
---

# Inspect

Show the current state of a project: phase progress, issue status by heat, and what to launch next.

**Project:** $ARGUMENTS

---

## Step 1: Load Project Data

1. If `$ARGUMENTS` is empty: ask the user for the project name
2. Read `docs/specs/<project>/phases.md` for phase breakdown and issue IDs
3. Glob for `docs/specs/<project>/issues/*.md`
4. For each issue file: parse YAML frontmatter to extract `id`, `title`, `status`, `phase`, `heat`, `kind`, `blocked_by`

---

## Step 2: Build Status Map

**Classify each issue:**
- `done` — completed
- `in-progress` — currently being worked on
- `todo` + all `blocked_by` issues are `done` → **ready** (launchable)
- `todo` + has unfinished blockers → **blocked**
- `canceled` — skip from progress calculation

**Calculate per-phase:** total, done, in-progress, blocked, progress percentage.

---

## Step 3: Display Dashboard

**Use this compact horizontal format — one line per heat, chain issues with arrows:**

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

**Rules:**
- One line per heat — issues chained horizontally with `→`
- Heat name left-aligned, padded to align issue chains
- Icons: `✅` done, `🔄` in-progress, `⏳` ready, `🔒` blocked, `❌` canceled
- Progress bar: filled/empty blocks proportional to done/total, then `done/total`
- Ready-to-launch line at the bottom — one line, all launchable issues with their command (`/strike` or `/mend` based on `kind` field)
- If nothing ready: show what's blocking next (e.g. "⏳ P2-002 waiting on P2-001")
- Omit completed phases unless `--all` flag

**Phase completion:**
```
🎉 Phase 1 complete (4/4) — run /forge myapp to forge next phase
```

**All done:**
```
🎉 All phases complete — 25 issues across 4 phases
```

---

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| No `phases.md` found | Glob all issue .md files, group by `phase` field in frontmatter |
| Standalone issues (no phase) | Show under "Standalone" section |
| Issue canceled | Show with `❌`, skip from progress calculation |
| All phases complete | Celebration message + total stats |
| No issues ready | Show what's blocking and estimated next unblock |
