---
type: idea
title: Redesign submodule management — stop using raw git submodule commands in hooks.
slug: 2026-02-17-redesign-submodule-management-stop-using-raw-git-submodule-c
lifecycle: done
status: resolved
created: 2026-02-17
author: mnbiehl
---

# Redesign submodule management — stop using raw git submodule commands in hooks. 

**Resolved:** 2026-05-05

## Resolution

Landed via the `feature-submodule-management` exec plan. `reins kb sync` and
`reins kb publish` are the canonical interface; raw `git submodule update`
was removed from post-checkout / post-merge hooks; the M5 hardening pass
added auto-commit/auto-push for kb mutations. Subsequent commands
(`doc create`, `idea`, `plan create`, etc.) auto-commit kb changes via
`commit_kb()`.

- Plan: `kb/reins/exec-plans/completed/feature-submodule-management/`
- Design: `kb/reins/design-docs/submodule-management.md`

## Description

Redesign submodule management — stop using raw git submodule commands in hooks. post-checkout and post-merge hooks should not run git submodule update (resets to detached HEAD, clobbers in-progress work). reins kb sync/kb publish should be the only interface for harness movement. Consider configuring submodule to track main branch, or replacing submodule with subtree/other approach.

## Key insight

The harness is docs/plans, not code. It should **always be latest main**, never pinned to a specific commit. Reproducibility doesn't matter here.

## Concrete plan

1. Set `branch = main` and `ignore = all` in `.gitmodules` — stops git from tracking the pointer commit, suppresses "modified content" noise in `git status`
2. `reins kb sync` does `git submodule update --remote --merge` to pull latest main
3. Remove `git submodule update` from post-checkout and post-merge hooks entirely — these reset to detached HEAD and clobber work
4. `reins kb publish` (or a new push workflow) should commit+push inside the harness submodule to its main branch, then push the parent — no more detached HEAD commits
5. Pre-push hook verifies harness commits are pushed before allowing parent push (see sibling idea)
6. Never commit submodule pointer updates in the parent repo — the pointer becomes meaningless with `ignore = all`

## Every reins command should auto-commit and auto-push the harness

Any `reins` command that modifies harness files (`plan create`, `plan complete`, `idea`, `sync`, `lint` fixes, etc.) should automatically commit and push those changes to the harness remote. The user should never have to manually `git add/commit/push` inside the submodule. The whole point of the tooling is to hide the git plumbing.
