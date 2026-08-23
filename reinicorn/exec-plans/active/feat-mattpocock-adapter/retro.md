---
type: retro
title: 'Retro: feat-mattpocock-adapter'
slug: feat-mattpocock-adapter
lifecycle: active
status: draft
created: 2026-08-20
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: feat-mattpocock-adapter
---

# Retro: feat-mattpocock-adapter

## What Went Well

- The adapter grammar from `feat-skillset-adapters` generalised without a
  single engine change: 12 files / 378 lines on this branch, all under
  `adapters/`, `tests/`, `docs/`, README. Category-nested upstream paths
  (`skills/engineering/<name>` → flat installed name) were already covered
  by the generic installer tests, so the second adapter was pure content.
- Keeping the tracker contract in a repo doc (`docs/agents/issue-tracker.md`)
  instead of patching each skill's tracker prose kept the patch series to
  four small pointer swaps; upstream's own config-driven design made that
  the natural seam.
- The patch/append split held: `## Reinicorn` trailers went in as appends,
  so only genuine prose edits live in `.patch` files and future upstream
  rebases touch the minimum.
- Patch regeneration stayed mechanical (extract pinned tarball → apply →
  edit → re-diff → `git apply --check` against a fresh extract), including
  for the mid-branch fix. No hand-edited hunks.
- CI green on the first push (pytest, kb lints, structural tests, CodeQL).

## What Could Be Improved

- The `code-review` patch shipped (commit 5693220) pointing at "the
  mattpocock-skills adapter notes" — a doc this adapter never installs.
  Only the schema tests written afterwards in Task 2 caught it. Content
  assertions on patch replacement text should have been written alongside
  the patch, not a commit later.
- `docs/agents/issue-tracker.md` documented the wayfinder frontier query as
  `gh issue list --json ...` "filtered to open, unassigned, blockedBy
  empty" — but `--json` selects fields, it doesn't filter. CodeRabbit
  flagged it as Major (PR #59, still open). The doc was described as
  "verified against gh 2.97" in the PR body, but the *semantics* of the
  query were never exercised against a real map issue.
- Plan drift: acceptance criteria name three patches
  (to-spec/to-tickets/code-review); four shipped (wayfinder too). The plan
  doc's AC checkboxes are also still unticked while all three tasks are
  ticked — the plan wasn't kept current as scope shifted.

## Lessons Learned

- A skill-contract doc the installed skills *read* is executable prose:
  every command in it should be run once end-to-end, not just
  `--help`-checked for flag existence.
- Patch replacement text must only reference things the adapter ships or
  the repo commits. Worth a generic adapter test: every path/doc named in a
  patch's `+` lines resolves to an installed skill, a `files:` entry, or a
  committed repo path.
- When an adapter swaps upstream setup pointers for a repo-level contract,
  that contract doc *is* part of the adapter's surface — review it with the
  same rigour as the patches.

## Action Items

- Resolve the two open CodeRabbit findings on PR #59: `ATTRIBUTION.md`
  license fence → ```` ```text ```` (MD040); frontier query → add
  `--state open` and a `--jq` expression selecting unassigned children of
  the map with empty `blockedBy` — and run it against a real map issue.
- Tick the plan's AC checkboxes and correct the patch count before
  `rcorn plan complete`.
- Post-merge, local only: `rcorn skills install mattpocock-skills` in this
  repo, create the `ready-for-agent` label, reinstall the `rcorn` tool.
- README overlap with `docs-readme-doc-governance` — merge whichever lands
  second carefully; this branch's README change was kept to one section.
- Still deferred from plan 1: `authoring-skillset-adapters` skill (own
  branch). Candidate addition: the patch-reference-resolves test above.
