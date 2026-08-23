---
type: plan
title: 'Execution Plan: feature/mvp-unified-attach'
slug: mvp-unified-attach
lifecycle: done
status: completed
created: 2026-02-19
author: mnbiehl
branch: feature/mvp-unified-attach
ticket: N/A
---

# Execution Plan: feature/mvp-unified-attach

## Goal
Bring reins to MVP — teammates can pip install, run `reins attach`,
and get working agent instructions + git hooks for Cursor and Copilot.

## Acceptance Criteria
- [x] `pip install git+<repo>` puts `reins` on PATH
- [x] `reins attach` in an existing repo sets up harness submodule
- [x] `.claude/skills/` copied to target repo (works for Cursor, Copilot, Claude Code)
- [x] AGENTS.md copied to target repo
- [x] Git hooks installed via Python (no bash dependency)
- [x] Hooks have no dead `bin/reins` fallback code
- [x] `reins feedback` opens a GitHub issue with context
- [x] All tests pass, linters clean

## Approach
Unified Attach — single Python `reins attach` command replaces the bash
script. Submodule-only (inline mode removed). Multi-tool agent instructions
via `.claude/skills/` universal format. New feedback command for bug reporting.

Design doc: `harness/reins/design-docs/mvp-unified-attach.md`
Full implementation plan: `harness/reins/exec-plans/active/2026-02-19-mvp-unified-attach-plan.md`

## Tasks
- [x] Task 1: Add PyPI metadata to pyproject.toml
- [x] Task 2: Remove inline mode from attach command
- [x] Task 3: Add .claude/skills/ copying to attach
- [x] Task 4: Switch attach to Python hook installation
- [x] Task 5: Clean up git hook scripts (remove bin/reins fallback)
- [x] Task 6: Add `reins feedback` command
- [x] Task 7: Update AGENTS.md for new commands
- [x] Task 8: Run full test suite and fix issues
- [x] Task 9: Manual integration test

## Dependencies
None — this is foundational MVP work.
