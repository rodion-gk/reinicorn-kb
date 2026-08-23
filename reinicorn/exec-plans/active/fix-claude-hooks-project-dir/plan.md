---
type: plan
title: 'Execution Plan: fix/claude-hooks-project-dir'
slug: fix-claude-hooks-project-dir
lifecycle: active
status: in-progress
created: 2026-08-11
author: Michael Biehl
branch: fix/claude-hooks-project-dir
ticket: N/A
spec: N/A
---

# Execution Plan: fix/claude-hooks-project-dir

## Goal
`rcorn hooks install` writes bare-relative Claude Code hook commands
(`.reinicorn/hooks/foo.sh`) into `.claude/settings.json`. Claude Code runs hook
commands via /bin/sh from wherever the session was launched, so these break in
worktrees and subdirectory sessions. Emit `"$CLAUDE_PROJECT_DIR"`-prefixed
commands instead, and repair stale entries left by earlier installs.

## Acceptance Criteria
- [x] New installs write `"$CLAUDE_PROJECT_DIR"/.reinicorn/hooks/<script>` commands
- [x] Re-running install replaces stale `.reins/hooks/` (pre-rename) entries
- [x] Re-running install upgrades bare-relative `.reinicorn/hooks/` entries
- [x] Unrelated user hooks on the same matcher are preserved
- [x] Success message reports repaired count alongside added count

## Approach
Stale entries share the matcher with their replacements, so matcher-only dedup
would keep a broken hook alive forever. `_merge_claude_settings` now drops
entries whose command matches a known stale path (`.reins/hooks/*` or
bare-relative `.reinicorn/hooks/*`) before the dedup pass. This is the
settings.json counterpart to the shell-side stale-hook repair in #37; the
repo's own settings were hand-fixed in #16 but the installer kept emitting
relative paths for every other repo.

## Tasks
- [x] Prefix generated Claude hook commands with `"$CLAUDE_PROJECT_DIR"`
- [x] Strip stale `.reins/` and bare-relative entries in `_merge_claude_settings`
- [x] Report repaired count in the success message
- [x] Tests: stale-reins repair, bare-relative migration, unrelated-hook preservation

## Dependencies
Follows #16 (quoted `$CLAUDE_PROJECT_DIR` in this repo's own settings) and
#37 (shell-side stale reins-era hook repair). No open-branch overlap.
