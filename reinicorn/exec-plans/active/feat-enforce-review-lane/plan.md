---
type: plan
title: 'Execution Plan: feat/enforce-review-lane'
slug: feat-enforce-review-lane
lifecycle: active
status: in-progress
created: 2026-07-26
author: Michael Biehl
branch: feat/enforce-review-lane
ticket: '#23'
spec: specs/enforce-the-review-lane-draft-refs-resolution-declared-plan.md
---

# Execution Plan: feat/enforce-review-lane

## Goal

Make the review lane an enforced gate rather than a documented norm. Fix the
`kb/draft-refs` blind spot, give plans a declared `**Spec:**` field, promote the
rule to error severity, and block a push whose plan builds on an unapproved spec.

## Acceptance Criteria

- [ ] `specs/x.md`, `reinicorn/specs/drafts/x.md` and `kb/reinicorn/specs/x.md`
      all resolve to the same doc and are all diagnosed when it is unapproved.
- [ ] A plan citing `specs/<slug>.md` when only `specs/drafts/<slug>.md` is
      tracked is reported — the second-order miss from the incident.
- [ ] A reference matching no tracked kb path is reported as unresolved.
- [ ] A reference whose candidate forms hit two distinct tracked paths is
      reported as ambiguous, naming both.
- [ ] A doc present on disk but untracked does not satisfy a reference.
- [ ] `..` and absolute paths resolve to nothing; no filesystem access for them.
- [ ] An approved `specs/x.md` still wins when a same-named draft is tracked.
- [ ] A plan with no `**Spec:**`, or the template placeholder, is reported.
- [ ] `**Spec:** N/A` is exempt from the field check but prose is still scanned.
- [ ] A reference inside a fenced code block is still ignored.
- [ ] A draft reference declared in `**Spec:**` reports once, not twice.
- [ ] `rcorn kb lint` exits non-zero on a draft reference; Lint Kb fails.
- [ ] Active plans carry a backfilled `**Spec:**`; `main` green after the flip.
- [ ] `git push` is blocked when the plan's spec is draft/in-review/unresolved/
      ambiguous, naming the doc; `--no-verify` bypasses.
- [ ] `git push` is not blocked for `N/A`, no plan, approved spec, or
      incognito/disabled mode.
- [ ] An exception in the gate warns (branch, plan, exception, "gate did not
      run") and allows the push; `_ensure_kb_pushed` still fails closed.

## Approach

Per the approved spec. Key decisions:

- **Resolution against git, not the filesystem.** A shared helper builds the kb's
  tracked-path set via `run_git("ls-files", "-z", cwd=kb_dir)` and resolves by
  exact lookup. Containment and untracked-file handling fall out for free.
- **One shared resolver.** `draft_refs.py` and `pre_push.py` import the same
  helper so the lint rule and the push gate can never disagree.
- **Two CodeRabbit findings from review**, decided here since they postdate spec
  approval:
  - *Unresolved/ambiguous at the push gate*: the gate **blocks**. Fail-open
    applies to internal exceptions, not to determinable policy violations — a
    reference we cannot resolve is an error state, not a pass.
  - *Double-reporting*: diagnostics are deduplicated by resolved path per plan,
    so a draft declared in `**Spec:**` and mentioned in prose reports once.

## Tasks

- [ ] Add `FIELD_SPEC` to `docmeta.py`
- [ ] Write `linter/spec_refs.py` shared resolver (tracked set, candidate forms,
      drafts fallback, unresolved/ambiguous outcomes)
- [ ] Rewrite `linter/rules/draft_refs.py` on the resolver + dedup
- [ ] Add the `**Spec:**` field check to `linter/rules/plan_structure.py`
- [ ] Add `**Spec:**` to `exec-plans/_template/plan.md`
- [ ] Backfill `**Spec:**` on all active plans
- [ ] Flip `kb/draft-refs` to `severity: error` in `linters/.lint-config.json`
- [ ] Add `_ensure_plan_spec_approved` to `commands/internal/pre_push.py`
- [ ] Tests for every acceptance criterion above
- [ ] Full suite + coverage floor green

## Dependencies

Spec `specs/enforce-the-review-lane-draft-refs-resolution-declared-plan.md`
(approved, reinicorn-kb#5). Independent of the markdownlint work in #26 /
reinicorn-kb#6, which is still in review.
