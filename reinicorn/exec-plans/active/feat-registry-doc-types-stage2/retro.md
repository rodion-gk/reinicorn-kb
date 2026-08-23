---
type: retro
title: 'Retro: feat-registry-doc-types-stage2'
slug: feat-registry-doc-types-stage2
lifecycle: active
status: draft
created: 2026-08-21
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-registry-doc-types-stage2
---

# Retro: feat-registry-doc-types-stage2

Stage 2 of the `registry-driven-doc-types` spec. PR #51 targeted the
`feat-registry-doc-types` integration branch (not `main`); that branch
merged to `main` as PR #57 on 2026-08-20 together with stage 1 (#50) and
the skill-set adapters (#55). Spec stages 3-5 were not part of it.

## What Went Well

- The whole CLI surface collapsed into two generators: `_add_doc_type_groups`
  (e9eb841) and `_doc_dispatch_rows` (84b2f55) replaced five hand-written
  parser blocks and 16 `_DISPATCH` rows, net -40 lines in `cli.py` with no
  command-behavior change (7 files, +248/-163 overall). The four
  implementation commits landed in 20 minutes on 2026-08-16, straight after
  stage 1 merged, because stage 1's `Addressing`/`TitleSource`/`CreateMode`
  enums were exactly the inputs the generators needed.
- The phantom-type test from the spec's testing section became the
  executable proof: `test_phantom_create_end_to_end` parses `phantom create`,
  runs the generated row, and finds the file on disk with no change outside
  `REGISTRY`. The stage-1 helpers (`kb_repo`, `_create_env`) were reused as
  the plan instructed instead of duplicated.
- The plan's own code had a stringly-typed comparison
  (`create_verb == "add"`) that would have violated golden principle 15;
  the implementer caught it mid-task and shipped `create_mode is
  CreateMode.APPEND` (0faeb0e), disclosing the deviation in the PR body.
- Review response was verify-first and sorted: of CodeRabbit's 1 actionable
  + 7 nitpicks, 3 applied (registry-order test, BRANCH phantom, returning
  the subparser mapping instead of reflecting over private argparse
  internals), 3 declined with spec citations, 1 deferred into an existing
  tech-debt doc
  (`small-duplication-cleanups-from-quality-review.md` item 7). All in one
  commit (8ae2591); gate went 1061 -> 1063 tests, coverage 88.94 -> 88.97%.
- CI green on first push (pytest, kb lints, structural tests, CodeRabbit
  status), and every PR-body normalization claim (help wording, `plan
  --help` verb order) was checked by the final reviewer diffing live
  `--help` output against base.

## What Could Be Improved

- The loop made `REGISTRY` row order load-bearing (CLI group order and
  `by_dir` precedence for the shared `exec-plans` dir), which the plan did
  not anticipate: 0faeb0e had to reorder the rows and add a comment, and
  CodeRabbit then had to ask for the comment to become a test
  (`test_plan_precedes_retro_for_shared_dir`). An invariant introduced by a
  refactor should ship with its test in the same commit.
- The plan's phantom test covered only `Addressing.SLUG`; the BRANCH path
  was exercised solely through the real plan/retro rows until review asked
  for `PHANTOM_BRANCH` ("ghost"). One phantom per enum branch should have
  been in the plan from the start.
- Michael's human review comment on `cli.py:26` ("this function has gotten
  a bit too long", 2026-08-18) was never replied to or addressed.
  `_add_doc_type_groups` is still ~70 lines nested inside a ~200-line
  `_build_parser` on `main`. Both review threads on #51 are still
  unresolved (outdated, not resolved) because the inline reply was blocked
  by a pending human review and the responses went into PR comments
  instead.
- Plan hygiene: `plan.md` is still `status: planning` with 0/28 checkboxes
  ticked, in `exec-plans/active/`, although the PR merged on 2026-08-19 and
  the feature reached `main` on 2026-08-20. Stage 1's plan, by contrast, is
  in `completed/`.
- CodeRabbit auto-review is disabled for PRs whose base is not the default
  branch, so the review needed a manual `@coderabbitai review` two days
  after the PR opened, and the plan's 1-review/hour quota was exhausted by
  it. Stage PRs into integration branches need this trigger built into the
  PR checklist.

## Lessons Learned

- Generating a surface from an ordered mapping turns the mapping's order
  into an API. Whenever a loop replaces hand-written blocks, ask what the
  iteration order now controls and pin it.
- A spec-level "executable design goal" test needs one synthetic row per
  code path in the generator, not one row total; otherwise the real rows
  are still the only coverage of the paths they happen to use.
- `_DISPATCH` is built at import time while `_build_parser` reads
  `REGISTRY` at call time. Inert today, but CodeRabbit's merge-risk note
  is right that a config-loaded registry would make a command parse and
  then fail to dispatch. The two table constructions must share a lifetime
  before the config-file stage.
- Declining a reviewer's "make it a registry field" proposal is legitimate
  when the spec names the coupling as a non-goal (retro-rides-with-plan)
  and the field would be config-for-one, but the decline is stronger when
  the identity check is backed by an enforcing test
  (`test_no_doc_type_key_comparisons`), which is what made the answer
  stick.
- Plan code is not exempt from the golden principles: the plan's literal
  snippet contained the banned pattern. Reviewing the plan against the
  principles before execution is cheaper than catching it in the diff.

## Action Items

- `rcorn plan complete` for this branch after ticking the plan's task and
  AC checkboxes (all 28 are unticked; every AC was met per the PR body).
- Record the import-time `_DISPATCH` vs call-time parser asymmetry as a
  kb tech-debt item (today it lives only in the #51 body) and resolve it
  when the config-file registry loading spec is written: rebuild or defer
  the dispatch table with the parser.
- Answer and close the open `cli.py:26` review thread: either split
  `_add_doc_type_groups` into per-`TitleSource`/per-`Addressing` helpers
  or hoist it to module level with a short verdict that the length is
  accepted. Resolve both #51 threads either way.
- Spec stages 3-5 (gate generalization + literal sweep, seeding + linting
  from `readme_label`/`required_sections`, shipped-docs/test genericization)
  did not ship before the integration branch merged to `main`;
  `kb_seed.py` on `main` still has zero `REGISTRY` references. Each needs
  its own plan and branch off `main` now. The "Show a idea doc" article
  artifact and `plan --help` verb order ride with stage 5.
- Tech-debt item 7 in `small-duplication-cleanups-from-quality-review.md`
  (retro path duplicating `_branch_doc_show`'s frame in `cmd_branch_show`)
  should be picked up the next time `doc_show.py` is touched or a second
  rider type appears.
