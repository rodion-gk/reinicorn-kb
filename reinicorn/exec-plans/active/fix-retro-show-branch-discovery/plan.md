---
type: plan
title: 'Execution Plan: fix/retro-show-branch-discovery'
slug: fix-retro-show-branch-discovery
lifecycle: done
status: complete
created: 2026-07-24
author: Michael Biehl
branch: fix/retro-show-branch-discovery
ticket: N/A
spec: N/A
---

# Execution Plan: fix/retro-show-branch-discovery

## Goal
Fix the agent-discoverability dead-end from idea 2026-07-03: `rcorn retro show <missing-branch>` (and `plan show`) suggested `rcorn retro create` / `rcorn plan create`, which only operate on the current branch, and offered no way to discover which branches have docs.

## Acceptance Criteria
- [x] `retro show` / `plan show` for a missing doc on the **current** branch keeps the create hint (still valid there)
- [x] For any **other** branch: no create hint; list branches that have the doc ("branches with a retro: …"), mirroring cmd_doc_show's "valid slugs" pattern
- [x] Retro branch discovery covers both completed/ retros and active-plan-dir retros
- [x] Zero-docs case prints a definitive "retros: 0 found" / "plans: 0 found" with no misleading next step
- [x] Full test suite and pyright pass

## Approach
TDD in `src/reinicorn/commands/doc_show.py`: shared `_missing_branch_doc()` helper branching on `branch == current_branch()`, with branch discovery via registry filename globs (`{branch}` → `*`). Two existing tests asserted the buggy behavior and were replaced.

Also investigated idea 2026-03-13 ("plan create ignores name argument"): obsolete — the current CLI takes no name argument and argparse rejects extras loudly (exit 2). Resolved without code change.

## Tasks
- [x] Failing tests for other-branch miss (list branches, no create hint) and zero-docs case
- [x] Implement `_missing_branch_doc` + discovery globs in doc_show.py
- [x] Verify live: `rcorn retro show nonexistent-branch`, `plan show`, current-branch miss
- [x] Resolve ideas 2026-07-03 (retro-show dead-end) and 2026-03-13 (plan create name arg, obsolete)

## Dependencies
None. The `plan list` / `retro list` verbs mentioned in the idea remain optional future work (discovery now happens inline at the point of failure).
