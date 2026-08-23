---
type: retro
title: 'Retro: feat-skillset-adapters'
slug: feat-skillset-adapters
lifecycle: active
status: draft
created: 2026-08-20
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-skillset-adapters
---

# Retro: feat-skillset-adapters

## What Went Well

- The transactional installer design (backup/rollback, `preserve_work`,
  collision/local-edit checks) absorbed every review fix without
  restructuring — migration adoption (`adopt_hashes`) threaded through the
  existing transaction seams.
- Byte-identical fork→patch cutover was strong migration evidence during
  development, and could be retired cleanly (review follow-up fixed 7
  inherited content bugs, deliberately ending byte-identity) once it had
  done its job.
- Review response ran as verify-first subagent waves: every finding was
  independently verified against the codebase before any fix, which
  reclassified 7 of 17 CodeRabbit findings as inherited-from-fork and
  changed the fix strategy (regenerate patches, delete one outright).
- Patch regeneration method (apply→edit content→re-diff, never hand-edit
  hunk internals) made the content fixes mechanical and reviewable.

## What Could Be Improved

- Mocked installer tests hid that legacy migration could never succeed on
  a real repo (collision check aborted on the very fork dirs the migration
  replaces). External review caught it; a real-installer integration test
  should have existed from the first migration commit.
- `using-reinicorn` started as a too-close copy of `using-superpowers` and
  needed a full rewrite in review. Skills should be written in the tool's
  own voice from the start, stating what the tool owns rather than
  inheriting another skill's framing.
- A banned-word test fossilized a rejected idea (change-detector
  anti-pattern) and had to be removed.

## Lessons Learned

- Migration/upgrade paths need at least one end-to-end test through the
  real installer; mocks validate the plumbing but not the preconditions.
- Regression tests assert desired behavior, never the absence of a
  rejected approach — codified as golden principle 16, "No change-detector
  tests" (reviewer heuristic: "what desired behavior fails if this
  trips?").
- Vendored/forked content is not authoritative: reviewing the adapter
  patches surfaced 7 latent content bugs that had lived silently in the
  forks.

## Action Items

- Tech debt recorded: managed refresh for the symlink-unavailable compat
  skills copy
  (`tech-debt/managed-refresh-for-symlink-unavailable-compat-skills-copy.md`).
- Deferred plan 2: mattpocock-skills/wayfinder adapter +
  `authoring-skillset-adapters` skill (separate branch/spec).
- After the integration branch (`feat-registry-doc-types`) merges to main:
  reinstall the rcorn tool. 
