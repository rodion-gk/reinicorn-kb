---
type: plan
title: 'Execution Plan: fix/path-scoped-create-commits'
slug: fix-path-scoped-create-commits
lifecycle: active
status: in-progress
created: 2026-07-31
author: Michael Biehl
branch: fix/path-scoped-create-commits
ticket: '[#35](https://github.com/crystldm/reinicorn/issues/35)'
spec: N/A
---

# Execution Plan: fix/path-scoped-create-commits
## Goal

Fix issue #35: per-artifact create commands (`idea create`, `spec/prd/debt/retro
create`, `principle add`, `plan create/complete`) commit the entire dirty kb
working tree via `commit_kb`'s `git add -A`, not just the doc(s) they touched.
Unrelated pre-existing kb modifications get swept into `idea:`/`plan:`/`doc:`
commits, corrupting the audit trail and silently publishing (or diverging)
in-review docs.

## Acceptance Criteria

- [ ] With an unrelated modified file in the kb working tree, each create
      command commits only the file(s) it created/modified; the unrelated
      change stays uncommitted in the working tree.
- [ ] `plan complete` still commits the full move (deletion from `active/`,
      addition under `completed/`).
- [ ] `kb publish` and the review flow keep their deliberate sweep-everything
      behavior, unchanged.
- [ ] Regression tests cover the dirty-tree scenario for the path-scoped
      callers.

## Approach

Add an optional `paths` parameter to `commit_kb(root, message, *, kb_dir=None,
paths=None)`. When `paths` is given, stage with `git add -A -- <pathspecs>`
(relative to the kb dir; `-A` with a pathspec stages deletions too, which the
`plan complete` move needs) and commit with `git commit -m msg -- <pathspecs>`
so nothing outside the pathspecs can land even if already staged. When `paths`
is omitted, behavior is unchanged (full sweep) for `publish`/`review`.

Callers pass what they touched:
- `idea.py` → the created idea file
- `doc_create.py` → the created/updated doc file
- `plan.py` create → the plan directory
- `plan.py` complete → both the old `active/` dir and the new `completed/` dir

## Tasks

- [ ] Red: tests that seed an unrelated dirty kb file and assert each create
      command's commit contains only its own path(s)
- [ ] Green: `commit_kb` `paths` parameter + pass paths at the four call sites
- [ ] Verify: pytest, ruff, pyright, tests/run-all.sh

## Dependencies

PR #30 (unified frontmatter) rewrites the doc-emitting code in these same
files; this change is deliberately small at the call sites to keep the
eventual rebase trivial. Issue #34 is fixed by PR #30, not here.
