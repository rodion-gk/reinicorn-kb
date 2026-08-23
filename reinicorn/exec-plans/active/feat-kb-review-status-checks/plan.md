---
type: plan
title: 'Execution Plan: feat-kb-review-status-checks'
slug: feat-kb-review-status-checks
lifecycle: active
status: in-progress
created: 2026-08-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-kb-review-status-checks
ticket: N/A
spec: 'specs/kb-review-prs-get-real-github-status-checks-doc-lint-candida.md'
---

# Execution Plan: feat-kb-review-status-checks

## Goal
Give review-lane PRs on kb repos two real GitHub checks — "Doc lint" and
"Candidate integrity" — installed by `rcorn review setup` and required by
the `reinicorn-doc-review` ruleset, so a candidate that fails lint, drifted
from its draft, or touches more than its one file is red on the PR page
before anyone reads prose. Merge-time guards in `review merge` stay.

## Acceptance Criteria
- [ ] `workflows/reinicorn-doc-review-checks.yml` is bundled; `rcorn review
  setup` installs it next to the cleanup workflow with the same
  `__REINICORN_REPO__` substitution, modified-file refusal, and `--force`
  overwrite (both assets iterated from one tuple; all refusals checked
  before any write, one commit).
- [ ] The workflow runs on `pull_request` with two jobs whose `name:` values
  equal the ruleset contexts exactly: "Doc lint" runs `rcorn kb lint` from
  a Reinicorn@main checkout with the PR head checked out into `kb/`;
  "Candidate integrity" runs `rcorn _review-check "$HEAD_REF"` (env
  indirection, never inlined). Install uses the documented
  `REINICORN_INSTALL_TOKEN || github.token` fallback; no new secrets.
- [ ] `rcorn _review-check <head-ref>`: non-lane refs skip with rc 0; lane
  refs fail (rc 1, agent-readable messages, all failures listed) when the
  diff vs current main is not exactly one added file at the final path,
  the candidate's `status:` is not `in-review`, the candidate body differs
  from `drafts/<slug>.md` on current main, or the draft is gone from main.
- [ ] `_RULESET` carries a `required_status_checks` rule with both contexts
  (`strict_required_status_checks_policy: false`); bypass actors unchanged.
- [ ] `review setup` reconciliation flags an installed ruleset missing the
  checks rule (or a context) as outdated without `--force`, and under
  `--force` merges the rule in while preserving user rules/conditions and
  the existing bypass repair.
- [ ] Tests: `_review-check` end to end against `kb_pair` (pass, drift,
  extra file, wrong status, draft gone, non-lane skip, usage, cli
  dispatch); workflow asset structure/hardening/context-name consistency;
  setup multi-asset install + ruleset rule reconciliation. One manual
  live run of the new workflow on a kb PR after rollout.
- [ ] Gate green: `uv run pytest tests/ -q`, ruff, pyright,
  `bash tests/run-all.sh`.

## Approach
Mirror the cleanup workflow end to end (same template conventions, same
`_review-cleanup` entry pattern, same setup semantics). The integrity
check works inside the Actions checkout of the kb: `git fetch origin main`
then pure `git show`/`git diff` against FETCH_HEAD — no temp clones, no gh.
The draft-vs-candidate comparison is extracted from `candidate_matches_draft`
into a pure text function shared by the CLI guard and CI. GitHub's
`required_status_checks` parameter shape verified against the REST docs
(2022-11-28): `required_status_checks[].context` (required),
`strict_required_status_checks_policy` (required).

## Tasks
- [x] Task 1: Integrity core + `rcorn _review-check` — pure
  `candidate_matches(candidate, draft)` in `review.py`,
  `commands/internal/review_check.py`, `_INTERNAL_COMMANDS` wiring, tests
  in `tests/commands/internal/test_review_check.py`.
- [x] Task 2: Workflow asset `workflows/reinicorn-doc-review-checks.yml`
  + asset tests (structure, head.ref hardening, install source, job names
  match ruleset contexts).
- [x] Task 3: `review setup` — asset tuple loop, `required_status_checks`
  rule in `_RULESET`, reconciliation covers the rule; update/extend the
  setup tests; `review merge` error hint mentions required checks.
- [x] Task 4: Docs — kb doc-review-lane references + README/docs mention
  of the two checks and the rollout step; full gate; PR.
- [ ] Task 5 (post-merge, manual): reinstall rcorn, `rcorn review setup
  --force` on this kb repo, `rcorn review push` the open candidates, and
  confirm both checks attach green on a PR.

## Dependencies
Spec approved (kb PR #17). Touches `commands/review.py` and the setup
tests; no other active branch overlaps on those files.
