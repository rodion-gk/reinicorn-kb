---
type: plan
title: 'Execution Plan: feat-registry-doc-types-stage1'
slug: feat-registry-doc-types-stage1
lifecycle: done
status: complete
created: 2026-08-15
author: Michael Biehl
branch: feat-registry-doc-types-stage1
ticket: N/A
spec: reinicorn/specs/registry-driven-doc-types.md
---

# Execution Plan: feat-registry-doc-types-stage1

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stage 1 of `specs/registry-driven-doc-types.md` — add the new `DocType` fields, replace the five per-type creators (plus `idea.py`'s bespoke creator) with one generic `_create_doc`, and turn the gated⇒slug invariant into a load-time `ValueError`.

**Architecture:** All creation behavior moves into `doc_types.REGISTRY` rows; `doc_create.py` gains one rendering path (`render_doc`) and one creation path (`_create_doc` behind `cmd_doc_create`) that dispatch on `addressing` / `title_source` / `create_mode`. `plan.py` keeps its lifecycle logic but its fallback file-write routes through `render_doc`.

**Tech Stack:** Python 3.12+, uv, pytest, ruff, pyright. Frozen dataclasses; `frontmatter.render` remains the single validating serializer.

## Goal

Ship spec stage 1 ("Fields + generic creator") as one PR: registry rows become the single source of doc-creation behavior for all seven existing types, with byte-preserved templates and paths, and the latent gated⇒slug crash becomes a load-time validation error.

Stages 2–5 of the spec (CLI generation, gate generalization, seeding/linting, shipped docs/tests) are **out of scope** — each gets its own branch + plan after this merges.

**Branch flow (decided 2026-08-16):** the whole feature integrates on the long-lived branch `feat-registry-doc-types`; each stage branch PRs into it, and only the completed feature merges to `main`. No intermediary stage lands on `main`.

## Acceptance Criteria

- [ ] `DocType` carries `template_body`, `addressing`, `title_source`, `create_verb`, `create_mode`, `help_text`, `readme_label` (spec's seven) plus `create_status` and `extra_meta` (approved planning addition — see Approach).
- [ ] `title_required` is gone, subsumed by `title_source`.
- [ ] A gated registry row with non-slug addressing raises `ValueError` at import time (not `assert` — must fire under `python -O`), with a what/where/how-to-fix message (golden principle 4).
- [ ] `_create_spec`, `_create_prd`, `_create_debt`, `_create_retro`, `_create_principle`, `_CREATORS`, `_create_typed`, the five `cmd_*_create/add` wrappers, and `src/reinicorn/commands/idea.py` are deleted; one `cmd_doc_create(doc_type, title)` serves all seven types and `_DISPATCH` calls it.
- [ ] `plan.py`'s template-less fallback writes `plan.md` through the shared renderer; `test_plan_create_three_template_states_agree` still passes.
- [ ] Behavior preserved: creation paths, template bodies, gating semantics, no-clobber and drafts-annex rules, retro rides-with-plan, idea `-2` dedupe. The only diffs are the four normalizations listed in Approach.
- [ ] Full gate green: `uv run pytest tests/ -v`, `uv run ruff check src/reinicorn tests`, `uv run pyright src/reinicorn`, `bash tests/run-all.sh`.
- [ ] PR to the `feat-registry-doc-types` integration branch citing the spec, with verification evidence and the normalizations disclosed (never push to main; main only receives the completed feature).

## Approach

Key decisions, settled during planning:

1. **Two registry fields beyond the spec's seven** (approved by Michael 2026-08-15): `create_status: str = "draft"` (idea=`new`, principle=`active`, plan=`planning`) and `extra_meta: tuple[tuple[str, str], ...] = ()` (debt's static `severity`/`category`/`remediation`). Rationale: serves the spec's "one registry row = one complete doc type" goal; a frozen dataclass needs the tuple-of-pairs form.
2. **Idea provenance normalization** (the drift fix the spec calls for): idea create now prints `Created: <path>` (was `Idea captured:`), gains the `next: rcorn kb publish` hint, and commits as `doc(idea): <slug>` (was `idea: <slug>`, and the slug now includes the `-2` dedupe suffix when applied). Files, paths, dedupe, `status: new`, and frontmatter shape are unchanged.
3. **Plan frontmatter gains `origin` + `human_validated`** in both the template path and the fallback path. Forced by routing the fallback through the shared renderer while keeping the `test_plan_create_three_template_states_agree` invariant (all creation paths emit the same key set). Both are core fields allowed on every type.
4. **Retro's branch-heading fallback generalizes** to `f"{dt.key.capitalize()}: {branch}"` — byte-identical for retro (`Retro: feature/x`), no per-type literal.

What deliberately stays bespoke (spec Non-Goals): plan lifecycle verbs and overlap detection; retro-rides-with-active-plan target resolution (kept as an `addressing == "branch"` special case keyed on `dt is REGISTRY["retro"]` — an identity check against the registry row, because `tests/test_source_of_truth.py::test_no_doc_type_key_comparisons` forbids doc-type key strings in comparisons); the `"Golden Principles"` singleton H1 literal (stage 3's literal sweep).

Task order keeps the suite green at every commit: registry first (with `title_required` migrated in the same commit), then the generic creator with the old wrappers as one-line delegates, then idea, then wrapper/dispatch collapse, then plan fallback, then the phantom-type test and full gate.

## Global Constraints

- Python 3.12+; run tests `uv run pytest tests/ -v`; lint `uv run ruff check src/reinicorn tests`; types `uv run pyright src/reinicorn`; full runner `bash tests/run-all.sh` (AGENTS.md).
- Never `uv run rcorn`; the installed binary is the CLI surface. No CLI invocation is needed by this plan's tests.
- Red-green TDD; conventional commits; stdout is the agent-facing result surface.
- Error messages state what/where/how-to-fix (golden principle 4).
- Never push to `main` — stage PRs target the `feat-registry-doc-types` integration branch; only the finished feature PRs to `main`.

## Tasks

### Task 1: Registry fields + load-time validation

**Files:**
- Modify: `src/reinicorn/doc_types.py` (full rewrite of `DocType` and `REGISTRY`)
- Modify: `src/reinicorn/commands/doc_create.py:193` (the one `title_required` use)
- Test: `tests/test_doc_types.py`

**Interfaces:**
- Consumes: nothing new.
- Produces: `DocType` with fields exactly as written below (later tasks read `template_body`, `addressing`, `title_source`, `create_verb`, `create_mode`, `create_status`, `extra_meta`, `help_text`, `readme_label`); module-level `_validate_registry() -> None` raising `ValueError`.

- [ ] **Step 1: Write the failing tests** — append to `tests/test_doc_types.py`:

```python
def test_addressing_values():
    assert REGISTRY["spec"].addressing == "slug"
    assert REGISTRY["prd"].addressing == "slug"
    assert REGISTRY["debt"].addressing == "slug"
    assert REGISTRY["idea"].addressing == "slug"
    assert REGISTRY["plan"].addressing == "branch"
    assert REGISTRY["retro"].addressing == "branch"
    assert REGISTRY["principle"].addressing == "singleton"


def test_title_source_values():
    assert REGISTRY["idea"].title_source == "free_text"
    assert REGISTRY["plan"].title_source == "none"
    assert REGISTRY["retro"].title_source == "none"
    for key in ("spec", "prd", "debt", "principle"):
        assert REGISTRY[key].title_source == "title"


def test_principle_append_mode():
    p = REGISTRY["principle"]
    assert p.create_verb == "add"
    assert p.create_mode == "append"
    assert p.create_status == "active"


def test_create_status_values():
    assert REGISTRY["idea"].create_status == "new"
    assert REGISTRY["plan"].create_status == "planning"
    for key in ("spec", "prd", "debt", "retro"):
        assert REGISTRY[key].create_status == "draft"


def test_debt_extra_meta():
    assert dict(REGISTRY["debt"].extra_meta) == {
        "severity": "medium", "category": "_domain_", "remediation": "planned",
    }
    for key in ("spec", "prd", "idea", "plan", "retro", "principle"):
        assert REGISTRY[key].extra_meta == ()


def test_readme_labels():
    assert REGISTRY["spec"].readme_label == "Approved specs"
    assert REGISTRY["prd"].readme_label == "Product requirements"
    assert REGISTRY["plan"].readme_label == "Active plans"
    assert REGISTRY["debt"].readme_label == "Technical debt"
    assert REGISTRY["principle"].readme_label == "Golden principles"
    assert REGISTRY["idea"].readme_label is None
    assert REGISTRY["retro"].readme_label is None


def test_registry_invariant_required_fields_nonempty():
    for dt in REGISTRY.values():
        assert dt.help_text, dt.key
        assert dt.create_verb, dt.key
        assert dt.create_status, dt.key
        assert dt.create_hint, dt.key


def test_gated_implies_slug_addressing():
    for dt in gated_types():
        assert dt.addressing == "slug"


def test_registry_rejects_gated_non_slug_row():
    import pytest
    from unittest.mock import patch
    from reinicorn.doc_types import _validate_registry
    bad = DocType(
        key="phantom", dir_path="phantoms", filename="active/{branch}/doc.md",
        protected=True, create_hint="rcorn phantom create",
        help_text="Phantom ops", template_body="",
        addressing="branch", gated=True,
    )
    with patch.dict(REGISTRY, {"phantom": bad}):
        with pytest.raises(ValueError, match="phantom"):
            _validate_registry()
```

- [ ] **Step 2: Run to verify they fail**

Run: `uv run pytest tests/test_doc_types.py -v`
Expected: FAIL — `TypeError: DocType.__init__() got an unexpected keyword argument 'help_text'` (and collection-level `AttributeError` for `_validate_registry`).

- [ ] **Step 3: Rewrite `src/reinicorn/doc_types.py`** — replace the `DocType` class and `REGISTRY` (module docstring, helper functions `get_doc_dir`/`get_protected_map`/`by_dir`/`drafts_dir`/`gated_types`, and `DRAFTS_DIR_NAME` stay as-is):

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pathlib import Path
    from typing import Literal


@dataclass(frozen=True)
class DocType:
    """Metadata for a single kb document type."""

    key: str
    dir_path: str  # Relative to repo-scoped dir (e.g. "specs")
    filename: str  # Pattern: "{slug}.md", "active/{branch}/plan.md", etc.
    protected: bool  # Whether direct kb edits are blocked
    create_hint: str  # Exact CLI command that creates docs of this type
    help_text: str  # CLI group help (hand-written in cli.py until stage 2)
    # Creation body appended after the frontmatter + H1. Placeholders:
    # {title} {author} {date} {sections} {text} {num} — formatted with a
    # superset of params, so a body only names the ones it needs.
    template_body: str
    addressing: Literal["slug", "branch", "singleton"]
    title_source: Literal["title", "free_text", "none"] = "title"
    create_verb: str = "create"
    create_mode: Literal["file", "append"] = "file"
    create_status: str = "draft"  # frontmatter `status:` a new doc opens with
    # Static per-type frontmatter, as (key, value) pairs (frozen-friendly).
    extra_meta: tuple[tuple[str, str], ...] = ()
    readme_label: str | None = None  # Seeded kb README row; None = no row
    index_file: str | None = None  # For freshness linter
    required_sections: tuple[str, ...] = ()  # Linter checks these headers
    gated: bool = False  # Review-gated: create writes to drafts/, approval via the review lane


REGISTRY: dict[str, DocType] = {
    "spec": DocType(
        key="spec",
        dir_path="specs",
        filename="{slug}.md",
        protected=True,
        create_hint='rcorn spec create "<title>"',
        help_text="Spec doc operations (the implementation contract)",
        template_body=(
            "\n## Problem\n\n_Describe the problem._\n"
            "\n## Design Goals\n\n_What must be true when this is done._\n"
            "\n## Design\n\n_How it works._\n"
            "\n## Non-Goals\n\n_What this explicitly does not cover._\n"
        ),
        addressing="slug",
        readme_label="Approved specs",
        index_file="index.md",
        required_sections=("Problem", "Design Goals", "Design", "Non-Goals"),
        gated=True,
    ),
    "plan": DocType(
        key="plan",
        dir_path="exec-plans",
        filename="active/{branch}/plan.md",
        protected=True,
        create_hint="rcorn plan create",
        help_text="Execution plan operations",
        template_body="",  # fallback plan.md is frontmatter + H1 only
        addressing="branch",
        title_source="none",
        create_status="planning",
        readme_label="Active plans",
        required_sections=("Goal", "Acceptance Criteria", "Tasks"),
    ),
    "prd": DocType(
        key="prd",
        dir_path="prds",
        filename="{slug}.md",
        protected=True,
        create_hint='rcorn prd create "<title>"',
        help_text="Product requirements doc operations",
        template_body=(
            "\n## Overview\n\n_One-paragraph summary._\n"
            "\n## User Stories\n\n- As a [role], I want [goal] so that [benefit].\n"
            "\n## Acceptance Criteria\n\n- [ ] _Criterion 1_\n"
            "\n## Out of Scope\n\n_What this PRD explicitly does not cover._\n"
            "\n## Open Questions\n\n_Unresolved decisions._\n"
        ),
        addressing="slug",
        readme_label="Product requirements",
        index_file="index.md",
        required_sections=(
            "Overview",
            "User Stories",
            "Acceptance Criteria",
            "Out of Scope",
            "Open Questions",
        ),
    ),
    "debt": DocType(
        key="debt",
        dir_path="tech-debt",
        filename="{slug}.md",
        protected=True,
        create_hint='rcorn debt create "<title>"',
        help_text="Tech debt doc operations",
        template_body=(
            "\n## Impact\n\n_What this debt causes._\n"
            "\n## Remediation Plan\n\n_How to fix it._\n"
        ),
        addressing="slug",
        extra_meta=(
            ("severity", "medium"),
            ("category", "_domain_"),
            ("remediation", "planned"),
        ),
        readme_label="Technical debt",
        index_file="index.md",
        required_sections=("Impact", "Remediation Plan"),
    ),
    "idea": DocType(
        key="idea",
        dir_path="ideas",
        filename="{username}/{slug}.md",
        protected=True,
        create_hint='rcorn idea create "<idea>"',
        help_text="Idea capture",
        template_body=(
            "\n## Description\n\n{text}\n"
            "\n## Notes\n\n_No additional notes yet._\n"
        ),
        addressing="slug",
        title_source="free_text",
        create_status="new",
    ),
    "retro": DocType(
        key="retro",
        dir_path="exec-plans",
        filename="completed/{branch}/retro.md",
        protected=True,
        create_hint="rcorn retro create",
        help_text="Retrospective operations",
        template_body="{sections}",
        addressing="branch",
        title_source="none",
        required_sections=(
            "What Went Well",
            "What Could Be Improved",
            "Lessons Learned",
            "Action Items",
        ),
    ),
    "principle": DocType(
        key="principle",
        dir_path=".",
        filename="golden-principles.md",
        protected=False,
        create_hint='rcorn principle add "<title>"',
        help_text="Golden principle operations",
        template_body=(
            "\n\n{num}. **{title}**\n"
            "   - _Rule description_\n"
            "   - Prevents: _What this rule prevents_\n"
        ),
        addressing="singleton",
        create_verb="add",
        create_mode="append",
        create_status="active",
        readme_label="Golden principles",
    ),
}


def _validate_registry() -> None:
    """Load-time invariants over REGISTRY rows.

    A plain raise, not `assert`: it must fire under `python -O` too, so an
    invalid row can never load (spec: registry-driven-doc-types).
    """
    for dt in REGISTRY.values():
        if dt.gated and dt.addressing != "slug":
            raise ValueError(
                f"doc_types.REGISTRY['{dt.key}']: gated=True requires "
                f"addressing='slug', got '{dt.addressing}' — the review lane "
                "derives candidate paths from slugs (review.make_target). "
                "Fix the row in src/reinicorn/doc_types.py."
            )


_validate_registry()
```

`title_required` is deleted. In the same commit, migrate its one use — `src/reinicorn/commands/doc_create.py:193`:

```python
    if REGISTRY[doc_type].title_source == "title" and not title.strip():
```

(`retro` was `title_required=False`, now `title_source="none"` — same behavior.)

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/test_doc_types.py tests/commands/test_doc_create.py -v`
Expected: PASS (all).

- [ ] **Step 5: Full suite + lint + types**

Run: `uv run pytest tests/ -q && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn`
Expected: all green.

- [ ] **Step 6: Commit**

```bash
git add src/reinicorn/doc_types.py src/reinicorn/commands/doc_create.py tests/test_doc_types.py
git commit -m "refactor(doc-types): add creation fields and load-time registry validation"
```

### Task 2: Shared renderer + generic creator

**Files:**
- Modify: `src/reinicorn/commands/doc_create.py` (delete `_create_spec`, `_create_prd`, `_create_debt`, `_create_retro`, `_create_principle`, `_CREATORS`, `_create_typed`; add `render_doc`, `_branch_target`, `_append_doc`, `_create_doc`, `cmd_doc_create`; keep the five `cmd_*` names as one-line delegates)
- Test: `tests/commands/test_doc_create.py`

**Interfaces:**
- Consumes: Task 1's `DocType` fields.
- Produces:
  - `render_doc(dt: DocType, title: str, author: str, *, extra: dict[str, object] | None = None, body_params: dict[str, str] | None = None) -> str` — public: Task 5 (plan.py) imports it.
  - `cmd_doc_create(doc_type: str, title: str = "") -> int` — Tasks 3–4 dispatch to it.
  - `_create_doc(dt: DocType, repo_dir: Path, title: str, author: str) -> Path` (raises `FileExistsError` on slug collision for title-addressed types).

- [ ] **Step 1: Write the failing tests** — in `tests/commands/test_doc_create.py`, replace the two `_create_typed` tests and add generic-entry tests:

```python
def test_doc_create_unknown_type_returns_error():
    """cmd_doc_create must guard against unknown doc types."""
    from reinicorn.commands.doc_create import cmd_doc_create
    assert cmd_doc_create("nonexistent", "some title") == 1


def test_doc_create_empty_title_rejected_for_title_types():
    from reinicorn.commands.doc_create import cmd_doc_create
    assert cmd_doc_create("spec", "") == 1
    assert cmd_doc_create("spec", "   ") == 1


def test_debt_create_carries_static_extra_meta(kb_repo: Path):
    """Debt's severity/category/remediation now come from REGISTRY.extra_meta."""
    from reinicorn.commands.doc_create import cmd_doc_create
    p1, p2, p3, p4 = _create_env(kb_repo)
    with p1, p2, p3, p4:
        assert cmd_doc_create("debt", "Old coupling") == 0
    doc = kb_repo / "kb" / "testproject" / "tech-debt" / "old-coupling.md"
    text = doc.read_text()
    assert fm.get(text, "severity") == "medium"
    assert fm.get(text, "category") == "_domain_"
    assert fm.get(text, "remediation") == "planned"


def test_principle_add_appends_numbered_item(kb_repo: Path):
    """Second add appends item 2 to the singleton file, no new file."""
    from reinicorn.commands.doc_create import cmd_doc_create
    p1, p2, p3, p4 = _create_env(kb_repo)
    with p1, p2, p3, p4:
        assert cmd_doc_create("principle", "First rule") == 0
        assert cmd_doc_create("principle", "Second rule") == 0
    doc = kb_repo / "kb" / "testproject" / "golden-principles.md"
    content = doc.read_text()
    assert "1. **First rule**" in content
    assert "2. **Second rule**" in content
    assert fm.get(content, "status") == "active"
```

Delete `test_create_typed_unknown_type_returns_error` and `test_create_typed_empty_title_rejected_for_non_retro` (replaced above).

- [ ] **Step 2: Run to verify the new tests fail**

Run: `uv run pytest tests/commands/test_doc_create.py -v`
Expected: FAIL — `ImportError: cannot import name 'cmd_doc_create'`.

- [ ] **Step 3: Implement in `src/reinicorn/commands/doc_create.py`.** Keep `_get_author`, `_slugify`, `_provenance`, `_typed_dir`, `_slug_target`, `cmd_doc_check_path` unchanged. Delete `_create_spec`, `_create_prd`, `_create_debt`, `_create_retro`, `_create_principle`, `_CREATORS`, `_create_typed`. Add:

```python
def render_doc(
    dt: DocType, title: str, author: str, *,
    extra: dict[str, object] | None = None,
    body_params: dict[str, str] | None = None,
) -> str:
    """Frontmatter + H1 + the type's template body — the one rendering path
    every doc-type creation goes through (registry-driven-doc-types stage 1)."""
    sections = "".join(f"\n## {s}\n\n- \n" for s in dt.required_sections)
    params: dict[str, str] = {
        "title": title, "author": author,
        "date": date.today().isoformat(), "sections": sections,
    }
    params.update(body_params or {})
    merged: dict[str, object] = dict(dt.extra_meta)
    merged.update(extra or {})
    return _provenance(
        title, author, status=dt.create_status, doc_type=dt.key, extra=merged,
    ) + dt.template_body.format(**params)


def _branch_target(dt: DocType, repo_dir: Path, branch: str) -> Path:
    """Branch-addressed target. Retro rides with an active plan when one
    exists (spec non-goal: this coupling stays code, not registry data)."""
    if dt is REGISTRY["retro"]:  # identity, not key string: see test_source_of_truth
        active_dir = branch_doc_path("plan", repo_dir, branch).parent
        if active_dir.is_dir():
            return active_dir / Path(dt.filename).name
    return branch_doc_path(dt.key, repo_dir, branch)


def _append_doc(dt: DocType, repo_dir: Path, title: str, author: str) -> Path:
    """create_mode="append": add one templated item to the singleton file."""
    target = repo_dir / dt.filename
    if not target.is_file():
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(_provenance(
            "Golden Principles", author or "unknown",
            status=dt.create_status, doc_type=dt.key,
            extra={"slug": target.stem},
        ) + "\n")
    content = target.read_text()
    num = len(re.findall(r"^\d+\.", content, re.MULTILINE)) + 1
    target.write_text(
        content.rstrip() + dt.template_body.format(num=num, title=title)
    )
    return target


def _create_doc(dt: DocType, repo_dir: Path, title: str, author: str) -> Path:
    """Create (or append) one doc from its registry row."""
    if dt.create_mode == "append":
        return _append_doc(dt, repo_dir, title, author)
    if dt.addressing == "branch":
        branch = current_branch() or "unknown"
        target = _branch_target(dt, repo_dir, branch)
        target.parent.mkdir(parents=True, exist_ok=True)
        heading = title.strip() or f"{dt.key.capitalize()}: {branch}"
        target.write_text(render_doc(
            dt, heading, author,
            extra={"branch": branch, "slug": branch_dir_name(branch)},
        ))
        return target
    if dt.title_source == "free_text":
        username = re.sub(r"[^a-z0-9-]", "", author.lower().replace(" ", "-"))
        slug = _slugify(title)
        target = get_doc_dir(dt.key, repo_dir) / dt.filename.format(
            slug=slug, username=username,
        )
        target.parent.mkdir(parents=True, exist_ok=True)
        if target.exists():
            # Derived slugs collide silently (the user never chose one), so
            # suffix instead of erroring like title-addressed creates do.
            target = target.with_stem(f"{slug}-2")
        heading = title.split("\n")[0][:80]
        target.write_text(render_doc(
            dt, heading, author,
            extra={"slug": target.stem}, body_params={"text": title},
        ))
        return target
    slug = _slugify(title)
    target = _slug_target(dt.key, repo_dir, slug)
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(render_doc(dt, title, author))
    return target


def cmd_doc_create(doc_type: str, title: str = "") -> int:
    """Create a kb doc of any registry type — the one generic entry point."""
    dt = REGISTRY.get(doc_type)
    if dt is None:
        console.error(f"Unknown doc type '{doc_type}'.")
        return 1

    if dt.title_source == "title" and not title.strip():
        console.error("Title is required.")
        return 1
    if dt.title_source == "free_text" and not title.strip():
        console.error(
            f'Usage: rcorn {dt.key} {dt.create_verb} "your {dt.key} here"'
        )
        return 1

    root = repo_root()
    if root is None:
        return 1
    kb_dir = require_kb_dir(root)

    repo_dir = kb_dir / kb_scope(root)
    repo_dir.mkdir(parents=True, exist_ok=True)

    author = _get_author()
    try:
        filepath = _create_doc(dt, repo_dir, title, author)
    except FileExistsError as e:
        console.error(str(e))
        return 1

    console.success(f"Created: {filepath}")
    # Branch-addressed docs derive identity from the branch (encoded in the
    # path); free-text docs from the deduped filename; the rest from the title.
    if dt.addressing == "branch":
        slug = filepath.parent.name
    elif dt.title_source == "free_text":
        slug = filepath.stem
    else:
        slug = _slugify(title)
    if dt.gated:
        console.next_step(f"rcorn review start {slug}")
    commit_kb(root, f"doc({dt.key}): {slug}", paths=[filepath])
    console.next_step("rcorn kb publish")
    return 0
```

The five public names become one-line delegates (deleted in Task 4):

```python
def cmd_spec_create(title: str) -> int:
    return cmd_doc_create("spec", title)


def cmd_prd_create(title: str) -> int:
    return cmd_doc_create("prd", title)


def cmd_debt_create(title: str) -> int:
    return cmd_doc_create("debt", title)


def cmd_retro_create() -> int:
    return cmd_doc_create("retro", "")


def cmd_principle_add(title: str) -> int:
    return cmd_doc_create("principle", title)
```

Import changes at the top of the file: add `DocType` to the `reinicorn.doc_types` import.

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/commands/test_doc_create.py tests/commands/test_retro.py tests/test_born_passing.py -v`
Expected: PASS — behavior parity: drafts annex for spec, no-clobber errors, retro heading `# Retro: feature/x`, retro commit `doc(retro): feature-x`, principle numbering.

- [ ] **Step 5: Full suite**

Run: `uv run pytest tests/ -q`
Expected: PASS (test_idea.py still exercises the old `idea.py` — untouched until Task 3).

- [ ] **Step 6: Commit**

```bash
git add src/reinicorn/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "refactor(doc-create): one generic creator replaces the five per-type creators"
```

### Task 3: Idea through the generic creator

**Files:**
- Delete: `src/reinicorn/commands/idea.py`
- Modify: `src/reinicorn/cli.py:246` (the `("idea", "create")` dispatch row)
- Test: `tests/commands/test_idea.py` (migrate), `tests/test_born_passing.py:22,50` (migrate import + call)

**Interfaces:**
- Consumes: `cmd_doc_create` from Task 2.
- Produces: `("idea", "create")` dispatches to `_load("doc_create", "cmd_doc_create")("idea", ...)`. No `reinicorn.commands.idea` module exists afterward.

- [ ] **Step 1: Migrate `tests/commands/test_idea.py` first.** Replace the import and every patch target/call:

```python
from reinicorn.commands.doc_create import cmd_doc_create
```

- Patch targets change from `reinicorn.commands.idea.<name>` to `reinicorn.commands.doc_create.<name>` (`repo_root`, `run_git`, `commit_kb`, `kb_scope`).
- Calls change from `cmd_idea(text)` to `cmd_doc_create("idea", text)`.
- Keep every assertion as-is: they pin the preserved behavior — `ideas/test-user/` dir, `status: new` frontmatter, commit scoped to the created file with `my-cool-idea-for-testing` in the message, `-2` dedupe filename. (The commit message now reads `doc(idea): my-cool-idea-for-testing` — the existing `in`-assertion already accepts it.)

In `tests/test_born_passing.py`: delete line 22 (`from reinicorn.commands.idea import cmd_idea`), change line 50 to `assert cmd_doc_create("idea", "Born idea") == 0`, importing `cmd_doc_create` alongside the existing `doc_create` imports at lines 15-21.

- [ ] **Step 2: Run to verify current state**

Run: `uv run pytest tests/commands/test_idea.py tests/test_born_passing.py -v`
Expected: PASS already (Task 2 made `cmd_doc_create("idea", ...)` work) — this step verifies the migration is faithful; the deletion below proves nothing else depended on `idea.py`.

- [ ] **Step 3: Delete the module and rewire dispatch**

```bash
git rm src/reinicorn/commands/idea.py
```

In `src/reinicorn/cli.py`, change the dispatch row:

```python
    ("idea", "create"): lambda a: _load("doc_create", "cmd_doc_create")(
        "idea", " ".join(a.text)
    ),
```

- [ ] **Step 4: Run the full suite**

Run: `uv run pytest tests/ -q && uv run ruff check src/reinicorn tests`
Expected: PASS; ruff clean (no dangling imports of `idea`).

- [ ] **Step 5: Commit**

```bash
git add -A src/reinicorn tests
git commit -m "refactor(idea): route idea capture through the generic creator"
```

### Task 4: Collapse the per-type wrappers into dispatch

**Files:**
- Modify: `src/reinicorn/commands/doc_create.py` (delete the five delegates)
- Modify: `src/reinicorn/cli.py` (`_DISPATCH` rows for spec/prd/debt/retro/principle create)
- Test: `tests/commands/test_doc_create.py`, `tests/commands/test_retro.py:7`, `tests/test_born_passing.py:16-20,46-52`

**Interfaces:**
- Consumes: `cmd_doc_create` from Task 2.
- Produces: `cmd_doc_create` is the only creation entry point in `doc_create.py`; `_DISPATCH` create rows all call it.

- [ ] **Step 1: Migrate tests.** Replace every wrapper import/call:
  - `tests/commands/test_doc_create.py`: `cmd_spec_create("X")` → `cmd_doc_create("spec", "X")`; same for prd/debt; `cmd_retro_create()` → `cmd_doc_create("retro", "")`; `cmd_principle_add("X")` → `cmd_doc_create("principle", "X")`. Patch targets are already `reinicorn.commands.doc_create.*` — unchanged.
  - `tests/commands/test_retro.py`: import `cmd_doc_create`; calls become `cmd_doc_create("retro", "")`.
  - `tests/test_born_passing.py`: import only `cmd_doc_create` from `doc_create`; calls become `cmd_doc_create("spec", "Born spec")`, `("prd", "Born prd")`, `("debt", "Born debt")`, `("principle", "Born principle")`, `("retro", "")`.

- [ ] **Step 2: Delete the five delegates** from `doc_create.py`, and rewire `_DISPATCH` in `cli.py`:

```python
    ("spec", "create"): lambda a: _load("doc_create", "cmd_doc_create")(
        "spec", " ".join(a.title)
    ),
    ("prd", "create"): lambda a: _load("doc_create", "cmd_doc_create")(
        "prd", " ".join(a.title)
    ),
    ("debt", "create"): lambda a: _load("doc_create", "cmd_doc_create")(
        "debt", " ".join(a.title)
    ),
    ("retro", "create"): lambda _: _load("doc_create", "cmd_doc_create")("retro"),
    ("principle", "add"): lambda a: _load("doc_create", "cmd_doc_create")(
        "principle", " ".join(a.title)
    ),
```

- [ ] **Step 3: Run the full suite + dispatch coverage**

Run: `uv run pytest tests/ -q`
Expected: PASS, including `test_every_parser_verb_has_a_dispatch_entry` (parser shape untouched — stage 2 generates it).

- [ ] **Step 4: Commit**

```bash
git add src/reinicorn/commands/doc_create.py src/reinicorn/cli.py tests
git commit -m "refactor(cli): dispatch doc creation through cmd_doc_create"
```

### Task 5: Plan fallback through the shared renderer

**Files:**
- Modify: `src/reinicorn/commands/plan.py:104-137` (template-path meta + fallback write)
- Test: `tests/commands/test_plan.py`

**Interfaces:**
- Consumes: `render_doc` from Task 2; `REGISTRY["plan"].create_status`.
- Produces: every plan-creation path (template, stale-template, fallback) emits the same frontmatter key set, now including `origin` + `human_validated`.

- [ ] **Step 1: Write the failing test** — append to `tests/commands/test_plan.py`:

```python
def test_plan_create_carries_provenance_fields(kb_repo: Path, capsys):
    """All plan-creation paths emit origin/human_validated (stage-1
    normalization: the fallback routes through the shared renderer, and the
    three-states-agree invariant pulls the template path along)."""
    tmpl_dir = kb_repo / "kb" / "testproject" / "exec-plans" / "_template"
    shutil.rmtree(tmpl_dir)
    fallback = _create_plan(kb_repo, "feature/prov-fallback")
    assert fallback["origin"] == fm.ORIGIN_AI
    assert fallback["human_validated"] is False

    generate_seed_tree(kb_repo / "kb", "testproject")
    seeded = _create_plan(kb_repo, "feature/prov-seeded")
    assert seeded["origin"] == fm.ORIGIN_AI
    assert seeded["human_validated"] is False
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/commands/test_plan.py -v`
Expected: FAIL with `KeyError: 'origin'`.

- [ ] **Step 3: Implement in `src/reinicorn/commands/plan.py`.** Add to the template-path `meta.update({...})` (after `"author": author,`):

```python
                "origin": frontmatter.ORIGIN_AI,
                "human_validated": False,
```

Replace the fallback `else:` branch body (keep the `console.success` line):

```python
    else:
        from reinicorn.commands.doc_create import render_doc
        (pdir / "plan.md").write_text(render_doc(
            REGISTRY["plan"], f"Execution Plan: {branch}", author,
            extra={
                "slug": pdir.name,
                "branch": branch,
                "ticket": ticket_id or "N/A",
                "spec": SPEC_PLACEHOLDER,
            },
        ))
        console.success("Created minimal plan.md (no templates found).")
```

(Function-local import matches the lazy `_load` convention — no command-to-command coupling at module load.) `status: planning` now comes from `REGISTRY["plan"].create_status`; the H1 body is identical to the old literal (`\n# Execution Plan: {branch}\n`).

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/commands/test_plan.py -v`
Expected: PASS — including `test_plan_create_three_template_states_agree` (all three states now share the enlarged key set).

- [ ] **Step 5: Full suite + commit**

Run: `uv run pytest tests/ -q`

```bash
git add src/reinicorn/commands/plan.py tests/commands/test_plan.py
git commit -m "refactor(plan): route fallback plan creation through the shared renderer"
```

### Task 6: Phantom-type creation test + full gate

**Files:**
- Test: `tests/commands/test_doc_create.py` (append)

**Interfaces:**
- Consumes: `cmd_doc_create`, `DocType`, `REGISTRY`.
- Produces: the stage-1 slice of the spec's "phantom type" test (creation coverage; CLI-surface and lint slices land with stages 2 and 4).

- [ ] **Step 1: Write the test**

```python
def test_phantom_type_creates_with_no_other_change(kb_repo: Path):
    """Spec's executable design goal, stage-1 slice: a synthetic registry row
    gets working creation from the row alone."""
    from reinicorn.commands.doc_create import cmd_doc_create
    from reinicorn.doc_types import REGISTRY, DocType
    phantom = DocType(
        key="phantom", dir_path="phantoms", filename="{slug}.md",
        protected=True, create_hint='rcorn phantom create "<title>"',
        help_text="Phantom doc operations",
        template_body="\n## Body\n\n_Filled by {author} on {date}._\n",
        addressing="slug",
    )
    p1, p2, p3, p4 = _create_env(kb_repo)
    with patch.dict(REGISTRY, {"phantom": phantom}), p1, p2, p3, p4:
        assert cmd_doc_create("phantom", "A Test Doc") == 0
    doc = kb_repo / "kb" / "testproject" / "phantoms" / "a-test-doc.md"
    assert doc.is_file()
    text = doc.read_text()
    assert fm.get(text, "type") == "phantom"
    assert "## Body" in text
    assert "Filled by Test User on" in text
```

Note: `frontmatter.render` validates via `_allowed_keys(doc_type)` — `PER_TYPE.get` returns `()` for unknown types and core fields are always allowed, so a phantom row renders fine. If `validate` ever rejects unknown `type:` values, this test is the tripwire.

- [ ] **Step 2: Run it**

Run: `uv run pytest tests/commands/test_doc_create.py::test_phantom_type_creates_with_no_other_change -v`
Expected: PASS immediately — it's an invariant pin, not red-green; if it fails, the generic creator has a hidden per-type dependency to fix.

- [ ] **Step 3: Run the complete gate**

Run: `uv run pytest tests/ -v && uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && bash tests/run-all.sh`
Expected: all green. Record the pytest pass count for the PR body.

- [ ] **Step 4: Commit**

```bash
git add tests/commands/test_doc_create.py
git commit -m "test(doc-create): phantom-type creation coverage for the registry contract"
```

- [ ] **Step 5: Open the PR** against `feat-registry-doc-types` (never push to main). Body must cite `kb/reinicorn/specs/registry-driven-doc-types.md` (stage 1 of 5), state the verification evidence (test count, full gate), declare scope boundaries (stages 2–5 follow-up branches), and disclose the four normalizations from Approach as intentional behavior diffs.

## Dependencies

- Spec: `reinicorn/specs/registry-driven-doc-types.md` (approved, PR #10). This branch is stage 1 of its 5-stage sequence; stages 2–5 each branch off `feat-registry-doc-types` after their predecessor merges into it, and get their own plan. The integration branch merges to `main` only when stage 5 lands.
- Interacts with nothing on other active branches (`rcorn plan create` overlap check: none).
- After merge: reinstall the CLI (`uv tool install --force .`) per AGENTS.md.
