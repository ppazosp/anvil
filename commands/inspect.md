---
description: Project status dashboard — phase progress, heat diagrams, next launchable issues
argument-hint: Project name (e.g. "myapp")
---

# Inspect

Show the current state of a forged project: phase progress, heat diagrams with issue status, and what to launch next.

**Project:** $ARGUMENTS

---

## Step 1: Load Project Data

1. If `$ARGUMENTS` is empty: ask the user for the project name
2. Read `docs/specs/<project>/phases.md` for phase breakdown and issue IDs
3. Glob for `docs/specs/<project>/issues/*.md`
4. For each issue file: parse YAML frontmatter to extract `id`, `title`, `status`, `phase`, `heat`, `blocked_by`
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
  ✅ P1-001  Create user schema
  ✅ P1-002  Add relations and indexes
  │
  auth
  ✅ P1-003  Setup auth middleware
  │
  config
  🔄 P1-004  Environment and config setup
  │

Phase 2: <name>  ░░░░░░░░░░░░░░░░ 0% (0/5 done)
──────────────────────────────────────────────────

  api
  ⏳ P2-001  Create REST endpoints         ← READY
  ──▶ P2-002  Add validation middleware
  ──▶ P2-003  Rate limiting
  │
  ui
  ⏳ P2-004  Login page                    ← READY
  ──▶ P2-005  Dashboard layout
```

**Legend:**
- `✅` done
- `🔄` in-progress
- `⏳` todo, ready to launch (no unfinished blockers)
- `🔒` blocked (has unfinished blockers)
- `──▶` blocked by previous issue in heat
- `← READY` can be launched now

---

## Step 4: Next Actions

```
──────────────────────────────────────────────────
⚡ Ready to launch (2 issues, 2 parallel agents):

  /strike P2-001
  /strike P2-004

⏳ Waiting (3 issues, blocked):

  P2-002 ← waiting on P2-001
  P2-003 ← waiting on P2-002
  P2-005 ← waiting on P2-004
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
| No `phases.md` found | Glob all issue .md files, group by `phase` field in frontmatter |
| Issue canceled | Show with `❌`, skip from progress calculation |
| All phases complete | Celebration message + total stats |
| No issues ready | Show what's blocking and estimated next unblock |
