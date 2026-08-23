---
type: debt
title: CLI hints printed without checking their preconditions
slug: cli-hints-printed-without-checking-their-preconditions
lifecycle: active
status: draft
created: 2026-07-09
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: cli
severity: low
remediation: planned
---

# CLI hints printed without checking their preconditions

## Impact

Next-step hints and guard messages that don't verify they apply send users to
commands that refuse to run, or misstate the situation (found in the
2026-07-09 repo-wide review sweep; the two worst cases — sync conflict
misdiagnosis and the review-merge approval dead end — were fixed on the
doc-review-lane branch):

- `plan.py:41` guards the default branch by hard-coding `main`/`master` — the
  guard silently misses repos whose default is `develop`/`trunk`, and the
  message overclaims on them.
- Gated doc creation prints contradictory next steps: `reins review start`
  AND `reins kb publish` for the same draft (`doc_create.py:164-167`); the
  publish hint should be non-gated only.
- "No execution plan" → `next: reins plan create` is printed on branches
  where `plan create` immediately refuses (main/master/detached) —
  `status.py:83`, `status.py:147`, `plan.py:129`.
- "gh unavailable" (`review.py` ×4) fires on `not _gh_ready()`, conflating
  gh-not-installed with gh-not-authenticated; the two checks already exist
  separately, so the message could name the actual cause and suggest
  `gh auth login`.

## Remediation Plan

- Resolve the real default branch (`git symbolic-ref
  refs/remotes/origin/HEAD`, cached) for the plan-create guard.
- Gate the `kb publish` next-step on `not REGISTRY[doc_type].gated`.
- Gate `plan create` suggestions on the same branch check `plan create`
  itself uses.
- Split the "gh unavailable" wording by `gh_available()` vs
  `gh_authenticated()`.
