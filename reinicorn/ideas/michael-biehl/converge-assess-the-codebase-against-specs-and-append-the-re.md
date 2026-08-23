---
type: idea
title: 'Converge: assess the codebase against specs and append the remaining work — from SpecKit'
slug: converge-assess-the-codebase-against-specs-and-append-the-re
lifecycle: active
status: new
created: 2026-07-30
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Converge: assess the codebase against specs and append the remaining work — from SpecKit
## Description

SpecKit ships `/speckit.converge`: assess the codebase against the specs and
append the remaining work to the task list. It is a drift-repair loop — the
answer to "the code and the spec have diverged, now what."

Reinicorn has no equivalent. The workflow assumes you enter through a spec and
walk forward: idea → spec → review → plan → execute → retro. There is no move
for arriving at an existing repo whose code has drifted from its documented
intent, and no move for a branch where implementation wandered away from the
plan mid-flight.

Of the four tools compared on 2026-07-30 (Agent OS, BMAD, SpecKit, govctl),
SpecKit is the only one with a real answer here.

## Why this matters

Two situations, both common:

1. **Brownfield adoption.** Someone runs `rcorn init` on a repo with years of
   history. The kb starts empty and every existing behaviour is undocumented.
   Today the only path is writing specs by hand for code that already works.
2. **Mid-flight drift.** A plan says one thing, the diff says another. Retros
   catch this after the fact; nothing catches it while the branch is open.

Both are the same operation: compare documented intent against actual code and
emit the delta as work.

## Relationship to the analyze/check ideas

SpecKit also has `/speckit.analyze` (cross-artifact consistency) and
`/speckit.checklist`; govctl has `govctl check`. The distinction worth holding
onto:

- **analyze** — do spec, plan, and progress agree *with each other*?
- **converge** — does the spec agree *with the code*?

The first is a document-consistency problem and could plausibly be a lint
rule. The second requires reading the codebase and judging, so it is genuinely
agent work. Both were flagged in
[[2026-05-04-feature-mood-board-from-agent-os-bmad-speckit]]; converge is the
one with no Reinicorn analogue at all.

## Sketch

- A skill rather than a pure CLI command — it needs to read code and judge, not
  just validate structure.
- Input: the specs in scope for this repo, or this branch's plan. Output:
  appended tasks on the active exec-plan, or a fresh idea/spec draft when the
  gap is large enough to need a human decision first.
- Must not silently rewrite specs to match the code — drift can mean the code
  is wrong. Emit the delta and let a human pick a direction.
- Pairs naturally with `rcorn plan status` and with the retro step.

## Notes

Surfaced during the 2026-07-30 four-tool comparison.
