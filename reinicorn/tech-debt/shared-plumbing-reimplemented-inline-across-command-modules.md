---
type: debt
title: shared plumbing reimplemented inline across command modules
slug: shared-plumbing-reimplemented-inline-across-command-modules
lifecycle: active
status: draft
created: 2026-07-09
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: cli
severity: medium
remediation: planned
---

# shared plumbing reimplemented inline across command modules

## Impact

The same few lines of logic are re-inlined instead of calling the module that
owns the concept, so behavior drifts silently between copies (found in the
2026-07-09 repo-wide review sweep):

- The "iterate kb repo-scope dirs, skip `.`/`_`" loop is copy-pasted at 11
  sites (`kb.py`, `status.py` ×3, `kb_manage.py`, `post_merge.py`, linter
  rules); skip rules already drift (`doc_create.py` also skips `generated`,
  the others don't).
- `feedback.py:_open_issue` invokes `gh` via raw `subprocess` + `shutil.which`
  instead of `github.gh_available()`/`run_gh()` — the only gh call bypassing
  the centralized error handling.
- `git config user.name` fetch inlined 3× (`doc_create._get_author`,
  `plan.py`, `idea.py`).
- Slugify regex duplicated (`doc_create._slugify`, `idea.py`).
- `submodule.py` inlines a URL-keyed `protocol.file.allow` check 3× while
  `git.file_transport_args` owns the cwd-keyed variant (and also checks
  `file://`, which submodule.py misses).
- `pre_push.py` reads `.reins-mode` by hand instead of `mode.can_publish()`.
- `plan complete` mutates `**Status:**` with an ad-hoc regex instead of
  `docmeta.set_field`; `review.py:_doc_title` and `doc_show._title_and_status`
  hand-roll provenance-block reads that `docmeta.get_field` owns.
- `hooks_install.py` has three ~35-line settings-merge functions differing
  only in hooks key + dedup field.
- The `root = repo_root(); if root is None: return 1; kb_dir =
  require_kb_dir(root)` preamble repeats ~13×.
- `cmd_review_merge` interleaves guards, gh branching, and pr_url/approved_by
  bookkeeping at 3–4 nesting levels.

## Remediation Plan

- Promote `iter_scope_dirs(kb_dir)` (and `iter_active_plan_dirs`) to `kb.py`;
  point all 11 sites at it, unifying the skip rules.
- Migrate `feedback.py` to `github.gh_available()` + `run_gh()` (github.py's
  docstring already anticipates this).
- Add `git.git_user_name(default="unknown")`; use in the 3 sites.
- Move `_slugify` to a shared home; import from `idea.py`.
- Add `git.file_transport_args_for_url(url)`; replace submodule.py's 3 inline
  checks.
- `pre_push.py` → `mode.can_publish()`.
- `plan complete` / title-status readers → `docmeta.set_field`/`get_field` +
  a shared `doc_title()` helper.
- Collapse `hooks_install.py` merges into one parameterized
  `_merge_hook_settings()`.
- Optional: `kb.require_root_and_kb()` for the repeated preamble; extract
  `_ensure_pr_merged()` from `cmd_review_merge` (mirrors the `_finalize_tree`
  extraction).
