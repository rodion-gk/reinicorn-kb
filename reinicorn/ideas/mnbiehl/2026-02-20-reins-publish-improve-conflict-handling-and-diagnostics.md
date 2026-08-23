---
type: idea
title: 'reins kb publish: improve conflict handling and diagnostics'
slug: 2026-02-20-reins-publish-improve-conflict-handling-and-diagnostics
lifecycle: done
status: resolved
created: 2026-02-20
author: mnbiehl
---

# reins kb publish: improve conflict handling and diagnostics

**Resolved:** 2026-05-05

## Resolution

Landed in M5 publish/sync hardening. `reins kb publish` now reports
diverged commits, runs rebase non-interactively, and prints actionable
resolution steps on failure. Pull-first sync paths and edge-case tests
were added.

- Plan: `kb/reins/exec-plans/active/feature-m5-publish-hardening/`
- Design: `kb/reins/design-docs/m5-publish-sync-hardening.md`

## Description

reins kb publish: improve conflict handling and diagnostics

## Problem
When `reins kb publish` encounters a diverged remote, it outputs:
```
Push failed, pulling latest and retrying...
Push failed after retry. Resolve manually.
```

This provides no actionable information. The user has to manually investigate with git commands.

## Issues Encountered
1. **No indication of what diverged** — doesn't show which remote commits we're missing or what local commits we have
2. **Rebase conflicts hang on editor** — when `git rebase --continue` needs a commit message, it opens an editor and hangs in non-interactive terminals
3. **Simple conflicts could be auto-resolved** — parallel appends to markdown tables (common in harness files) are trivially resolvable
4. **No pre-flight sync check** — `reins idea` commits locally even when harness is significantly behind origin; could warn or auto-sync first

## Suggested Improvements
1. **Better diagnostics on failure:**
   - Show `git log --oneline origin/main..HEAD` (local commits)
   - Show `git log --oneline HEAD..origin/main` (remote commits we're missing)
   - Identify conflicting files if rebase fails

2. **Non-interactive rebase handling:**
   - Set `GIT_EDITOR=:` or `GIT_EDITOR=true` when running rebase
   - Or use `--no-edit` flag where appropriate

3. **Auto-resolve simple conflicts:**
   - Detect parallel appends to markdown lists/tables
   - Apply simple resolution (keep both) and continue
   - Flag for human review only if heuristics fail

4. **Pre-operation sync check:**
   - Before `reins idea`, check if harness is behind origin
   - Offer to sync first, or warn that publish may need manual resolution

## Recommended Approach: Pull-First

The simplest fix is probably: **always `git pull --rebase` before any harness-mutating operation** (idea, plan create, etc.).

### Why this works

Current flow accumulates divergence:
```
idea → commit locally → (drift accumulates) → publish → push fails → pull --rebase → conflicts
```

Pull-first keeps you synchronized:
```
idea → pull --rebase → commit on top of latest → push → (usually succeeds)
```

### What pull-first solves
- Prevents accumulated local commits that diverge from origin
- Your commit is always on top of latest, so push almost always succeeds
- Conflicts become rare (only race conditions where someone pushes between your pull and push)

### What you still need regardless
1. `GIT_EDITOR=:` for non-interactive environments (conflicts still require this)
2. Retry logic for rare race conditions
3. Good diagnostics when conflicts do happen (fallback, not primary path)

### Testing needed
Options to evaluate before implementing:
- Pull-first on every mutating command vs. only on `publish`
- Whether to auto-push after each idea (immediate sync) vs. batch with explicit publish
- Handling of network failures during pull (offline mode?)
- Performance impact of pull on every operation

Pull-first is probably the 90% solution, but edge cases need testing before committing to the approach.
