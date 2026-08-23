---
type: spec
title: unified kb doc frontmatter schema
slug: unified-kb-doc-frontmatter-schema
lifecycle: active
status: approved
created: 2026-07-22
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/3
---

# unified kb doc frontmatter schema

## Problem

Every kb doc type (exec-plan, retro, idea, spec, prd, tech-debt, principle)
stores its metadata as a **prose provenance block** — a `# Title` H1 followed
by bold `**Key:** value` lines. There is no machine-readable structure. Every
consumer regex-scans human prose to recover fields:

- `doc_show.py` does `line.startswith("**Status:**")`.
- `status.py` line-scans for status across scopes.
- `plan.py` recovers the branch name with `sed -n 's/^# Execution Plan: //p'`.
- `.github/workflows/doc-gardening.yml` orphan detection has **no** field to
  read the branch from at all, so it sanitizes remote heads (`tr '/' '-'`) and
  compares them to sanitized plan-dir names.

That last point is a live correctness bug (surfaced by CodeRabbit, root-caused
here): sanitization is lossy and many-to-one (`/` and `\` both map to `-`), so
a deleted `feature/mvp` plan is not flagged as orphaned if a literal
`feature-mvp` branch happens to exist. In a repo with mixed slash/dash naming
conventions this is not rare. The branch name is *recoverable* today only
because `plan.py` writes it into the H1 (`# Execution Plan: <branch>`), but no
consumer reads it structurally — it is trapped in a heading.

The fields themselves are also inconsistent across types (plans carry
`Ticket`, ideas carry `Date`, doc_create carries `Origin`/`Human-validated`,
tech-debt carries `Category`), with no shared core and no validation.

## Design Goals

- Every kb doc carries real, fenced YAML frontmatter with a validated schema.
- All metadata reads/writes go through **one** module — no consumer regex-scans
  prose ever again.
- The exec-plan branch name lives in a `branch:` field; doc-gardening orphan
  detection matches the **exact** ref, eliminating the lossy comparison.
- A shared core field set spans all types; per-type fields are explicit and
  validated by an enum/allow-list, not convention.
- A `lint-kb` CI check fails the build on missing/invalid frontmatter.
- Existing docs are migrated in one shot with body content preserved.

## Design

### Format conventions

- Fenced `---` YAML at the very top of every doc, above the `# Title` H1.
- The `# Title` H1 is **retained** in the body (readable git diffs and GitHub
  rendering); `title:` in frontmatter is the tooling source of truth. The
  exec-plan branch moves out of the H1 — the H1 becomes a normal human title.
- **snake_case** keys, ISO-8601 dates.
- One new `src/reinicorn/frontmatter.py` module owns parse + serialize +
  validate. Every current consumer switches to it.

### Shared core fields (all doc types)

```yaml
type:            exec-plan        # enum: exec-plan|retro|idea|spec|prd|tech-debt|principle
title:           Worktree-aware kb resolution
slug:            worktree-aware-kb-resolution
lifecycle:       active           # enum: active | done | dropped  ← tooling/workflows key off THIS
status:          implemented      # free-form, per-type human nuance
created:         2026-07-19
updated:         2026-07-21        # optional; powers staleness/gardening
author:          Michael Biehl
origin:          human             # human | ai-assisted
human_validated: true
tags:            []                # for the aggregation/mining bet
related:         []                # slugs/paths of linked docs — the cross-doc graph
```

Two lifecycle axes: `lifecycle` is the coarse 3-value axis everything queryable
keys off; `status` keeps each type's fine-grained word verbatim.

Lifecycle mapping (used by the migration and documented for authors):

| Type       | today's status → lifecycle                                      |
| ---------- | --------------------------------------------------------------- |
| exec-plan  | planning, in-progress → active · complete → done · abandoned → dropped |
| idea       | new, exploring → active · promoted → done · dropped → dropped    |
| spec / prd | draft, approved → active · implemented → done                   |
| tech-debt  | open → active · resolved → done · wontfix → dropped              |
| retro      | active → active · final → done                                  |

### Per-type fields

- **exec-plan:** `branch` (exact unsanitized name), `ticket`, `spec` (slug it
  implements), `retro` (slug, once created)
- **retro:** `branch`, `plan` (slug it retrospects)
- **idea:** `promoted_to` (spec slug if it graduated)
- **spec / prd:** `supersedes` / `superseded_by`, `implemented_by` (branch or
  plan slug)
- **tech-debt:** `id` (e.g. `SEC-08`), `category`, `severity`
- **principle:** `id`, `category`

The "Decision Log" and "Surprises & Discoveries" content the aggregation bet
wants stays as **body sections with structured `##` headings** — it is prose,
not scalars. Frontmatter carries only queryable/relational fields (`tags`,
`related`, per-type link fields), keeping the corpus mineable without stuffing
paragraphs into YAML.

### `frontmatter.py` interface

A single module, the only thing that touches the `---` block:

- `read(path) -> (dict, body_str)` — parse frontmatter + return remaining body.
- `write(path, meta: dict, body: str)` — serialize with stable key ordering.
- `validate(meta: dict) -> list[str]` — required-field + enum checks; returns
  errors (empty = valid). Used by both create paths and the CI lint.

Stable key order for serialization: `type, title, slug, lifecycle, status,
created, updated, author, origin, human_validated, <per-type fields>, tags,
related`.

### Consumers to repoint

- `doc_show.py` — read `title` + `status`/`lifecycle` from frontmatter.
- `status.py` — read `lifecycle` (drop prose scan).
- `plan.py` — read/write `branch`, `status`, `lifecycle` via frontmatter (drop
  the `sed`/`splitlines` parsing).
- `doc_create.py` `_provenance`, `idea.py`, `kb_seed.py`, retro create path —
  emit frontmatter instead of the bold-label block.
- `.github/workflows/doc-gardening.yml` — read `branch:` from each active
  plan's frontmatter, then `git ls-remote --heads origin "refs/heads/$branch"`;
  flag as orphaned only on an exact-ref miss.

### Migration

1. Land `frontmatter.py` + a schema reference doc in kb.
2. One-shot migration script: walk every kb doc, heuristically parse the
   existing prose block, emit frontmatter, preserve the body verbatim, convert
   `# Execution Plan: <branch>` → `branch:` field + a generic H1.
3. Switch all create paths to emit frontmatter.
4. Repoint all consumers at `frontmatter.py`.
5. Add the `lint-kb` frontmatter validation check to CI.

### CI enforcement

`lint-kb` runs `frontmatter.validate` over all kb docs: frontmatter present,
required core fields set, `type`/`lifecycle` within their enums, per-type
required fields present. Build fails on any error.

## Open Questions (for contributor review)

These are deliberately left open for reviewer feedback rather than frozen here:

1. **Lifecycle mapping.** Do the coarse `active/done/dropped` buckets and the
   per-type word mappings (esp. `planning`/`in-progress` → `active`, `approved`
   → `active`) match how each doc type is actually used? Any type that needs a
   fourth lifecycle value?
2. **Per-type link fields.** The `spec:`/`retro:` (plan), `promoted_to:`
   (idea), `supersedes:`/`superseded_by:`/`implemented_by:` (spec) fields seed
   the `related` graph but are speculative. Keep all, trim, or defer until a
   consumer needs them?
3. **Migration strategy.** One-shot rewrite of every existing kb doc vs.
   incremental / new-docs-first with a compatibility shim in `frontmatter.py`
   that still reads the legacy prose block during a transition window.
4. **Enum strictness.** Should `type` and `lifecycle` be hard-failed by
   `lint-kb`, or warn-only for a grace period after rollout?

## Non-Goals

- No new doc *types* — only a schema for the existing seven.
- Decision-log / surprises content is **not** moved into frontmatter; it stays
  as body sections (structured headings may be specified separately later).
- No cross-doc aggregation/mining tooling in this change — this only lays the
  `tags`/`related`/link-field substrate it will later consume.
- No change to where docs live on disk or to the review/publish lane; only the
  in-file metadata format changes.
- Status vocabulary per type is not exhaustively frozen here; the migration maps
  today's observed values, and `status` remains free-form (only `lifecycle` is
  enum-validated).
