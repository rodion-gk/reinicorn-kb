---
type: plan
title: 'Execution Plan: feat-commit-skillset-lock'
slug: feat-commit-skillset-lock
lifecycle: active
status: in-progress
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-commit-skillset-lock
ticket: N/A
spec: 'specs/skill-base-agnostic-reinicorn-adapter-infrastructure-for-ext.md'
---

# Execution Plan: feat-commit-skillset-lock

## Goal
Make `.reinicorn/skillset-lock.json` the committed, declared methodology of
a project — and make a clone that has the lock but not the (gitignored)
adapter skill files a state Reinicorn repairs, instead of one where
`rcorn skills status` lists every file as missing and `rcorn skills
install <name>` collides on whatever is left.

The approved spec gitignores the adapter-installed *files* in this repo and
is silent on the lock; this branch settles that gap: lock committed, files
restorable from it.

## Acceptance Criteria
- [x] `rcorn skills install` with no argument restores the adapter the lock
  records: fetches the lock's pin (digest-checked against the lock), writes
  back only the missing files, each verified to hash as the lock says,
  regenerates the wiring doc and the compatibility link. Never rewrites the
  lock; never touches a path that exists (a locally edited sibling neither
  blocks nor is clobbered). A lock that no longer matches what the adapter
  stages is refused with a pointer at `rcorn skills update`.
- [x] `rcorn update`, the hooks-only `rcorn init` (teammate clone), and the
  post-checkout hook run the same restore when lock-recorded files are
  missing; silent when nothing is missing; a failed restore is reported
  with the retry command and never fails the command or the checkout.
- [x] One enforcing seam: `reinicorn.skillset.restore.ensure_adapter_files`
  is the only restore entry point those callers use.
- [x] This repo dogfoods it: `.gitignore` stops ignoring the lock and the
  wiring doc; both are committed from a fresh `skills install superpowers`.
- [x] README, GETTING-STARTED, `using-reinicorn`, and an `upgrades/v0.4.md`
  note say: commit the lock; files may be gitignored; how restore runs.
- [x] Gate green: `uv run pytest tests/` (floor 87%), ruff, pyright,
  `bash tests/run-all.sh`; live restore verified in this worktree.

## Approach
Restore is deliberately narrower than `update_adapter`: it reuses
`fetch_source` + `build_staging` but copies only lock-recorded paths absent
from disk, so a drifted sibling cannot abort it (update's force gate would)
and nothing present is ever replaced. `locked_adapter` (re-resolve by name,
re-pin to the lock's commit) moves out of `cmd_skills_update` into the
restore module so update and restore share it. `RestoreOutcome` is an Enum
(golden principle 15); every error carries what/where/how-to-fix.

## Tasks
- [x] `skillset/restore.py`: `missing_files`, `restore_from_lock`,
  `ensure_adapter_files`, `locked_adapter`, `resolve_adapter_dir`
- [x] `rcorn skills install [<name>]` — optional argument → restore
- [x] Wire `ensure_adapter_files` into `update`, `init` (hooks-only), and
  `_post-checkout`
- [x] Tests: `tests/skillset/test_restore.py` + command-level tests in
  `test_skills_cmds`, `test_update`, `test_post_checkout`, `test_init`;
  shared `fake_skillset_fetch` fixture
- [x] `.gitignore`, committed lock + wiring doc, docs, upgrade note

## Dependencies
Independent of PR #59 (mattpocock-skills adapter). After #59 merges and
this repo swaps its install, the committed lock simply changes in that
commit — which is the point.
