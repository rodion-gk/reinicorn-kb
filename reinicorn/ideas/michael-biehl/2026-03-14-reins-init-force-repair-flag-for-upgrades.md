---
type: idea
title: reins init --force/--repair flag for upgrades
slug: 2026-03-14-reins-init-force-repair-flag-for-upgrades
lifecycle: active
status: new
created: 2026-03-14
author: Michael Biehl
---

# reins init --force/--repair flag for upgrades

## Description

`reins init` currently has two modes: full setup (no harness) and teammate-clone (harness exists). There's no upgrade path for repos initialized with an older reins version — running `init` again just installs hooks and exits early.

## Problem

When reins gains new assets (skills, hooks, templates), existing repos can't pick them up via `init`. The `update` command handles version-tracked files in the manifest, but can't add entirely new asset categories or fix a broken setup.

## Proposed Solution

Add `--force` or `--repair` flag to `reins init` that:
- Re-copies all assets (skills, hooks, linters, AGENTS.md template) even if harness already exists
- Respects locally-modified files (same skip logic as `update`)
- Re-runs hook installation and settings.json merge
- Writes a fresh manifest

Distinct from `update` because `update` only touches files already in the manifest. `--force` would re-run the full init flow regardless of existing state.

## Open Questions

- Should it be `--force` (scary name) or `--repair` (implies something is broken)?
- Could `update` be extended instead of adding a flag to `init`?
- Should it prompt before overwriting or always skip locally-modified files?
