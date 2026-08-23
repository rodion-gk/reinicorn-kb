---
type: plan
title: 'Execution Plan: fix/plan-create-frontmatter'
slug: fix-plan-create-frontmatter
lifecycle: done
status: complete
created: 2026-08-07
author: Michael Biehl
branch: fix/plan-create-frontmatter
ticket: N/A
spec: reinicorn/specs/fix-plan-create-frontmatter-so-the-push-gate-can-pass.md
---

# Execution Plan: fix/plan-create-frontmatter

## Goal

Fix #41: `rcorn plan create` emits a frontmatter-less `plan.md` that the
pre-push spec gate then rejects. Implements the approved spec named above.
(This plan doc was itself created by the broken path and hand-converted —
the last one that should ever need it.)

## Acceptance Criteria

- [ ] A plan created against a stale (frontmatter-less) template, a seeded
      template, or no template carries the full unified frontmatter with the
      `spec:` placeholder — one field-edit away from a passing push.
- [ ] The placeholder string is a shared constant next to
      `SPEC_PLACEHOLDER_RE`; generator and gate cannot drift.
- [ ] `kb_seed.py`'s template includes `spec:`.
- [ ] This kb's live `_template/plan.md` is migrated to unified frontmatter.
- [ ] Born-passing test: a fresh doc of every registry type passes
      `rcorn kb lint` clean.

## Approach

Per the spec: harden the template loop in `commands/plan.py` (unconditional
meta injection, `setdefault` on `spec`), share the placeholder constant from
`linter/spec_refs.py`, fix the seeded template, migrate the live template,
and add the three-path parity + born-passing tests.

## Tasks

- [ ] Harden `plan.py` meta injection; add `SPEC_PLACEHOLDER` constant
- [ ] Add `spec:` to `kb_seed.py` template
- [ ] Migrate `kb/reinicorn/exec-plans/_template/plan.md`
- [ ] Tests: parity, placeholder/regex agreement, born-passing sweep
- [ ] Full suite + lint, PR closing #41

## Dependencies

Approved spec `reinicorn/specs/fix-plan-create-frontmatter-so-the-push-gate-can-pass.md`.
Non-goals #35 (path-scoped commits) and registry-driven doc types remain
separate.
