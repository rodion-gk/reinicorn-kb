# Worktree-aware kb resolution

**Date:** 2026-07-19
**Author:** Michael Biehl
**Status:** approved
**Origin:** ai-assisted
**Human-validated:** false
**Review-PR:** https://github.com/crystldm/reinicorn-kb/pull/1

## Problem

`git worktree add` never initializes submodules, so a fresh linked worktree
starts with an empty `kb/`. The existing `post-checkout` hook
(`rcorn _post-checkout`) already heals this by running
`git submodule update --init kb` — but in a worktree that performs a **full
network clone** into the worktree's private module dir
(`.git/worktrees/<name>/modules/kb`), even though the main checkout's module
(`.git/modules/kb`) already has every object. Wasteful and slow, and it fails
entirely when offline.

Alternatives considered and rejected:
- **Symlinking `kb/`** into the worktree — tested; git reports a `T`
  typechange, `git add -A` would commit a symlink over the gitlink, and
  `reset --hard` silently deletes the link.
- **kb.py redirection** (leave worktree kb uninitialized, resolve the main
  checkout's kb at CLI runtime) — works, but adds path-resolution complexity
  to `get_kb_dir` and loosens its containment check; unnecessary given the
  hook already exists. Decision: no special handling in `kb.py`
  (review discussion on PR #1).

## Design Goals

- A fresh linked worktree gets a working kb automatically (via the existing
  post-checkout hook), with **no network fetch** — objects are borrowed from
  the main checkout's module clone via `--reference`.
- `kb.py` is untouched: every worktree has a real initialized kb submodule,
  so `get_kb_dir` works unchanged.
- Behavior on fresh clones (no reference available) is unchanged: plain
  `--init` network clone as today.
- `rcorn hooks install` lands hooks where git actually reads them, whether
  run from the main checkout or a linked worktree.

Accepted costs (explicit decision): worktrees with an initialized kb cannot
`git worktree move` and need `git worktree remove --force`. We don't move
worktrees, and `--force` on remove is fine.

## Design

### 1. `--reference` in the post-checkout kb init

In `src/reinicorn/commands/internal/post_checkout.py`, the existing kb-init
block gains a reference to the shared module when one exists:

1. Resolve the common git dir: `git rev-parse --git-common-dir`, made
   absolute via `Path.resolve()` (not `--path-format=absolute`, which would
   require git ≥ 2.31 for no benefit).
2. Candidate reference: `<common-dir>/modules/kb`.
3. If that directory exists, run
   `git submodule update --init --reference <common-dir>/modules/kb -- kb`;
   otherwise run the current plain `--init` (fresh-clone case — the module
   dir being created *is* `<common-dir>/modules/kb`, so the guard naturally
   selects plain init there).
4. Everything after init is unchanged (`ensure_kb_on_main`,
   `stage_kb_pointer`).

`git submodule update --reference` dates to git 1.6.4 (2009), so no version
floor. The borrowed clone uses alternates; both sides track the same
`origin/main` and the main module is never pruned aggressively, so object
loss is not a practical concern (`--dissociate` remains available manually
if that ever changes).

### 2. Hook install destination

`cmd_hooks_install` currently writes git hooks to `rev-parse --git-dir`,
which inside a linked worktree is `.git/worktrees/<name>` — a location git
never reads hooks from. Switch to `rev-parse --git-common-dir` so the hooks
land in the shared `hooks/` dir regardless of where the command runs. This
is load-bearing for the design: the post-checkout hook must fire in every
worktree.

### 3. Docs

The `using-git-worktrees` skill (Step 2, project setup) notes that kb
initializes automatically via the post-checkout hook, and gives
`git submodule update --init kb` as the manual fallback when hooks are not
installed.

`session-start.sh` needs no change: its existing `kb/.git` guard covers
initialized worktrees, and the remote block already handles hook-less CI.

## Non-Goals

- No changes to `kb.py` / `get_kb_dir` — worktrees always have a real kb.
- No symlinking of `kb/` (tested; fights git — see Problem).
- No support for `git worktree move` with an initialized kb, and no
  `--force`-avoidance work for `remove` — accepted costs, see Design Goals.
- No locking/coordination for concurrent kb writes across worktrees — the
  same race already exists between any two sessions and is handled by
  rebase-on-publish.
