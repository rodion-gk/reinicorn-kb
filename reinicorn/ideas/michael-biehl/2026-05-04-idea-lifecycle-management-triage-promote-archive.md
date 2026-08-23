---
type: idea
title: Idea lifecycle management — triage, promote, archive
slug: 2026-05-04-idea-lifecycle-management-triage-promote-archive
lifecycle: active
status: open
created: 2026-05-04
author: Michael Biehl
---

# Idea lifecycle management — triage, promote, archive

## Description

We have a solid capture path (`reins idea` / `/capture-idea`) but no
lifecycle. Ideas accumulate in `kb/{repo}/ideas/{author}/` and stay open
forever. `michael-biehl/` already has 12+ ideas going back to Feb 2026,
mostly without a Status field, and `ideas/index.md` still claims "No ideas
captured yet" despite the directory being full. The triage / promote / park
/ archive flow described in the index is purely aspirational.

## What's missing

- No way to list open ideas (by author, by age, by theme).
- No triage UX — no command that walks you through pending ideas and
  records a decision.
- No promotion tracking — when an idea becomes a product spec or ticket,
  the link back to the originating idea is lost.
- No archival mechanism — old/irrelevant ideas just stay in place.
- `index.md` is not regenerated, so it drifts immediately.
- The "end-of-sprint digest via `/update-kb`" referenced in the index
  doesn't actually exist as a working pass.

## Two design directions

**(a) CLI-driven lifecycle.** Add `reins idea list` / `reins idea triage`
/ `reins idea archive`. `list` groups open ideas by author and age.
`triage` walks each open idea, prompts for a decision (promote / park /
archive), and on promote captures the link to the resulting spec or
ticket. `index.md` is regenerated on any change. Deterministic,
scriptable, easy to test.

**(b) Agent-driven lifecycle.** Make idea triage a pass inside
`/update-kb`. The agent reads open ideas, proposes a triage decision per
idea (promote candidate → draft a product-spec stub; park → leave note;
archive → reason), and the human confirms each in turn. Fits the
"humans steer, agents execute" framing.

The two aren't exclusive — (a) is the data layer, (b) is the workflow on
top. Probably build (a) first because it gives the agent something to
operate on.

## Schema sketch

Promote idea status frontmatter to first-class. Each idea file gets:

```
---
status: open | promoted | parked | archived
promoted_to: kb/reins/product-specs/foo.md  # if status == promoted
promoted_at: 2026-05-04
archived_reason: "Superseded by ..."         # if status == archived
---
```

Linter rule: ideas older than N days with `status: open` warn at the
author. `index.md` is generated from frontmatter, never edited by hand.

## Open questions

- Should ideas live forever or roll up into a monthly digest doc once
  archived?
- Cross-author triage: does the team triage `shared/` collectively at
  end-of-sprint, or does each author own their own dir?
- Promotion to a JIRA ticket vs. to an in-repo product-spec — both should
  work, but the link-back format differs.

## Notes

_No additional notes yet._
