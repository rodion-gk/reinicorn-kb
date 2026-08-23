---
type: spec
title: Submodule Pointer Staleness Fix
slug: submodule-pointer-staleness-fix
lifecycle: done
status: implemented
created: 2026-07-17
author: Michael Biehl
---

# Submodule Pointer Staleness Fix

Date: 2026-02-22
Issue: #7

## Problem

The harness submodule pointer in the parent repo can become stale after history
rewriting (rebase), causing `fatal: not our ref` on other machines. Three factors
contribute:

1. `sync` falls back to `rebase origin/main`, rewriting local SHAs
2. `publish` retries with `pull --rebase`, rewriting local SHAs
3. `ignore = all` in `.gitmodules` hides pointer drift from `git status`

## Design

### 1. Remove `ignore = all` from `.gitmodules`

Make pointer drift visible. `git status` will show `modified: harness` if the
local harness HEAD doesn't match the recorded pointer, acting as a safety net.

### 2. Auto-stage parent pointer in `commit_harness()`

After committing inside the harness submodule, run `git add harness` in the
parent repo. This stages the pointer — no parent commit is created. The pointer
gets picked up by the next parent commit naturally.

### 3. Replace rebase with merge

**`sync.py`:** Change the fallback from `rebase origin/main` to
`merge origin/main` (drop `--ff-only`). Remove `rebase --abort` cleanup.
Stage pointer after successful sync.

**`publish.py`:** Change retry from `pull --rebase` to `pull` (merge).
Remove the separate "optionally update parent pointer" block — now handled by
`commit_harness()`.

No SHA rewriting means no orphaned commits.

### 4. Auto-push worker stages parent pointer

After pushing harness, `harness_push_worker.py` runs `git add harness` in the
parent. Just stages — no commit from the background process.

### 5. Post-checkout hook stages pointer after init

After `git submodule update --init`, stage the pointer with `git add harness`.

## Files Changed

| File | Change |
|------|--------|
| `.gitmodules` | Remove `ignore = all` |
| `src/reins/harness.py` | Stage parent pointer in `commit_harness()` |
| `src/reins/commands/sync.py` | Merge fallback instead of rebase; stage pointer |
| `src/reins/commands/publish.py` | Merge retry instead of rebase; remove redundant pointer block |
| `src/reins/harness_push_worker.py` | Stage parent pointer after push |
| `src/reins/commands/internal/post_checkout.py` | Stage pointer after init |

## Trade-offs

- Merge commits in harness history (acceptable for docs/metadata repo)
- More frequent pointer staging (but no extra standalone commits)
- `git status` now shows harness pointer changes (feature, not bug)
