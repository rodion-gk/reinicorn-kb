---
type: spec
title: 'Kb review PRs get real GitHub status checks: doc lint + candidate integrity'
slug: kb-review-prs-get-real-github-status-checks-doc-lint-candida
lifecycle: active
status: approved
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/17
---

# Kb review PRs get real GitHub status checks: doc lint + candidate integrity

## Problem

Review-lane PRs on the kb repo carry zero GitHub checks — PR #16's status
rollup is empty. The reviewer's approval is the only signal on the page.
Meanwhile every mechanical guarantee the lane depends on runs somewhere
else: `rcorn kb lint` runs locally and in the *code* repo's CI, and the
candidate-vs-draft divergence guard runs only inside `rcorn review merge`
on the maintainer's machine. A candidate that fails lint, drifted from its
draft on main, or touches more than its one gated doc looks exactly like a
healthy one at the bottom of the PR page.

## Design Goals

- The kb PR page shows real checks: doc lint and review-lane integrity,
  green or red before anyone reads prose.
- Installed and maintained by `rcorn review setup`, exactly like the
  cleanup workflow — no hand-tended YAML in kb repos.
- Red checks block the merge via the `reinicorn-doc-review` ruleset;
  direct `kb publish` pushes to main stay bypassed as today.
- No new secrets: Reinicorn installs from its public repo with the runner
  token fallback already documented in the cleanup workflow.
- The merge-time local guards stay; CI duplicates them earlier, it does
  not replace them.

## Design

### 1. Second workflow asset: `reinicorn-doc-review-checks.yml`

New template in `workflows/`, installed by `rcorn review setup` to the kb
repo's `.github/workflows/` with the same `__REINICORN_REPO__`
substitution, modified-file refusal, and `--force` overwrite semantics as
the cleanup workflow. `review.py`'s single `_WORKFLOW_ASSET` constant
becomes a tuple of (asset, dest) pairs iterated by setup.

### 2. Check "Doc lint"

Job on `pull_request`: checkout Reinicorn source at `main`, `pip install`
it, checkout the PR head *into `kb/` inside that checkout* (the same
shape `lint-kb.yml` uses in reverse), run `rcorn kb lint`. `repo_root()`
resolves to the Reinicorn checkout, so the existing lint entrypoint works
unchanged — no new flags. An in-review candidate at its final path is
lint-legal today (`draft-refs` judges plans, not the candidate itself),
so a green run is meaningful, not vacuous.

### 3. Check "Candidate integrity": `rcorn _review-check <head-ref>`

Internal CI command, same entry pattern as `_review-cleanup` (parses the
`review/` ref, non-lane refs skip with rc 0). On a lane ref it verifies,
against the Actions checkout:

- the PR diff touches exactly one file, at the target's final path;
- the candidate's frontmatter `status:` is `in-review`;
- the candidate body matches `drafts/<slug>.md` on current main
  (`candidate_matches_draft`, reused) — a drifted draft reds the check
  instead of surfacing at merge time;
- the draft still exists on main (a cancelled/landed slug reds rather
  than merging a ghost).

### 4. Ruleset: required status checks

`_RULESET` gains a `required_status_checks` rule listing both contexts
("Doc lint", "Candidate integrity"). `review setup`'s existing
verify/repair reconciliation extends to cover the new rule so pre-existing
installs upgrade under `--force`. Bypass actors are unchanged: pushes to
main bypass, PRs into main do not.

### 5. Rollout

After the implementation merges: `rcorn review setup --force` on this kb
repo, then re-push any open candidates (`rcorn review push`) so the new
checks attach to their PRs.

## Non-Goals

- No checks on direct kb pushes to main (`kb publish` stays instant).
- No CodeRabbit or content review on the kb repo — these checks are
  mechanical only.
- No removal or weakening of the merge-time guards in `review merge`.
- No per-doc-type semantic checks beyond what `rcorn kb lint` already
  encodes.
