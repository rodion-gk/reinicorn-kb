---
type: spec
title: Centralized Doc-Type Registry
slug: centralized-doc-type-registry
lifecycle: done
status: implemented
created: 2026-02-26
author: mnbiehl
origin: ai-assisted
human_validated: false
---

# Centralized Doc-Type Registry

## Problem

Doc-type definitions (directory paths, filename patterns, protection flags, required sections, index files) are hard-coded across 8+ files: `doc_create.py`, `post_merge.py`, `plan.py`, `harness.py`, `status.py`, `idea.py`, and two linter rule files. Adding or renaming a doc type requires hunting down every reference. This is error-prone and makes the system fragile.

## Design Goals

- Single source of truth for all doc-type metadata
- No hard-coded doc-type directory names outside the registry module
- All existing consumers refactored to use registry lookups
- Zero behavior change — purely structural refactor

## Design

### Registry Module: `src/reins/doc_types.py`

A `DocType` dataclass defines each doc type:

```python
@dataclass(frozen=True)
class DocType:
    key: str                        # "design", "plan", "spec", etc.
    dir_path: str                   # Dir relative to repo scope, e.g. "design-docs"
    filename: str                   # Pattern: "{slug}.md", "active/{branch}/plan.md", etc.
    protected: bool                 # Whether direct harness edits are blocked
    index_file: str | None          # For freshness linter, e.g. "index.md"
    required_sections: list[str]    # Linter checks for these headers
    creator: Callable | None        # Set by doc_create.py at import time
```

Module-level `REGISTRY: dict[str, DocType]` maps keys to entries.

Helper functions:
- `get_doc_dir(key, repo_dir) -> Path` — resolves the full directory path
- `get_protected_map() -> dict[str, str]` — returns `{"design-docs": "design", ...}`
- `by_dir(dir_name) -> DocType | None` — reverse lookup by directory name

### Registry Entries

| Key | dir_path | filename | protected | index_file | required_sections |
|-----|----------|----------|-----------|------------|-------------------|
| design | design-docs | {slug}.md | yes | index.md | Problem, Design Goals, Design, Non-Goals |
| plan | exec-plans | active/{branch}/plan.md | yes | - | Goal, Acceptance Criteria, Tasks |
| spec | product-specs | {slug}.md | yes | index.md | Overview, User Stories, Acceptance Criteria, Out of Scope, Open Questions |
| debt | tech-debt | by-domain/{slug}.md | yes | index.md | Impact, Remediation Plan |
| idea | ideas | {username}/{slug}.md | yes | - | - |
| retro | exec-plans | completed/{branch}/retro.md | yes | - | Outcome Assessment, What Worked, What Didn't, Learnings, Follow-ups |
| principle | . | golden-principles.md | no | - | - |

### Circular Import Avoidance

The registry module defines data only. Creator functions remain in `doc_create.py` and register themselves into the corresponding `DocType` entry at import time (the `creator` field defaults to `None` and is set via a `register_creator()` helper).

### Consumer Refactoring

Files to update:
1. **doc_create.py** — `_DOC_TYPES`, `_CREATORS`, `protected` dict replaced by registry
2. **post_merge.py** — `"exec-plans" / "active"` replaced by `get_doc_dir("plan", ...)`
3. **plan.py** — Same exec-plans path references
4. **harness.py** — `plan_dir()` delegates to registry
5. **status.py** — exec-plans and tech-debt path references
6. **idea.py** — ideas path reference
7. **linter/rules/docs_freshness.py** — iterates registry entries with `index_file`
8. **linter/rules/plan_structure.py** — uses `required_sections` from registry

## Non-Goals

- **Template externalization** — Inline markdown templates stay in creator functions. Externalizing to editable files is a separate effort (captured as idea).
- **User-extensible doc types** — The registry is internal Python code only. Per-repo custom doc types via config files may come later.
- **Manual test plans for harness functions** — Separate effort (captured as idea).
