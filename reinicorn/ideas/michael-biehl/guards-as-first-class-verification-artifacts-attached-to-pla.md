---
type: idea
title: Guards as first-class verification artifacts attached to plans — from govctl
slug: guards-as-first-class-verification-artifacts-attached-to-pla
lifecycle: active
status: new
created: 2026-07-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Guards as first-class verification artifacts attached to plans — from govctl
## Description

govctl models **guards** as their own artifact type: `GUARD-*.toml` files
sitting alongside RFCs, ADRs, and work items in `gov/`. A guard is an
executable completion gate — a declared, reviewable statement of what must be
true before a phase advances. `govctl check` runs them, and a work item cannot
reach the next phase until its guards pass.

Reinicorn already has verification, but it is scattered: git hooks, editor
hooks, `linters/rules/`, the verification-before-completion skill, and whatever
tests a plan happens to mention in prose. None of it is attached to a specific
plan or task, and none of it is declared as data.

## Why this is the interesting part of govctl

The four-tool comparison on 2026-07-30 put Reinicorn ahead on enforcement:
SpecKit's checks are slash commands the agent may simply skip, while
Reinicorn's hooks and linters fire outside the model's control. Guards are the
one place govctl goes further than that.

The difference is that a guard is **per-work-item data**, not global
infrastructure. Reinicorn's linters enforce that every kb doc has the right
shape — the same rules for everyone, forever. A guard says "*this* plan is not
done until *this* specific thing is true," and that statement is written down,
diffable, and reviewable before implementation starts rather than remembered
during it.

This is also the natural home for the acceptance criteria that currently live
as prose in exec-plans, where nothing checks that they were met.

## Sketch

- Guards attach to an exec-plan, and probably to individual plan tasks.
- Declared as structured data, not free prose — the whole point is that
  something can run them. Frontmatter or a sibling file in the plan directory;
  the exact shape can follow whatever
  [[registry-driven-doc-types]] settles on.
- Kinds worth supporting: a shell command that must exit zero, a test selector
  that must pass, a coverage floor, a file-must-exist assertion, and an
  explicit human-judgement guard for the things a machine cannot check.
- `rcorn plan complete` refuses to archive a plan with failing guards, the same
  way the review lane refuses to merge an unreviewed spec. That is the gate
  that makes it real rather than decorative.
- Guards are written when the plan is written, so they get reviewed as part of
  the plan rather than invented at the end to justify what was built.

## Open questions

- Does this overlap the verification-before-completion skill enough to replace
  it, or does the skill become the thing that *authors* guards?
- Should a spec carry guards too, or only plans? govctl puts them on work
  items, which maps to exec-plans here.
- Human-judgement guards need a recorded answer, which starts to look like the
  review lane. Worth checking they do not become two mechanisms for one job.

## Notes

Surfaced during the 2026-07-30 four-tool comparison — see
[[2026-05-04-feature-mood-board-from-agent-os-bmad-speckit]] for the full
landscape. govctl (github.com/govctl-org/govctl) was new to that comparison and
had not previously been considered.
