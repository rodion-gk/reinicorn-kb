---
type: retro
title: 'Retro: feat/enforce-review-lane'
slug: feat-enforce-review-lane
lifecycle: active
status: draft
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat/enforce-review-lane
---

# Retro: feat/enforce-review-lane

## What Went Well

- Resolving references against a git-emitted path set (`linter/spec_refs.py`,
  f543f1c) made three acceptance criteria fall out by construction: `..` and
  absolute paths, untracked files, and "approved `specs/x.md` wins over a
  same-named draft" needed tests but no dedicated code. One resolver shared by
  `draft_refs.py` and `spec_gate.py` (`resolve_ref`, `is_spec_path`,
  `unapproved_reason`) means the lint and the push gate cannot disagree.
- Dogfooding caught the original #23 violation on real data: early revisions
  of PR #27 were red on their own lint because the
  `test-lint-and-runner-coverage` plan built on the draft
  `test-coverage-reports` spec, including the second-order miss via the
  drafts fallback. It cleared only when reinicorn-kb#4 merged on 07-27.
- CodeRabbit's staged-but-uncommitted finding (thread 3699952157) was treated
  as a symptom: root-causing it to "the gate consults index/worktree instead
  of the pushed tree" exposed four holes (staged spec, unpinned kb commit,
  non-checked-out branch, uncommitted `Status:` edit) plus the identical
  `HEAD:kb` bug in `_ensure_kb_pushed`, all closed in eaef8c0 with one
  regression test each. A point fix would have left three bypasses open.
- The fail-open / fail-closed asymmetry between `ensure_plan_spec_approved`
  and `_ensure_kb_pushed` is argued in the commit message and the
  `spec_gate.py` docstring, and the fail-open path is required to be loud
  (branch, plan, exception, "gate did not run") — a direct lesson from the
  silent `reins`-era hooks in #24. The next reader cannot "fix" it by
  accident.
- The plan's Approach section doubled as a decision log for findings that
  postdated spec approval (block on unresolved/ambiguous; dedupe by resolved
  path), so plan, PR body, and code tell one story. 46 pre-push tests and 36
  rule tests; 898 passed at 88.75% coverage (floor 87%).

## What Could Be Improved

- The gitlink anchoring that consumed the final review round (`kb_gitlink`,
  `tracked_paths_at` on `<branch>:kb`, per-branch pointer verification in
  `_ensure_kb_pushed`) landed 08-03 in eaef8c0 and was deleted 08-15 by #48.
  The `remove-the-kb-submodule` spec was created 07-26 — the day this PR
  opened — and 798178f (08-02) already cites it. An approved sibling spec
  was removing the very primitive this one was building deeper on, and nobody
  checked the in-flight spec set before choosing the anchor.
- Generator/checker drift: the gate shipped without a "born-passing" check on
  `rcorn plan create`. The PR updated the kb template and `plan.py`'s
  no-template fallback but missed `kb_seed.py`'s seeded template (no `Spec:`
  field on PR head); together with #30 missing the template, every
  `plan create` was blocked at push (#41, fixed by #42 four days after merge).
- Spec §6 ("next-step hints carry the gate": `rcorn review start` /
  `rcorn kb status` say "wait for approval") was never in the plan's Tasks,
  never shipped, and is absent from main today — yet the approved spec still
  lists it as in scope. The spec called it "the first thing to cut", but the
  cut was recorded nowhere.
- Three force-pushes (07-27, 08-02 x2) dismissed rodion-gk's approval
  (07-30, at 33f22d2) and left CodeRabbit's "Addressed in commit ef07a14"
  markers pointing at SHAs that no longer existed. Thread 3655291078 carried a
  stale check mark while the unguarded `read_text()` was still present and
  needed a manual correction (e51025b). Five of the seven merged commits are
  dated after the only human approval; nobody re-approved.
- Plan hygiene: 18 days after merge the plan is still `active` /
  `in-progress` with every AC and Task checkbox unticked, and
  `rcorn plan complete` was never run. Same pattern as the
  `feat-mattpocock-adapter` retro.

## Lessons Learned

- Before anchoring a design on a repo primitive (gitlink, submodule pointer,
  pinned SHA), grep approved and in-review specs for that primitive. Two
  approved specs pulling one primitive in opposite directions is a failure the
  review lane itself cannot catch; only the planner can, at plan time.
- A new gate must be tested against the tool's own generators, not just
  hand-written fixtures: every `rcorn <type> create` output should pass the
  gate unedited except for the one field the author is expected to fill. #42
  added that test for plans; the rule generalises to every future lint/gate.
- When a spec section is cut during execution, amend the spec or record the
  cut in the plan in the same PR. An approved spec that over-promises is
  worse than a smaller approved spec.
- A force-push invalidates both human approvals and bot "addressed" markers.
  Re-verify each open thread against the new head before replying, and
  re-request review when commits land after the approval.
- 46dc1e8: rebasing onto #29 put `spec_refs.py` under
  `test_git_error_surface`, a rule that did not exist on this branch's base,
  so neither branch's CI could fail. Long-lived branches need a rebase plus
  full suite before merge, not just a green check on the pre-rebase head.

## Action Items

- Run `rcorn plan complete` for `feat/enforce-review-lane`: tick the ACs and
  Tasks that shipped, note that spec §6 was cut, archive the plan.
- Decide spec §6's fate: implement the `rcorn review start` /
  `rcorn kb status` wording (#23 direction 5) as a small follow-up, or amend
  `specs/enforce-the-review-lane-draft-refs-resolution-declared-plan.md` to
  drop it.
- Linting archived plans is still an open non-goal: `draft_refs.py` on main
  globs `exec-plans/active/` only, so `rcorn plan complete` removes the
  evidence from the lint surface. File as tech-debt with the migration cost
  the spec flagged.
- Carry the gate items from the
  `post-submodule-removal-follow-ups-from-the-branch-review-1-u` idea into a
  plan: untested pre-push fail-open without `origin/main`, untested
  spec-gate empty-clone guard, redundant `.git` check in `spec_gate.py`.
- The `review-lane-solo-maintainer-self-review-gate` draft spec should say
  what a force-push after approval requires (re-request or explicit
  self-attest), since that exact sequence happened on this PR.
