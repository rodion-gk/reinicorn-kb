---
type: debt
title: Managed refresh for symlink-unavailable compat skills copy
slug: managed-refresh-for-symlink-unavailable-compat-skills-copy
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

# Managed refresh for symlink-unavailable compat skills copy

## Impact

On platforms where `maintain_link()` cannot create the `.claude/skills` ->
skills-dir compatibility symlink (e.g. Windows without symlink privilege),
it falls back to a one-time real-directory copy. That copy is unmanaged:
later `rcorn update` / `rcorn skills update` runs refresh the real skills
dir but never the copy, so it silently drifts stale. PR #55 shipped the
minimal mitigation only — a stale *symlink* is detected and relinked, and
an unmanaged real directory triggers a warning telling the user to remove
it and rerun `rcorn update` (see the CodeRabbit thread on
`src/reinicorn/skillset/installer.py` in PR #55). The lifecycle gap itself
remains: no ownership tracking, no automatic refresh.

## Remediation Plan

Give the fallback copy a managed lifecycle:

- Record ownership of a fallback copy (e.g. a marker file inside the copy
  or an entry in the lock/manifest) so `maintain_link()` can distinguish
  "copy we made" from a user-created directory.
- On `rcorn update` / `rcorn skills update`, when the compat path is an
  owned copy, re-sync it from the skills dir (or replace it with a symlink
  if symlinks have become available).
- Keep the current warning for genuinely unmanaged real directories.
- Tests: owned-copy refresh, owned-copy upgraded to symlink, unmanaged dir
  still warns and is left untouched.
