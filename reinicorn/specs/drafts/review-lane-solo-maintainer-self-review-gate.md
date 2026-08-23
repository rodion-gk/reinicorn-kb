---
type: spec
title: 'review lane: solo-maintainer self-review gate'
slug: review-lane-solo-maintainer-self-review-gate
lifecycle: active
status: draft
created: 2026-07-24
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# review lane: solo-maintainer self-review gate

Closes #2.

## Problem

`rcorn review start` pushes the candidate ref and opens the PR as the invoking
user, so the maintainer is always the PR author. GitHub forbids approving your
own PR, so on a repo with a single collaborator `reviewDecision` can never
become `APPROVED`. In `cmd_review_merge` the approval check
(`src/reinicorn/commands/review.py`, ~line 288) then dead-ends: every merge
needs `--force`, and `review setup` never warns that the required-review gate
it installs is structurally unsatisfiable on such a repo.

`--force` is a blunt escape — it also bypasses the divergence guard and the
slug-collision paths — and it stamps no record that the merge was self-reviewed
rather than genuinely approved.

## Design Goals

- A solo maintainer can land a review without `--force` and without a
  misleading dead-end, while keeping the gesture explicit (not a silent
  auto-merge).
- The finalized doc records that it was self-reviewed, distinct from a doc a
  second party approved.
- Non-solo repos are completely unaffected: a real unmet approval still blocks.
- `review setup` tells a solo operator up front that the gate will require
  self-review, so the dead-end is never a surprise.
- Agent (non-TTY) flows stay deterministic — no hang waiting on a prompt.

## Design

### Solo-repo detection (`src/reinicorn/github.py`)

Add `gh_repo_is_solo(repo: str) -> bool | None`: list push-capable
collaborators via `gh api repos/{repo}/collaborators --paginate` and return
`True` when at most one collaborator has push/admin/maintain permission. Return
`None` when the listing fails or the token can't inspect collaborators (unknown
— never assume solo). Boundary-validate the JSON: each entry's
`permissions.push` drives the count; anything unparseable → `None`.

### Merge path (`cmd_review_merge`)

When `decision != APPROVED and not force`, before printing the existing
"not approved" / "no required-review rule" errors, consult
`gh_repo_is_solo(gh_repo)`:

- If solo is `True`, this is the structurally-unsatisfiable case. Offer a
  self-review path instead of the dead-end:
  - Print that the repo has no second reviewer, so approval must be a
    self-review.
  - If `console.confirm(...)` returns `True` (interactive TTY, operator
    assents), proceed with the merge; otherwise print the `--force` next step
    and return 1. `--force` continues to work as the non-interactive escape.
  - When the merge proceeds via the solo path (confirm or `--force` on a solo
    repo), set `approved_by = f"{author} (self-reviewed)"` instead of deriving
    it from `latestReviews`.
- If solo is `False` or `None`, keep today's exact behavior (the two existing
  error branches).

`author` is the authenticated GitHub login (`gh api user --jq .login`), falling
back to git `user.name` when gh can't answer.

### Confirmation helper (`src/reinicorn/console.py`)

Add `confirm(prompt: str) -> bool`: when stdout/stdin is not a TTY, return
`False` (agent flows must pass `--force` explicitly — never block). On a TTY,
read a y/N answer, defaulting to no. This is the shared helper other commands
can reuse (golden principle 2).

### Setup warning (`cmd_review_setup`)

After the ruleset is applied (or detected already-installed), if
`gh_repo_is_solo(gh_repo)` is `True`, warn that the required-review gate is
structurally unsatisfiable on a solo repo and that merges will need
`rcorn review merge <slug>` self-review (or `--force`). Unknown (`None`) →
no warning (don't cry wolf).

## Non-Goals

- Bot / GitHub App identity so the human can genuinely approve — larger surface,
  deferred.
- Changing how `--force` interacts with the divergence or slug-collision
  guards.
- Auto-merging on a solo repo with no explicit gesture (confirm or `--force`).
