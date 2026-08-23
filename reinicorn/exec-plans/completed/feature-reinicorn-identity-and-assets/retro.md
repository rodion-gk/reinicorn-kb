---
type: retro
title: 'Retro: feature/reinicorn-identity-and-assets'
slug: feature-reinicorn-identity-and-assets
lifecycle: active
status: draft
created: 2026-07-16
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feature-reinicorn-identity-and-assets
---

# Retro: feature/reinicorn-identity-and-assets

## What Went Well

- Commit-based task tracking made picking up mid-flight work reliable: mapping
  commit messages to plan tasks reconstructed exactly where the prior agent
  stalled (Task 6 committed; a follow-on scanner refactor left uncommitted and red).
- Red-green TDD per task kept every checkpoint independently verifiable. The full
  suite + legacy-identity scan + isolated wheel smoke test caught regressions
  immediately rather than at release time.
- The `block-raw-kb-git` hook correctly blocked raw `git -C kb` and forced the
  `rcorn kb git` escape hatch — the guardrail did its job under real use.
- The coordinated-cutover precondition (`kb sync` reporting "No overlap with other
  active branches") gave an unambiguous go/no-go before the shared KB scope move.
- Smoke-testing the built wheel in an isolated venv verified the real packaged
  artifact (entry point, `_data` asset groups, no `reins` executable) — not just source.

## What Could Be Improved

- The scanner hardening inherited from the prior agent over-reached: "fail closed
  on any dynamic executable position" flagged a legitimate real workflow line
  (`"$test_file"` in `lint-architecture.yml`). Strictness meant for untrusted input
  broke the repo's own trusted workflow.
- The plan's own legacy-identity scan used a substring check (`"reins" in
  text.lower()`) that false-positived on "Reinstall" — a defect in the test spec,
  surfaced only at execution.
- Task 6 flipped the config scope to `reinicorn` before the physical KB move in
  Task 7, opening a window where `plan show` / `spec show` could not resolve their
  own scope. The transient breakage was expected but avoidable with tighter sequencing.
- Plan checkboxes were never updated; the `- [ ]` state diverged from commit
  reality, which is confusing for anyone reading the plan mid-execution.

## Lessons Learned

- Identity-rename scans must match on word boundaries, not substrings: `reins` is a
  prefix of `reinstall` and lives inside `reins-kb`. Word-boundary matching is the
  correct default for any rename-enforcement check.
- When a rename touches semantic values (git commit identity, file-destination
  paths, branch prefixes, remote URLs), a blind `sed` is unsafe. Inventory the
  landmines first, special-case them, and let the full suite confirm behavior.
- "Fail closed" is right for untrusted input and wrong when it breaks the project's
  own legitimate usage. Scope the strictness to the actual threat surface.
- Flipping a config pointer before the physical target it names creates a dangling
  window. Sequence identity/scope migrations so lookups never resolve to nothing.

## Action Items

- Gate 4: rename the GitHub repo and KB remote to `reinicorn` / `reinicorn-kb`,
  update `.gitmodules`, and publish to PyPI (removing the "not yet on PyPI" caveats).
- Consider a lint (or a `plan complete` step) that reconciles unchecked `- [ ]`
  boxes against commit reality so plan state does not silently drift.
- Decide whether `HUMANS.md` gets a light identity pass or stays deliberately hand-authored.
- Address the Gate 3 KB backlog surfaced by `kb lint`: 150 stale docs (>30 days) and
  the docs-freshness / plan-structure / provenance warning families.
