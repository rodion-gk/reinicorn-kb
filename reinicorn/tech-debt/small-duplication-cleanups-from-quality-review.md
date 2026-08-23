---
type: debt
title: Small duplication cleanups from quality review
slug: small-duplication-cleanups-from-quality-review
lifecycle: active
status: draft
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: maintainability
severity: low
remediation: planned
---

# Small duplication cleanups from quality review

## Impact

Batchable low-risk cleanups from the 2026-07-27 quality review. Individually minor; together roughly 150 lines of duplicated or dead code where a change made in one copy will drift from the others. None are user-visible today. Items that overlap the registry-driven doc-types spec (`specs/drafts/registry-driven-doc-types.md`) are excluded — this doc is only the type-agnostic leftovers.

## Remediation Plan

One or two PRs, behavior-preserving, existing tests green throughout:

1. **`hooks_install.py:157-271`** — collapse `_merge_claude_settings`/`_merge_cursor_settings`/`_merge_copilot_settings` (three ~35-line copies differing in hooks key, dedup field, and version seed) into one parameterized `_merge_hook_config`; the per-editor variance is already captured by the `_*_entry` builders. Deletes ~70 lines.
2. **`linter/runner.py:36-117`** — unify the built-in-rules loop and the external-`.sh` loop (duplicated severity resolution, tallies, PASS/FAIL printing) by normalizing both sources into `(name, thunk)` pairs run by one engine. Replace the `try: rule_cls(**kwargs) except TypeError: rule_cls()` constructor probe (which would mask a real `TypeError` inside a rule) with explicit config passing. Roughly halves the file.
3. **`submodule.py:32,40,106`** — the file-transport tuple is rebuilt inline three times with a URL check that has drifted from `git.file_transport_args`; add `file_transport_args_for_url(url)` next to the canonical helper and use it at all three sites.
4. **Dead/duplicate code:** delete production-dead `review.resolve_draft` (port its tests to `resolve_drafts`); delete the `kb.branch_dir_name` identity alias for `git.sanitize_branch` (pick one canonical name); delete the unused `name()` method from the linter rule ABC and its four implementations (the `BUILTIN_RULES` key is authoritative); extract `git_author()` into `git.py` replacing the three inline "git config user.name or unknown" copies (`doc_create.py:16`, `plan.py:62`, `idea.py:26`).
5. **`feedback.py:121-128`** — route the `gh issue create` call through `github.run_gh` (add `gh_issue_create`, `interactive=True`) instead of raw `subprocess.run`, then drop the "can migrate later" hedge from `github.py`'s docstring.
6. **`commands/review.py:354-383`** — append a recovery next-step to the surfaced push failure in `cmd_review_cancel`/`cmd_review_start` (rerun converges; the message should say so).
7. **`commands/doc_show.py:159-189`** (added 2026-08-18, from PR #51 review) — `cmd_branch_show`'s retro path repeats `_branch_doc_show`'s frame (repo-dir resolution, branch defaulting, missing-branch error, `_print_doc`, return). Inherited shape from the old `cmd_retro_show`; unify by passing a fallback target resolver and extra glob patterns into `_branch_doc_show` once a second rider type exists (parameterizing for one caller today adds indirection for no drift risk the tests don't already cover).
