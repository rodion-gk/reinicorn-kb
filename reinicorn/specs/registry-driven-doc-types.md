---
type: spec
title: Registry-driven doc types
slug: registry-driven-doc-types
lifecycle: active
status: approved
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/10
---
# Registry-driven doc types

## Problem

Doc-type knowledge is scattered outside the central registry. A 2026-07-27 codebase audit found that adding a new doc type today requires touching at least six files: `cli.py` (hand-written parser group plus `_DISPATCH` rows), `doc_create.py` (a copied `_create_*` function and a `cmd_*` wrapper), `kb_seed.py` (README row), the shell linters (index-file and section lists), and shipped docs. A type added only to `REGISTRY` gets zero CLI surface and is invisible to seeding and shell linting.

There are four parallel creation paths (`doc_create.py`, `idea.py`, `plan.py`, principle's append mode) that have already drifted: idea's provenance block differs from the others by accident, and `_create_spec`/`_create_prd`/`_create_debt` are one function copied three times with only the template body varying.

This blocks the product direction: doc types should become fully customizable (name, gating, templates, custom props) from a central config. The registry cannot be the config-loading target until it is the only place type behavior lives.

## Design Goals

- **One registry row = one complete doc type.** Adding a type touches `doc_types.py` and nothing else: CLI group, dispatch, creation, seeding, and linting all derive from the row.
- Behavior is preserved for all seven existing types (`rcorn <type> --help` output may be normalized, but commands, paths, templates, and gating semantics do not change).
- The registry's field set is a superset of what a future `doc-types` config file needs, so config loading becomes a deserialization step, not another refactor.
- Latent crash removed: `gated=True` on a non-slug-addressed type currently produces wrong paths or a `KeyError` in `review.make_target`; after this spec it is a load-time validation error.

## Design

### New `DocType` fields

These are additions to the existing `DocType` dataclass, whose current fields already carry the path contract: `dir_path` plus the `filename` pattern (`{slug}.md`, `active/{branch}/plan.md`, `{username}/{slug}.md`, `golden-principles.md`) define every type's canonical destination, and creation, gating, and the review lane already format paths from them. `addressing` classifies those existing patterns to drive CLI argument shape and validation; it does not replace them.

- `template_body: str` — creation template with `{title}`, `{author}`, `{date}`, `{sections}`, `{text}` placeholders. Absorbs the spec/prd/debt/retro bodies, idea's inline body, plan's fallback body, and principle's item template.
- `addressing: Literal["slug", "branch", "singleton"]` — replaces the `"{branch}" in filename` sniff in `doc_create.py`. Drives CLI argument shape (slug arg / optional branch arg / none), which verbs are generated, and validation.
- `title_source: Literal["title", "free_text", "none"]` — subsumes `title_required`; distinguishes spec/prd/debt (`title` arg), idea (`text` free-form, title derived), plan/retro (none).
- `create_verb: str = "create"` — principle uses `add`.
- `create_mode: Literal["file", "append"] = "file"` — principle appends to its singleton file.
- `help_text: str` — CLI group help, currently hand-written in `cli.py`.
- `readme_label: str | None` — human label for the seeded kb README table; `None` means no row (idea today).

Load-time invariant, enforced with a plain `ValueError` raised where `REGISTRY` is defined (not an `assert`, which vanishes under `python -O`): `gated` implies `addressing == "slug"`.

Plan's `active/` / `completed/` / `_template/` lifecycle directories become module constants co-located with the registry in `doc_types.py` (killing six scattered `"active"` literals), not a registry field: the plan lifecycle stays bespoke code (see Non-Goals), so a field would be config-for-one. Revisit as a field only if a second lifecycle type appears.

### Refactor stages (one PR each, stackable, each shippable alone)

1. **Fields + generic creator.** Add the fields; one `_create_doc(dt, ...)` replaces `_create_spec/_prd/_debt/_retro/_principle`, the `_CREATORS` map, the five `cmd_*` wrappers, and `idea.py`'s bespoke creator (fixing its provenance drift). Plan keeps its lifecycle logic but routes file creation through the same entry. Add the gated⇒slug load-time check.
2. **CLI generation.** One loop over `REGISTRY` emits both parser groups and `_DISPATCH` rows from `addressing`/`create_verb`/`title_source`/`help_text`. Only plan's `status`/`complete` verbs remain hand-wired. Drop the defensive `getattr(a, "include_drafts", False)` defaults.
3. **Gate generalization + literal sweep.** `cmd_doc_check_path`'s exec-plans special case becomes a filename-tail loop over all protected types; the review lane's `rcorn spec list --include-drafts` hint derives from `gated_types()`; stray literals (`index.md`, `plan.md`, `retro.md`, `golden-principles.md`, the desynced `"exec-plans"` string in `plan_structure.py:40`, the `"active"` literals) resolve through the registry/constants.
4. **Seeding + linting.** `kb_seed` README rows from `readme_label` and templates from `template_body`; generalize `plan-structure` into a `required-sections` rule over every type with a non-empty `required_sections` (the existing `tuple[str, ...] = ()` registry field; an empty tuple means the rule skips the type); delete the shell linter duplicates (`docs-freshness.sh`, `plan-structure.sh`) in favor of the Python rules — they are the assets most likely to silently ignore a new type.
5. **Shipped docs + tests.** Genericize the per-type command tables in platform-instructions and the using-reinicorn skill (including its protected-path list); convert test noun enumerations (`test_cli_shape.py`, `test_dispatch.py`) to registry-derived parametrization. `test_doc_types.py`'s `expected_keys` pin stays as the single intentional enumeration.

### Testing

Each stage keeps the existing suite green (behavior-preserving), plus: a registry-invariant test (gated⇒slug, required fields non-empty), and a "phantom type" test that inserts a synthetic registry row and asserts it gets CLI surface, creation, and lint coverage with no other change — the executable form of the design goal.

## Non-Goals

- **The config file itself.** Loading user-defined types from a `doc-types` config (and the trust/validation questions that come with it) is a follow-up spec; this one makes the registry able to receive it.
- **Plan lifecycle as data.** `plan create`/`status`/`complete`, overlap detection, and the retro-rides-with-plan coupling are real product behavior and stay code.
- **Review-lane expansion.** Which types are gated does not change here; the lane stays specs-only.
- The correctness and test-health findings from the same audit — tracked separately in `tech-debt/kb-publish-atomicity-and-gh-boundary-fixes.md`, `tech-debt/test-suite-health-vacuous-update-test-host-isolation-coverag.md`, and `tech-debt/small-duplication-cleanups-from-quality-review.md`.
