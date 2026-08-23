---
type: spec
title: Fix plan create frontmatter so the push gate can pass
slug: fix-plan-create-frontmatter-so-the-push-gate-can-pass
lifecycle: active
status: approved
created: 2026-08-04
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/12
---

# Fix plan create frontmatter so the push gate can pass

## Problem

Every plan created with `rcorn plan create` in this kb is blocked at push by
the spec-approval gate (#27):

```text
❌ Push blocked: kb/reinicorn/exec-plans/active/<slug>/plan.md builds on an
   unapproved spec.
   its 'spec:' frontmatter field is missing or still the template placeholder.
```

The generated `plan.md` has no YAML frontmatter at all — just the old
`**Ticket:** / **Spec:** / **Status:**` bold-label header — so the gate can
never pass without hand-converting the whole doc. This is issue #41, found
while preparing #40, and it is the same generator/checker drift that #34
diagnosed for ideas: the tool creates a doc that immediately fails the check
the tool itself imposes.

Three paths in `plan create` produce three different docs today:

1. **Stale template (this kb's actual state).** The #30 frontmatter migration
   converted every kb doc but missed `exec-plans/_template/plan.md`, and the
   template loop in `src/reinicorn/commands/plan.py` injects the standard
   meta only `if meta:` — an empty frontmatter parse falls through to writing
   the body verbatim, frontmatter-less. Silent tolerance of a stale template
   is the code half of the bug.
2. **Seeded template.** `kb_seed.py` writes a frontmatter template for new
   kbs, but it omits `spec:`, and the `meta.update({...})` in `plan.py` never
   sets it either. A freshly seeded kb therefore produces plans that the gate
   also blocks — with no in-doc placeholder telling the author what to fill
   in.
3. **No template.** The fallback branch renders full frontmatter including
   the `spec:` placeholder the gate recognizes. A kb with *no* template
   produces a better plan doc than a kb with either real template.

The gate itself is behaving correctly: `declared_spec` deliberately treats a
missing field and the placeholder alike, forcing the author to declare a spec
or `N/A` before pushing. The defect is entirely on the generator side — the
created doc should arrive one field-edit away from a passing push, not a
hand-conversion away.

## Design Goals

- **One edit to green.** A plan created by `rcorn plan create` carries the
  full unified frontmatter — including the `spec:` placeholder the gate
  recognizes — in every kb state: stale template, seeded template, or no
  template. Filling in `spec:` (or `N/A`) is the only edit between create
  and a passing push.
- **All three creation paths produce the same frontmatter key set**, so a
  kb's template age can no longer change what the tool emits.
- **The placeholder is defined once.** The generator writes the same
  placeholder string the gate's `SPEC_PLACEHOLDER_RE` matches, from a shared
  constant, so the two cannot drift apart again.
- **Born-passing docs are tested.** A freshly created doc of every type
  passes `rcorn kb lint` clean — the regression test #34 proposed, which
  would have caught this.

## Design

### 1. Harden `plan.py` (the durable fix)

In the template loop of `src/reinicorn/commands/plan.py`, inject the
standard meta unconditionally instead of `if meta:`:

- An empty parse starts from `{"type": "plan"}` rather than falling through
  to a verbatim body write.
- After the existing `meta.update({...})`, add
  `meta.setdefault("spec", SPEC_PLACEHOLDER)` so a template that lacks the
  field (every seeded kb today) still yields an editable placeholder. The
  update — not the template — remains the source of truth for the standard
  fields; the template contributes only fields the update does not set.

`SPEC_PLACEHOLDER` becomes a module-level constant exported next to
`SPEC_PLACEHOLDER_RE` in `reinicorn/linter/spec_refs.py`, replacing the
string literal currently inlined in `plan.py`'s fallback branch, and the
regex is asserted (in tests) to match the constant.

This half alone fixes every downstream kb, whatever the age of its template.

### 2. Fix the seeded template in `kb_seed.py`

Add `spec: '[kb path to the spec this implements, or N/A]'` (via the shared
constant) to the frontmatter block `kb_seed.py` writes for
`_template/plan.md`, so seeded templates and the hardened injection agree
rather than relying on `setdefault` to paper over the gap.

### 3. Migrate this kb's live template

Convert `kb/reinicorn/exec-plans/_template/plan.md` to the same shape the
seeder writes: unified frontmatter (with the `spec:` placeholder) plus the
`# Execution Plan:` heading and the existing body sections (Goal, Acceptance
Criteria, Approach, Tasks, Dependencies). The bold-label header lines whose
content moved into frontmatter are dropped from the body. This is a kb
change, not a code change, and rides in the same PR's kb pointer update.

### Testing

- Template-less, stale-template (no frontmatter), and seeded-template kbs
  all produce a `plan.md` whose parsed frontmatter has the identical key
  set, including `spec:` set to the placeholder.
- `declared_spec` returns `None` for the placeholder written by create (the
  gate still forces the declaration), and the shared constant matches
  `SPEC_PLACEHOLDER_RE`.
- Born-passing sweep: for each doc type in the registry, create a fresh doc
  and assert `rcorn kb lint` reports no violations for it. Plans run through
  the real template path, not just the fallback.

## Non-Goals

- **Gate semantics.** The gate keeps blocking on a placeholder `spec:` —
  that is the review lane working as designed, not a bug.
- **Path-scoped create commits.** `plan create` likely shares #35's
  whole-tree commit behavior; that is #35's fix, not this one.
- **Migrating downstream kbs' templates.** Other kbs keep their stale
  templates; the `plan.py` hardening makes them harmless. No re-seeding or
  auto-migration machinery.
- **Registry-driven templates.** The registry-driven-doc-types spec absorbs
  template bodies into the registry as `template_body`; this fix is minimal
  and compatible with that refactor, which supersedes rather than conflicts
  with it.
