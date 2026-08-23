---
type: debt
title: kb path literals duplicated outside the doc-type registry
slug: kb-path-literals-duplicated-outside-the-doc-type-registry
lifecycle: active
status: draft
created: 2026-07-09
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: cli
severity: low
remediation: planned
---

# kb path literals duplicated outside the doc-type registry

## Impact

`doc_types.REGISTRY` is the declared single source of truth for kb doc
layout, but several sites re-state fragments of it as literals (found in the
2026-07-09 repo-wide review sweep). All copies agree today, so no live bug —
but a registry change silently strands them:

- The literal `"active"` (first segment of the plan pattern) appears at 6
  sites (`status.py`, `kb.py` ×2, `post_merge.py`, linter rules ×2).
- `"index.md"` hard-coded in `status.py` (×2) and `doc_show.py:44` although
  `DocType.index_file` exists for exactly this; `docs_freshness.py` does it
  right.
- `"plan.md"` / `"retro.md"` basenames re-stated in `plan.py` (×4 sites) and
  `plan_structure.py` instead of `Path(REGISTRY[...].filename).name`.
- `kb_seed.py` inlines `"active"/"completed"`, `"golden-principles.md"`,
  `"plan.md"` despite its docstring promising registry-derived paths.
- `review.collect_gated_drafts` globs a flat `*.md` and assumes slug == stem;
  correct for spec today, silently wrong for any future nested gated pattern.
- The shell linters (`linters/rules/kb/*.sh`) re-encode registry facts the
  Python rules derive (worst: docs-freshness's frozen tech-debt/specs/prds
  index list). Shell can't import Python — the fix pattern is delegation to a
  reins subcommand, as `editor-hooks/enforce-doc-templates.sh` already does
  with `reins _check-path`.

## Remediation Plan

- Add registry accessors: `active_plans_root(repo_dir)`, and use
  `DocType.index_file` / `Path(filename).name` at the listed sites.
- Reuse the `re.sub(r"\{\w+\}", "*", dt.filename)` glob idiom in
  `collect_gated_drafts`.
- Align `kb_seed.py` with its own docstring.
- For shell linters: emit the needed facts from a `reins` subcommand (or a
  generated manifest) instead of hand-listing them in bash.
