---
type: spec
title: 'Branch-agnostic kb: discover the kb default branch instead of hardcoding main'
slug: branch-agnostic-kb-discover-the-kb-default-branch-instead-of
lifecycle: active
status: in-review
created: 2026-08-15
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/14
---

# Branch-agnostic kb: discover the kb default branch instead of hardcoding main

## Problem

The kb tooling hardcodes `main`/`origin/main` in roughly ten call sites:
`kb.py` (ensure-on-main, fast-forward, publish/push, pull), `commands/sync.py`
(fetch + merge), `commands/status.py` (pointer-staleness check),
`review.py` (review-branch base, fetch), `commands/init.py` (bare-remote
seeding), and `kb_setup.py` (`seed_remote` forces `refs/heads/main`).

Two concrete failures fall out of this:

1. **Seeded remote with a stale `master` HEAD** (CodeRabbit finding on PR
   #48, `tests/test_kb_setup.py`): `seed_remote()` pushes `main` but a push
   cannot rewrite the bare remote's symbolic HEAD. If the empty remote was
   created under `init.defaultBranch=master`, the subsequent plain
   `git clone` in `setup_kb_clone()` checks out nothing ("remote HEAD refers
   to nonexistent ref"), yet the CLI reports "Kb cloned (tracking main)".
2. **Pre-existing kb remotes whose default branch is not `main`** (older
   repos on `master`, orgs standardized on `trunk`) half-work: the clone
   succeeds on their default branch, then every sync/publish/status
   operation fails against a nonexistent `origin/main`.

Patching only the clone (`-b main`) fixes failure 1 but deepens the
hardcoding; a clone-time discovery helper alone fixes neither, because the
other nine call sites still assume `main`. The branch assumption has to move
behind one seam.

## Design Goals

- One helper is the single source of truth for the kb branch name; no
  `"main"`/`"origin/main"` literal survives in kb-operation code outside it.
- A kb remote whose default branch is `master` or `trunk` works end to end:
  clone, sync, publish, status, review.
- The seeded-remote-with-`master`-HEAD case produces a working checkout,
  never a silent unborn-branch clone.
- Discovery cost is paid once, not on every git call.

## Design

- Add `kb_branch()` (in `reinicorn.kb` or a small module both `kb.py` and
  `kb_setup.py` can import without cycles):
  - **At clone time**: discover the remote default branch via
    `git ls-remote --symref <url> HEAD`. If the symref target does not exist
    on the remote (the stale-`master`-HEAD seeding case), fall back to the
    branch that does exist, preferring `main`. Clone with `-b <branch>`.
  - **Record** the discovered branch in `.reinicorn-config` next to the kb
    remote URL (migration precedence rules already established there apply).
  - **At operation time**: read the recorded branch; fall back to
    discovering from the existing clone (`git symbolic-ref
    refs/remotes/origin/HEAD`, then `origin/main`, then `origin/master`)
    for kbs set up before this spec, and backfill the config record.
- Thread `kb_branch()` through every call site enumerated in Problem;
  `seed_remote` keeps seeding on the discovered/target branch rather than a
  forced `main`.
- Structural test: no `origin/main` / `refs/heads/main` literal in
  `src/reinicorn` kb-operation modules outside the helper (allowlist the
  helper itself and non-kb uses such as `commands/plan.py`'s branch-name
  guard).
- Regression tests: the PR #48 finding's scenario (bare remote,
  `HEAD=refs/heads/master`, seed, clone → working checkout), plus an
  end-to-end sync/publish round-trip against a `master`-default remote.

## Non-Goals

- Supporting a kb whose working branch differs from the remote's default
  branch (the kb tracks the remote default, full stop).
- Renaming existing kb branches or migrating remotes to `main`.
- Branch-agnosticism for the *parent* repo (`commands/plan.py`'s
  `main`/`master` guard is about the parent and stays as is).
