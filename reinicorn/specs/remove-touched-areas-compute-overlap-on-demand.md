---
type: spec
title: Remove touched-areas, compute overlap on demand
slug: remove-touched-areas-compute-overlap-on-demand
lifecycle: done
status: implemented
created: 2026-03-21
author: Claude
origin: ai-assisted
human_validated: false
---

# Remove touched-areas, compute overlap on demand

## Problem

The pre-push hook writes `touched-areas.md` to the harness submodule on every push, then spawns a background worker to push harness after the parent push completes. This creates two bugs:

1. **Infinite cycle:** Writing to harness dirties the submodule pointer, so the stop hook forces another commit+push, which triggers another touched-areas write, ad infinitum.
2. **Dangling submodule pointers:** The parent repo push completes before the background harness push finishes. Other agents and CI that clone the repo get a submodule pointer that doesn't exist on origin yet.

## Design Goals

- Overlap detection still works — agents can check if another branch touches the same files before starting work.
- Pre-push hook never writes to harness.
- Harness is pushed before the parent repo, not after — no dangling pointers.
- Background push worker is eliminated.

## Design

### 1. Delete touched-areas.md

Remove `_update_touched_areas()` from `pre_push.py`. Delete all existing `touched-areas.md` files from harness. Remove any code that reads these files (e.g. `check_overlap()` in `harness.py`).

### 2. Compute overlap on demand from git

Replace the file-based overlap check with direct git queries. When an agent starts work on a branch (during plan creation or branch start):

1. `git fetch origin`
2. List remote feature branches: `git branch -r --list 'origin/claude/*'`
3. For each branch: `git diff --name-only origin/main...origin/{branch}`
4. Compare file lists to detect overlap with the current branch's planned changes
5. Warn the agent if overlap is found

This runs only at branch start, not on every push. Cost is N git-diff commands where N is the number of active branches (typically < 10).

### 3. Simplify pre-push hook

The pre-push hook's only remaining job: push harness synchronously before the parent push.

1. Check if harness has unpushed commits (local HEAD vs `origin/main`).
2. If yes: push harness inline. If that fails, warn but still exit 0 (never block pushes).
3. If no: no-op.

Remove `_schedule_harness_push()` and delete `harness_push_worker.py` entirely.

### 4. What stays the same

- Post-checkout hook (harness init on new worktrees)
- Post-merge hook (archiving stale plans)
- How agents write design docs / plans to harness (intentional harness mutation)

## Summary of changes

| Component | Before | After |
|---|---|---|
| `touched-areas.md` | Written on every push | Deleted |
| `_update_touched_areas()` | In pre_push.py | Removed |
| `harness_push_worker.py` | Background worker post-push | Deleted |
| `_schedule_harness_push()` | Spawns background process | Removed |
| Pre-push hook | Writes touched-areas + spawns background push | Synchronous harness push only |
| Overlap detection | Reads touched-areas.md from harness | `git diff --name-only` against remote branches |
| `check_overlap()` | Reads harness files | Queries git directly |

## Non-Goals

- Changing how agents write design docs or plans to harness.
- Modifying post-checkout or post-merge hooks.
- Real-time overlap detection during development (only checked at branch start).
