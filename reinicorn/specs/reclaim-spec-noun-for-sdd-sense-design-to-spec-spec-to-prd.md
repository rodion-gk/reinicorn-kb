---
type: spec
title: Reclaim spec noun for SDD sense (design to spec, spec to prd)
slug: reclaim-spec-noun-for-sdd-sense-design-to-spec-spec-to-prd
lifecycle: done
status: complete
created: 2026-06-08
author: Michael Biehl
origin: ai-assisted
human_validated: true
---

# Reclaim spec noun for SDD sense (design to spec, spec to prd)

## Problem

The reins CLI exposes a `spec` noun (`reins spec create`) that produces a narrow
"product-spec" document (user stories, acceptance criteria) under
`kb/<repo>/product-specs/`. This collides with the canonical meaning of "spec" in
the spec-driven-development methodology that reins lives inside — and specifically
with our upstream superpowers skills. There, a *spec* is **the specification that
drives implementation**: the artifact produced by brainstorming and the thing the
`subagent-driven-development` spec-reviewer checks code against ("spec compliance",
"spec gaps").

Worse, the artifact that actually plays the SDD "spec" role in reins today is
created by a *different* command — `reins design create` — which the brainstorming
skill saves to `design-docs/` and refers to as "the validated design (spec)". So
the word "spec" is attached to a minor doc type while the real spec is called
something else. This is a semantic mismatch with our own foundational methodology.

Usage data underlines it: the kb holds **36 design docs** but only **1 real
product-spec** (`smoke-test-spec.md`; `index.md` is generated). The `design` doc
type carries the weight; the `spec` doc type barely earns its keep and is squatting
on the more important word.

## Design Goals

- `reins spec create` produces **the** pre-implementation contract — the artifact
  brainstorming outputs and implementation is judged against (true SDD sense).
- The word "spec" in the reins CLI matches its meaning in the upstream superpowers
  skills, with no lingering second meaning.
- The existing product-requirements doc type survives under an unambiguous name
  (`prd`) with its template intact.
- Command nouns match on-disk kb directory names — no command/directory mismatch.
- All 37 existing docs are preserved (moved, not lost), with git history intact.
- The full test suite passes; `reins --help` and the per-noun help reflect the new
  surface; no stale references remain outside intentionally-frozen kb history.

## Design

### Naming

| Today | Becomes | Role | Template (unchanged) |
|-------|---------|------|----------------------|
| `reins design create` | `reins spec create` | THE pre-implementation contract; brainstorming output; spec-reviewer's reference | Problem · Design Goals · Design · Non-Goals |
| `reins spec create` (product-spec) | `reins prd create` | Product requirements doc | Overview · User Stories · Acceptance Criteria · Out of Scope · Open Questions |

The noun `design` is removed entirely. `spec` is reclaimed for the artifact `design`
used to produce. `prd` is introduced for the former product-spec.

### Directory layout (kb)

- `kb/<repo>/design-docs/` → `kb/<repo>/specs/` (36 docs + `index.md`)
- `kb/<repo>/product-specs/` → `kb/<repo>/prds/` (1 doc + `index.md`)

Both moves are done with `git mv` to preserve history.

Each directory has a hand-maintained `index.md` (an explainer of what the doc type
is — not generated). These need **content rewrites**, not just path fixes: the
current `design-docs/index.md` defines design docs and even contrasts them against
"product specs"; the new `specs/index.md` must explain the reclaimed SDD spec, and
`prds/index.md` must explain the PRD. Any cross-references between them are updated
to match.

### Code changes (`src/reins/`)

- **`doc_types.py`** — the single source of truth. Rename registry key `design` →
  `spec` (dir_path `specs`) and old key `spec` → `prd` (dir_path `prds`). Each key
  keeps its existing `required_sections`. Everything that reads dir names from the
  registry (`status.py`, `linter/rules/docs_freshness.py`, path protection via
  `get_protected_map`/`by_dir`) follows automatically.
- **`commands/doc_create.py`** — `_create_design` → `_create_spec`; old
  `_create_spec` → `_create_prd`; `cmd_design_create` → `cmd_spec_create`; old
  `cmd_spec_create` → `cmd_prd_create`; update the `_CREATORS` dict and the
  `_check-path` protection error strings ("Use `reins spec create` …", "Use
  `reins prd create` …").
- **`cli.py`** — replace the `design` noun group with `spec`, and the old `spec`
  group with `prd`, in both `_build_parser` and `_dispatch`.

### Docs / skills / linters / tests sweep

Mechanical replacement of the old surface across:

- **Skills:** `brainstorming/SKILL.md` (its own "save via `reins design create`" →
  `reins spec create`; `design-docs/` → `specs/`), `using-reins/SKILL.md` (command
  table), `requesting-code-review/SKILL.md`,
  `update-superpowers/replacements.yaml` (keep upstream merges mapping correctly).
- **Top-level docs:** `README.md`, `AGENTS.md`, `GETTING-STARTED.md`,
  `.claude/hooks/README.md`.
- **Platform instructions:** `claude.md`, `copilot.md`, `cursor.md`, `gemini.md`.
- **Linters:** `linters/rules/kb/docs-freshness.sh` (hardcoded dir names).
- **Tests:** `test_doc_create.py`, `test_cli_shape.py`, `test_doc_types.py`.

### Verification

- `uv run pytest` green.
- `reins --help`, `reins spec --help`, `reins prd --help` show the new surface;
  `reins design` errors with "invalid choice".
- Grep sweep: no `reins design create`, `reins spec create` (old product-spec
  sense), `design-docs/`, or `product-specs/` references outside frozen kb history.
- `reins kb lint` passes (freshness rule resolves new dirs/indexes).

## Non-Goals

- **Not rewriting historical kb prose.** Docs under `exec-plans/completed/` and the
  prose inside moved docs keep their original wording per the existing convention;
  only paths that physically move are corrected.
- **Not changing any other noun** (`debt`, `idea`, `plan`, `retro`, `principle`,
  `kb`, `mode`). This is scoped to the `design`/`spec`/`prd` triad.
- **Not changing the `prd` template.** It keeps today's product-spec sections; only
  its name and directory change.
- **Not introducing per-repo custom doc types** (still out of scope, as noted in
  `doc_types.py`).
- **Not a new branch.** This work rides on `feature/cli-noun-first-restructure`
  (PR #21), extending that PR rather than stacking a separate one.
