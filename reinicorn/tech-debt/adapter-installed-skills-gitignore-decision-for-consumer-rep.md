---
type: debt
title: Adapter-installed skills gitignore decision for consumer repos
slug: adapter-installed-skills-gitignore-decision-for-consumer-rep
lifecycle: active
status: draft
created: 2026-08-20
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: skillset
severity: medium
remediation: planned
---

# Adapter-installed skills gitignore decision for consumer repos

## Impact

`upgrades/v0.3.md` originally claimed adapter-installed skills are
"gitignored", but nothing implements that for consumer repos: the tool only
ever writes one gitignore entry (`kb/`, via
`kb_setup.ensure_kb_gitignored`). In a consumer repo the ~13 adapter skill
trees land as untracked files, `rcorn init`'s instructions lead to
committing them, and `rcorn skills update` then rewrites tracked files —
noisy diffs and an unclear ownership story. (The Reinicorn repo itself
hand-edits its own `.gitignore` to dogfood.) The upgrade note was corrected
in the PR #57 review follow-up; the underlying design decision remains
open.

## Remediation Plan

Decide the intended model, then implement it:

- **Option A — gitignore adapter output:** on `rcorn skills install`, write
  gitignore entries for adapter-owned paths (mirroring
  `ensure_kb_gitignored`), with `!` negations for the native skills so they
  stay tracked. Repro-by-lockfile: fresh clones run
  `rcorn skills install` to materialize skills.
- **Option B — commit adapter output:** document that adapter skills are
  vendored/committed, and make `rcorn skills update` diffs the expected
  review surface.

Either way, make `upgrades/v0.3.md`, `GETTING-STARTED.md`, and the init
next-steps consistent with the chosen model, and add a test asserting the
gitignore behavior (or its deliberate absence).
