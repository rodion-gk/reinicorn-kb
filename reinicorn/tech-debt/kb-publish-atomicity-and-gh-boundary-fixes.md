---
type: debt
title: kb publish atomicity and gh boundary fixes
slug: kb-publish-atomicity-and-gh-boundary-fixes
lifecycle: active
status: draft
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: kb-sync, review-lane
severity: high
remediation: planned
---

# kb publish atomicity and gh boundary fixes

## Impact

Three verified correctness problems from the 2026-07-27 quality review:

1. **The kb publish path can abandon the kb mid-merge and misreport why.** `kb.py:105-142`: `commit_kb` returns `False` for both "nothing to commit" and "commit failed", so a failed commit silently proceeds to push stale state; `ensure_kb_on_main` swallows a failed checkout, so the commit can land on the wrong branch or a detached HEAD. On push rejection, `push_main_with_retry` runs `pull --no-rebase` with `check=False` — a conflicting pull leaves the kb submodule with a `MERGE_HEAD` and conflict markers, while `commands/publish.py:39-45` discards the git stderr and asserts "resolve conflicts" whether or not conflicts are the cause (auth failure and offline remote fail identically). The retry branch is live in normal use, not an edge case. `cli.py` and docs also advertise publish as "rebase + push" while the implementation merges.
2. **`cmd_review_merge` can traceback on real GitHub data.** `github.py:133-146` returns the raw `gh pr view --json` dict; `commands/review.py:325-331` does `pr["number"]`, `pr["url"]`, and `r["author"]["login"]` over `latestReviews`. GitHub returns `"author": null` for deleted accounts, and the resulting `TypeError`/`KeyError` is not caught by `_surfacing_errors` (contract: `RuntimeError`/`CalledProcessError` only) — a raw traceback in the highest-stakes command of the review lane.
3. **Two `.gitmodules` parsers with divergent semantics.** `init.py:55-68` matches any submodule with `path = kb`; `kb.py:17-34` requires the section to be named `"kb"`. A repo where they disagree gets init'd against a kb that every other command refuses to see.

## Remediation Plan

1. `commit_kb` returns a three-state result (or raises with git stderr); `ensure_kb_on_main` verifies the checkout actually landed on main before committing; after a failed pull, detect `MERGE_HEAD` (`git rev-parse -q --verify MERGE_HEAD`) and either abort the merge (restoring pre-publish state) or print the conflicted files with exact resolution commands; include `push.stderr` in `cmd_publish`'s error and reserve the conflict wording for confirmed conflicts (the `sync.py:33` `--diff-filter=U` pattern). Align the "rebase" wording in `cli.py` and docs with the actual merge behavior, or switch the implementation to rebase — pick one.
2. Parse `gh pr view` output once at the boundary into a typed `PrInfo` record (`number: int`, `state`, `url`, `review_decision`, `approver_logins: list[str]`) with a guarded `json.loads`, following the existing `gh_repo_is_solo` / `GatedDraft` patterns. `commands/review.py` never touches the raw dict.
3. Delete `_has_kb_submodule_path` from `init.py`; base `_init_path` on `get_kb_dir(cwd) is not None`.
4. Related smaller items in the same seams: tty-guard the `input()` prompt in `update.py:120-127` (uncaught `EOFError` in agent/CI contexts, mid-update; follow `kb_manage.py`'s isatty + `--force` pattern) and remove `update.py`'s `_get_package_version` identity wrapper and `_get_repo_root`'s silent `Path("")` fallback. Note: fix the vacuous update test first (see `test-suite-health-vacuous-update-test-host-isolation-coverag.md`) so the refactor lands on a test that actually tests the skip logic.
