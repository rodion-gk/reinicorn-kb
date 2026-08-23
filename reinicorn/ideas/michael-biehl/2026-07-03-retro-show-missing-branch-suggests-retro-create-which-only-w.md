---
type: idea
title: retro show <missing-branch> suggests retro create which only works for current b
slug: 2026-07-03-retro-show-missing-branch-suggests-retro-create-which-only-w
lifecycle: done
status: resolved
created: 2026-07-03
author: Michael Biehl
---

# retro show <missing-branch> suggests retro create which only works for current b

**Resolved:** 2026-07-24

## Description

retro show <missing-branch> suggests retro create which only works for current branch; no plan list / retro list verbs to discover valid branches — agent discoverability dead-end

## Resolution

Fixed on branch fix/retro-show-branch-discovery. `retro show` / `plan show`
for a missing doc now only suggest the create command when the requested
branch is the current branch; for any other branch they list the branches
that do have the doc ("branches with a retro: …" — completed and active
retros both covered), or print a definitive "retros: 0 found". Discovery
happens inline at the point of failure, so dedicated `plan list` /
`retro list` verbs were not needed for this; they remain optional future
work.

- Code: `src/reinicorn/commands/doc_show.py` (`_missing_branch_doc`)
- Plan: `kb/reinicorn/exec-plans/active/fix-retro-show-branch-discovery/`
