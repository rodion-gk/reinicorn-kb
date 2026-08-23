---
type: idea
title: Are 'idea' and 'debt' the right nouns? 'reins debt create' reads weird. Alternat
slug: 2026-07-04-are-idea-and-debt-the-right-nouns-reins-debt-create-reads-we
lifecycle: active
status: new
created: 2026-07-04
author: Michael Biehl
---

# Are 'idea' and 'debt' the right nouns? 'reins debt create' reads weird. Alternat

## Description

Are 'idea' and 'debt' the right nouns? 'reins debt create' reads weird. Alternatives floated: backlog, issue, debt-log. To evaluate: whether the awkwardness is in the noun or in the verb-noun pairing (cf. 'reins principle add', which already deviates from create), collision risks (issue vs GitHub issues), and kb dir-layout churn if nouns rename (tech-debt/, ideas/{username}/).

## Notes

Discussion trail (2026-07-04 → 2026-07-06):

- Verb fix considered first: keep nouns, use native collocations per type
  (`idea capture`, `debt log`), precedent `principle add`. Verb changes are
  cheap (dispatch key + create_hint); noun renames churn kb dir layout.
- Decided: `debt` is the wrong noun. Rejected alternatives:
  `issue` (collides with GitHub issues / `reins feedback`), `backlog` (names a
  collection, flattens the personal `ideas/{username}/` space), `debt-log`.
- Converging duality: **idea** = new thing we should consider (forward-looking)
  vs a counterpart noun for "describes what already exists and suggests
  change". Proposed alternative: **report**. This also widens the too-narrow debt
  bucket (observed gaps/bugs/perf problems all fit).
- Open naming question: "report" can be misread as *producing* a summary
  (`reins report` → stats?). The filing verb disambiguates ("file a report");
  the alternative noun **finding** avoids the ambiguity entirely and matches
  review-workflow language ("review findings").
- Migration surface when picked up: registry entry (key, dir_path tech-debt/ →
  new dir, create_hint), existing kb docs move, skills/README mentions,
  templates. Pre-release: no deprecation wrappers (golden principle #11).
