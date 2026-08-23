---
type: plan
title: 'Execution Plan: fix/hooks-stale-repair'
slug: fix-hooks-stale-repair
lifecycle: done
status: complete
created: 2026-07-31
author: Michael Biehl
branch: fix/hooks-stale-repair
ticket: '[#24](https://github.com/crystldm/reinicorn/issues/24)'
spec: N/A
---

# Execution Plan: fix/hooks-stale-repair
## Goal

Fix issue #24: `reins`-era git hooks silently no-op (the `command -v reins`
guard fails, hook exits 0, kb protection is gone with no signal), and
`rcorn hooks install` cannot repair them — it appends the current hook after
the stale hook's unconditional `exit 0`, making the appended content
unreachable while reporting `APPENDED` as a success.

## Acceptance Criteria (from the issue)

- [ ] A `reins`-era `pre-push`/`post-checkout`/`post-merge` is detected as
      stale and replaced, restoring the guard.
- [ ] Appending to a hook that ends in an unconditional `exit` fails loudly
      with remediation, and never reports success.
- [ ] Appending to a hook that *can* fall through still works as today.
- [ ] A hook already carrying the current marker remains a no-op.
- [ ] `rcorn kb status` reports a stale/unreachable hook, and reports
      parent-pointer-behind-remote drift (carried over from #21).
- [ ] Tests cover: stale detection/replacement, unreachable-append refusal,
      safe-append path, idempotent path, and both status reports.

## Approach

New `src/reinicorn/hooks_health.py` owning hook-text classification (one
concern per file), shared by install and status:

- `is_stale_reins_hook(text)` — old `reins` delegation or old marker.
  Checked *before* the current-marker check so repos damaged by the old
  unreachable-append bug (stale hook + dead marker) are repaired too.
- `can_fall_through(text)` — last effective (non-blank, non-comment,
  top-level) statement is not an unconditional `exit`.
- `marker_reachable(text)` — current marker present with no top-level
  unconditional `exit` before it.
- `hook_issues(hooks_dir)` — per-hook classification for `kb status`.

`hooks_install`: stale → overwrite (`REPLACED`); foreign hook that cannot
fall through → refuse with remediation and return 1; foreign fall-through
hook → append as today, verified reachable before reporting `APPENDED`.

`kb status`: Health section warns on stale/unreachable hooks (next step
`rcorn hooks install`), and computes how far the parent's recorded kb
pointer is behind kb `origin/main` via local refs only (no network,
principle 6): `git -C kb rev-list --count <recorded>..origin/main`.

## Tasks

- [ ] Red: tests for stale replacement (incl. damaged-append shape),
      unreachable-append refusal, safe append, marker no-op, status reports
- [ ] Green: `hooks_health.py`, `hooks_install` branches, `kb status` checks
- [ ] Verify: pytest, ruff, pyright, tests/run-all.sh

## Dependencies

None. Touches `hooks_install.py` and `status.py`; no overlap with open PRs
apart from trivial proximity to PR #30's `status.py` import block.
