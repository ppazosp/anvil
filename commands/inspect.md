---
description: Project status dashboard — phase progress, branch diagrams, next launchable issues
argument-hint: Project name (e.g. "myapp")
---

# Inspect

Show the current state of a forged project: phase progress, branch diagrams with issue status, and what to launch next.

**Project:** $ARGUMENTS

---

## Step 1: Load Project Data

1. If `$ARGUMENTS` is empty: ask the user for the project name
2. Read `docs/specs/<project>/phases.md` for phase breakdown and issue IDs
3. Glob for `docs/specs/<project>/issues/*.md`
4. For each issue file: parse YAML frontmatter to extract `id`, `title`, `status`, `phase`, `branch`, `blocked_by`, `position`
5. Read CLAUDE.md for project context

---

## Step 2: Build Status Map

**For each phase, classify issues by state:**
- `done` — completed
- `in-progress` — currently being worked on
- `todo` + all `blocked_by` issues are `done` → **ready** (launchable)
- `todo` + has unfinished blockers → **blocked**
- `canceled` — skip from progress calculation

**Calculate per-phase:**
- Total issues, done count, in-progress count, blocked count
- Phase complete = all issues done
- Phase progress percentage

---

## Step 3: Display Dashboard

```
╔══════════════════════════════════════════════════╗
║  <PROJECT NAME> — Status Dashboard               ║
╚══════════════════════════════════════════════════╝

Phase 1: <name>  ████████████░░░░ 75% (3/4 done)
──────────────────────────────────────────────────

  data-model
  ✅ P1-01  Create user schema
  ✅ P1-02  Add relations and indexes
  │
  auth
  ✅ P1-03  Setup auth middleware
  │
  config
  🔄 P1-04  Environment and config setup
  │

Phase 2: <name>  ░░░░░░░░░░░░░░░░ 0% (0/5 done)
──────────────────────────────────────────────────

  api
  ⏳ P2-01  Create REST endpoints         ← READY
  ──▶ P2-02  Add validation middleware
  ──▶ P2-03  Rate limiting
  │
  ui
  ⏳ P2-04  Login page                    ← READY
  ──▶ P2-05  Dashboard layout
```

**Legend:**
- `✅` done
- `🔄` in-progress
- `⏳` todo, ready to launch (no unfinished blockers)
- `🔒` blocked (has unfinished blockers)
- `──▶` blocked by previous issue in branch
- `← READY` can be launched now

---

## Step 4: Next Actions

```
──────────────────────────────────────────────────
⚡ Ready to launch (2 issues, 2 parallel agents):

  /strike P2-01
  /strike P2-04

⏳ Waiting (3 issues, blocked):

  P2-02 ← waiting on P2-01
  P2-03 ← waiting on P2-02
  P2-05 ← waiting on P2-04
──────────────────────────────────────────────────
```

**If a phase just completed:**
```
🎉 Phase 1 complete! All 4 issues done.
   Ready to start Phase 2 — run /forge <project> to forge next phase, or launch ready issues above.
```

**If all phases complete:**
```
🎉 All phases complete! <total> issues across <N> phases.
```

---

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| No `phases.md` found | Glob all issue .md files, group by `phase` frontmatter field |
| Issue canceled | Show with `❌`, skip from progress calculation |
| All phases complete | Celebration message + total stats |
| No issues ready | Show what's blocking and estimated next unblock |
