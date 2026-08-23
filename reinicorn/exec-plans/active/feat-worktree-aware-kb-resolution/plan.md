---
type: plan
title: 'Execution Plan: feat/worktree-aware-kb-resolution'
slug: feat-worktree-aware-kb-resolution
lifecycle: done
status: implemented — verified end-to-end (hook auto-init via alternates in a real worktree); awaiting code review + PR
created: 2026-07-19
author: Michael Biehl
branch: feat/worktree-aware-kb-resolution
ticket: N/A
spec: specs/worktree-aware-kb-resolution.md
---

# Execution Plan: feat/worktree-aware-kb-resolution

## Goal

Implement spec `kb/reinicorn/specs/worktree-aware-kb-resolution.md`: the
post-checkout kb auto-init borrows objects from the main checkout's module
clone via `--reference` (no network re-clone in worktrees), and
`rcorn hooks install` writes git hooks to the shared common dir so the hook
fires in every worktree. `kb.py` resolution logic untouched.

## Acceptance Criteria

- [x] `rcorn _post-checkout` in a fresh linked worktree initializes kb with
      `--reference <git-common-dir>/modules/kb` (alternates file present in
      the worktree's module gitdir).
- [x] Fresh-clone behavior unchanged: no `<common>/modules/kb` → plain
      `--init` (no `--reference` args).
- [x] `rcorn hooks install` run from inside a linked worktree lands git hooks
      in the shared `<common>/hooks/`, not `.git/worktrees/<name>/hooks/`.
- [x] `using-git-worktrees` skill Step 2 documents auto-init + manual fallback.
- [x] All new behavior covered by tests (golden principle 7); full suite green.

## Approach

Two code changes, both TDD, one commit each, plus a doc commit:

1. New helper `kb_reference_args(root)` in `src/reinicorn/kb.py` — resolves
   `git rev-parse --git-common-dir` (made absolute via `Path.resolve()`, not
   `--path-format=absolute`, to avoid a git ≥ 2.31 floor), returns
   `["--reference", "<common>/modules/kb"]` when that dir exists, else `[]`.
   `cmd_post_checkout` splices it into the existing
   `submodule update --init` call. Fresh clones naturally get `[]` because
   `<common>/modules/kb` is the dir the clone is about to create.
2. `cmd_hooks_install` switches `rev-parse --git-dir` → `--git-common-dir`
   for the git-hooks destination.
3. Skill doc note in `.agents/skills/using-git-worktrees/SKILL.md` Step 2.

Tests build real repos via the existing `submodule_repo` fixture
(`tests/conftest.py`) + `git worktree add`; patch
`reinicorn.commands.internal.post_checkout.hook_check` to `True`.
`--reference` proof = `objects/info/alternates` in the worktree module gitdir
pointing at the main module's objects.

## Tasks

### Task 1: `kb_reference_args` + wire into post-checkout

**Files:**
- Modify: `src/reinicorn/kb.py` (new helper near `get_kb_dir`)
- Modify: `src/reinicorn/commands/internal/post_checkout.py:32-33`
- Test: `tests/test_post_checkout_worktree.py` (new)

**Steps:**
- [x] Write failing tests: `kb_reference_args` returns `[]` when
      `<common>/modules/kb` is absent; returns the `--reference` pair inside a
      linked worktree; `cmd_post_checkout(["", "", "1"])` in a fresh worktree
      of `submodule_repo` initializes kb with an alternates file pointing at
      `<parent>/.git/modules/kb/objects`.
- [x] Run: `pytest tests/test_post_checkout_worktree.py -v` → FAIL (ImportError).
- [x] Implement `kb_reference_args` in `kb.py`; splice into `post_checkout.py`.
- [x] Run same tests → PASS. Run `pytest` full suite → green.
- [x] Commit: `feat(kb): borrow objects via --reference in post-checkout kb init`

### Task 2: hooks install to git-common-dir

**Files:**
- Modify: `src/reinicorn/commands/hooks_install.py:43` (`--git-dir` →
  `--git-common-dir`)
- Test: `tests/test_post_checkout_worktree.py` (add test)

**Steps:**
- [x] Write failing test: repo + `git worktree add`, chdir into worktree,
      `cmd_hooks_install()`, assert `post-checkout` exists in the main
      `.git/hooks/` and `.git/worktrees/<name>/hooks/` was not created.
- [x] Run it → FAIL (hook lands in per-worktree gitdir).
- [x] Swap the rev-parse flag.
- [x] Run test + full suite → green.
- [x] Commit: `fix(hooks): install git hooks to the shared common dir`

### Task 3: skill doc note

**Files:**
- Modify: `.agents/skills/using-git-worktrees/SKILL.md` (Step 2)

**Steps:**
- [x] Add note: reinicorn repos auto-init kb via the post-checkout hook
      (`--reference`, no network); manual fallback
      `git submodule update --init kb` when hooks aren't installed.
- [x] Commit: `docs(skill): note kb auto-init in worktree setup`

## Dependencies

Spec approved and landed: `kb/reinicorn/specs/worktree-aware-kb-resolution.md`
(review PR crystldm/reinicorn-kb#1). No overlap with other active branches.
