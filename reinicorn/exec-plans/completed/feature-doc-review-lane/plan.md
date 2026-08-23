---
type: plan
title: 'Execution Plan: feature/doc-review-lane'
slug: feature-doc-review-lane
lifecycle: done
status: complete
created: 2026-07-06
author: Michael Biehl
branch: feature/doc-review-lane
ticket: N/A
spec: kb/reins/specs/doc-review-lane-pr-style-review-for-gated-kb-docs.md
---

# Execution Plan: feature/doc-review-lane

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Goal

PR-style review for gated kb docs: drafts live at `specs/drafts/` on kb main, review happens on a server-side add-only PR (full-file diff, inline comments), merge lands the approved doc at the canonical path, and reins/CI clean up the draft. The local kb checkout never leaves main.

**Architecture:** New core module `src/reins/review.py` (slug resolution, temp-clone ref push, merge detection, shared cleanup) + CLI verbs in `src/reins/commands/review.py` under a new top-level `review` noun. Registry-driven: `DocType.gated` controls which types use the drafts lifecycle. gh interactions live in `src/reins/github.py`; every verb degrades to a no-gh escape hatch (git half automated, GitHub half handed to the human).

**Tech Stack:** Python 3.12+ stdlib only, pytest, gh CLI (optional at runtime), GitHub Actions (kb-repo side).

## Working conventions (read first)

- Run all commands from the repo root. Tests: `uv run pytest <path> -v`. Full suite: `uv run pytest`.
- NEVER run raw `git` inside `kb/` — use `reins kb publish` for kb changes. (The reins *code* may run git in kb via `run_git(cwd=kb_dir)`; that rule is for agents editing docs.)
- After each kb-touching task: `uv run reins kb publish`, then `git add kb && git commit -m "chore(kb): <what>"`.
- No raw lifecycle/field strings in src/: import the `FIELD_*`/`STATUS_*` constants from `reins.docmeta` (defined in Task 2) and the `PR_STATE_*`/`REVIEW_DECISION_*` constants from `reins.github` (Task 5). Tests assert the raw literals on purpose — they pin the on-disk/API contract.
- Console output follows the axi channel model (`src/reins/console.py`): data + `next:` lines on stdout, progress on stderr. Never print data an agent needs to stderr.
- Commit messages end with: `Co-Authored-By: Claude <noreply@anthropic.com>`

## Acceptance Criteria

- [ ] `DocType` has `gated: bool = False`; `spec` is gated; seed tree grows `specs/drafts/`.
- [ ] `reins spec create` writes gated docs to `specs/drafts/<slug>.md`; non-gated types unchanged.
- [ ] `spec show/list` exclude `drafts/` by default; `--include-drafts` includes with `[DRAFT]` marker.
- [ ] `reins review start|push|merge|cancel|link|status|setup` all exist, follow axi output conventions, and each degrades per the no-gh escape hatch (ref pushed, human handed the exact URL + follow-up command).
- [ ] Review refs are `review/<repo-scope>/<type>-<slug>` (repo-scoped: two attached repos with the same slug cannot collide).
- [ ] `review merge` refuses on draft↔candidate divergence; merge detection works with git alone (candidate at final path on origin/main ⇒ merged).
- [ ] Shared idempotent cleanup (flip to approved, stamp `Approved-by`/`Review-PR`, delete draft) used by both `review merge` and the `_review-cleanup` internal command that the kb-repo CI workflow invokes.
- [ ] `review setup` installs the workflow yml into the kb repo idempotently (refuses to clobber user edits without `--force`) and best-effort applies the dismiss-stale-approvals ruleset, reporting when it can't.
- [ ] `review cancel` records `Review-cancelled: <date>` and keeps `Review-PR` (gardening signal); re-`start` clears it.
- [ ] Lint warns on active plans referencing `/drafts/` paths or in-review docs; legacy specs exempt.
- [ ] `kb status` shows an "In review" section (frontmatter-only, no gh required).
- [ ] Full `uv run pytest`, `uv run ruff check`, `uv run pyright` pass; spec's scripted acceptance run on the real kb repo executed and artifacts deleted.

## File Structure

- Create: `src/reins/docmeta.py` — provenance-field get/set/remove on the `**Field:** value` header block.
- Create: `src/reins/review.py` — review-lane core (no printing): resolve target, ref names, temp-clone push, merge detection, cleanup.
- Create: `src/reins/commands/review.py` — CLI verbs (printing, exit codes, escape hatches).
- Create: `src/reins/commands/internal/review_cleanup.py` — `_review-cleanup` entry point for CI (cwd = kb repo root).
- Create: `workflows/reins-doc-review-cleanup.yml` — shipped workflow template (bundled asset).
- Create: `src/reins/linter/rules/draft_refs.py` — lint rule.
- Modify: `src/reins/doc_types.py`, `src/reins/commands/doc_create.py`, `src/reins/commands/doc_show.py`, `src/reins/github.py`, `src/reins/commands/status.py`, `src/reins/cli.py`, `src/reins/kb_seed.py`, `src/reins/linter/rules/__init__.py`, `pyproject.toml` (bundle `workflows/`).
- Tests: `tests/test_docmeta.py`, `tests/test_review.py`, `tests/commands/test_review.py`, `tests/linter/test_draft_refs.py`, plus additions to `tests/test_doc_types.py`, `tests/commands/test_doc_create.py`, `tests/commands/test_doc_show.py`, `tests/test_github.py`, `tests/commands/test_status.py`.

---

## Tasks

### Task 1: Registry `gated` flag + drafts helpers

**Files:**
- Modify: `src/reins/doc_types.py`
- Test: `tests/test_doc_types.py`

- [ ] **Step 1: Write failing tests**

```python
# append to tests/test_doc_types.py
from pathlib import Path

from reins.doc_types import (
    DRAFTS_DIR_NAME, REGISTRY, drafts_dir, gated_types,
)

def test_spec_is_gated():
    assert REGISTRY["spec"].gated is True

def test_only_spec_gated_in_v1():
    assert [dt.key for dt in gated_types()] == ["spec"]

def test_gated_defaults_false():
    assert REGISTRY["plan"].gated is False
    assert REGISTRY["idea"].gated is False

def test_drafts_dir_under_type_dir():
    repo_dir = Path("/kb/myrepo")
    assert drafts_dir("spec", repo_dir) == repo_dir / "specs" / DRAFTS_DIR_NAME
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_doc_types.py -v -k "gated or drafts"`
Expected: FAIL — `ImportError: cannot import name 'DRAFTS_DIR_NAME'`

- [ ] **Step 3: Implement**

In `src/reins/doc_types.py`: add field to the dataclass (after `required_sections`), flag the spec entry, and append helpers:

```python
    required_sections: tuple[str, ...] = ()  # Linter checks these headers
    gated: bool = False  # Review-gated: create writes to drafts/, approval via the review lane
```

In `REGISTRY["spec"]` add `gated=True,` after `required_sections=(...)`.

```python
DRAFTS_DIR_NAME = "drafts"

def drafts_dir(key: str, repo_dir: Path) -> Path:
    """Drafts annex for a gated doc type within a repo scope dir."""
    return repo_dir / REGISTRY[key].dir_path / DRAFTS_DIR_NAME

def gated_types() -> list[DocType]:
    """All review-gated doc types (drafts lifecycle applies)."""
    return [dt for dt in REGISTRY.values() if dt.gated]
```

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/test_doc_types.py -v`
Expected: PASS (all, including pre-existing)

- [ ] **Step 5: Seed tree grows drafts dirs** — in `src/reins/kb_seed.py`, inside the registry loop of `generate_seed_tree`, after the `.gitkeep` touch:

```python
            if dt.gated:
                d = scope / dt.dir_path / "drafts"
                d.mkdir(parents=True, exist_ok=True)
                (d / ".gitkeep").touch()
```

(import nothing new; `drafts` literal matches `DRAFTS_DIR_NAME` — import and use the constant: `from reins.doc_types import DRAFTS_DIR_NAME, REGISTRY`.)

Add to `tests/test_kb_seed.py`:

```python
def test_seed_creates_drafts_dir_for_gated_types(tmp_path):
    from reins.kb_seed import generate_seed_tree
    generate_seed_tree(tmp_path, "myrepo")
    assert (tmp_path / "myrepo" / "specs" / "drafts" / ".gitkeep").is_file()
```

Run: `uv run pytest tests/test_kb_seed.py -v` — Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/reins/doc_types.py src/reins/kb_seed.py tests/test_doc_types.py tests/test_kb_seed.py
git commit -m "feat(registry): gated flag + drafts dir for review-gated doc types"
```

### Task 2: `docmeta` provenance-field helpers

Docs carry a `**Field:** value` header block (see `_provenance` in `src/reins/commands/doc_create.py`). The review lane reads/writes `Status`, `Review-PR`, `Approved-by`, `Review-cancelled`.

**Files:**
- Create: `src/reins/docmeta.py`
- Test: `tests/test_docmeta.py`

- [ ] **Step 1: Write failing tests**

```python
"""Tests for provenance-field helpers."""
from reins.docmeta import (
    FIELD_REVIEW_CANCELLED, FIELD_REVIEW_PR, FIELD_STATUS, STATUS_DRAFT,
    STATUS_IN_REVIEW, get_field, remove_field, set_field,
)

DOC = (
    "# My Spec\n\n"
    "**Date:** 2026-07-06\n"
    "**Author:** Test\n"
    "**Status:** draft\n"
    "**Origin:** ai-assisted\n"
    "\n## Problem\n\nBody **Status:** decoy\n"
)

def test_get_field():
    assert get_field(DOC, "Status") == "draft"
    assert get_field(DOC, "Author") == "Test"
    assert get_field(DOC, "Review-PR") is None

def test_set_existing_field_only_in_header():
    out = set_field(DOC, "Status", "in-review")
    assert "**Status:** in-review" in out
    assert "decoy" in out  # body untouched
    assert out.count("**Status:**") == 2  # header + body decoy

def test_set_new_field_appends_to_header_block():
    out = set_field(DOC, "Review-PR", "https://github.com/x/y/pull/9")
    lines = out.splitlines()
    idx = lines.index("**Review-PR:** https://github.com/x/y/pull/9")
    assert lines[idx - 1] == "**Origin:** ai-assisted"

def test_remove_field():
    out = remove_field(set_field(DOC, "Review-PR", "u"), "Review-PR")
    assert get_field(out, "Review-PR") is None

def test_remove_missing_field_is_noop():
    assert remove_field(DOC, "Review-PR") == DOC
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_docmeta.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'reins.docmeta'`

- [ ] **Step 3: Implement `src/reins/docmeta.py`**

```python
"""Read/write provenance fields in the `**Field:** value` doc header block.

The header block is the run of `**Field:** value` lines directly after the
`# title` heading (blank lines allowed between heading and block). Fields in
the body are never touched.
"""

from __future__ import annotations

import re

# Header-field vocabulary — the ONLY place these strings are defined.
# Implementation code imports these; tests assert the raw literals on
# purpose (pinning the on-disk format against constant typos).
FIELD_STATUS = "Status"
FIELD_REVIEW_PR = "Review-PR"
FIELD_APPROVED_BY = "Approved-by"
FIELD_REVIEW_CANCELLED = "Review-cancelled"

STATUS_DRAFT = "draft"
STATUS_IN_REVIEW = "in-review"
STATUS_APPROVED = "approved"

_FIELD_LINE = re.compile(r"^\*\*([A-Za-z-]+):\*\*\s*(.*)$")

def _header_span(lines: list[str]) -> tuple[int, int]:
    """(start, end) line indexes of the header field block (end exclusive).

    Returns (i, i) with i = insertion point when no block exists.
    """
    i = 0
    # Skip title heading and leading blanks
    while i < len(lines) and (not lines[i].strip() or lines[i].startswith("# ")):
        i += 1
    start = i
    while i < len(lines) and _FIELD_LINE.match(lines[i]):
        i += 1
    return start, i

def get_field(text: str, field: str) -> str | None:
    lines = text.splitlines()
    start, end = _header_span(lines)
    for line in lines[start:end]:
        m = _FIELD_LINE.match(line)
        if m and m.group(1) == field:
            return m.group(2).strip()
    return None

def set_field(text: str, field: str, value: str) -> str:
    lines = text.splitlines()
    start, end = _header_span(lines)
    for i in range(start, end):
        m = _FIELD_LINE.match(lines[i])
        if m and m.group(1) == field:
            lines[i] = f"**{field}:** {value}"
            break
    else:
        lines.insert(end, f"**{field}:** {value}")
    out = "\n".join(lines)
    return out + "\n" if text.endswith("\n") else out

def remove_field(text: str, field: str) -> str:
    lines = text.splitlines()
    start, end = _header_span(lines)
    kept = [
        line for i, line in enumerate(lines)
        if not (start <= i < end
                and (m := _FIELD_LINE.match(line))
                and m.group(1) == field)
    ]
    if kept == lines:
        return text
    out = "\n".join(kept)
    return out + "\n" if text.endswith("\n") else out
```

- [ ] **Step 4: Run tests** — `uv run pytest tests/test_docmeta.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/docmeta.py tests/test_docmeta.py
git commit -m "feat(docmeta): provenance-field get/set/remove helpers"
```

### Task 3: Gated `create` writes to drafts/

**Files:**
- Modify: `src/reins/commands/doc_create.py:39-51` (`_create_spec`)
- Test: `tests/commands/test_doc_create.py`

- [ ] **Step 1: Write failing tests** (use the existing test file's fixtures/pattern — it already exercises `cmd_spec_create` in a `kb_repo`; follow the file's existing call style)

```python
def test_spec_create_writes_to_drafts(kb_repo, monkeypatch):
    monkeypatch.chdir(kb_repo)
    from reins.commands.doc_create import cmd_spec_create
    assert cmd_spec_create("My Gated Spec") == 0
    target = kb_repo / "kb" / "testproject" / "specs" / "drafts" / "my-gated-spec.md"
    assert target.is_file()
    assert "**Status:** draft" in target.read_text()

def test_prd_create_stays_flat(kb_repo, monkeypatch):
    monkeypatch.chdir(kb_repo)
    from reins.commands.doc_create import cmd_prd_create
    assert cmd_prd_create("My PRD") == 0
    assert (kb_repo / "kb" / "testproject" / "prds" / "my-prd.md").is_file()
```

(If the existing tests in the file set up cwd or repo slug differently, mirror that setup exactly — the assertion paths are what matter.)

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_create.py -v -k "drafts or stays_flat"`
Expected: `test_spec_create_writes_to_drafts` FAILS (file created at `specs/my-gated-spec.md` instead)

- [ ] **Step 3: Implement** — in `_create_spec`, route through the registry:

```python
from reins.doc_types import DRAFTS_DIR_NAME, REGISTRY, get_protected_map

def _typed_dir(doc_type: str, repo_dir: Path) -> Path:
    """Directory a new doc of this type is created in (drafts annex when gated)."""
    dt = REGISTRY[doc_type]
    base = repo_dir / dt.dir_path
    return base / DRAFTS_DIR_NAME if dt.gated else base

def _create_spec(repo_dir: Path, title: str, author: str) -> Path:
    slug = _slugify(title)
    target = _typed_dir("spec", repo_dir) / f"{slug}.md"
    ...  # rest unchanged
```

Also in `_create_prd`/`_create_debt`, switch `repo_dir / "prds"` style paths to `_typed_dir("prd", repo_dir)` / `_typed_dir("debt", repo_dir)` so a future `gated=True` flip Just Works.

In `_create_typed`, after `console.success(...)`, add a gated next-step so authors discover the lane (the command exists from Task 7 on; adding the hint here keeps this task self-contained):

```python
    if REGISTRY.get(doc_type) and REGISTRY[doc_type].gated:
        console.next_step(f"reins review start {slug}  # when ready for review")
```

Place it before the existing `console.next_step("reins kb publish")` line so publish stays the immediate suggestion.

- [ ] **Step 4: Run tests** — `uv run pytest tests/commands/test_doc_create.py -v` — Expected: PASS (existing spec-path tests may assert the old flat path — update them to the drafts path; that is the intended behavior change)

- [ ] **Step 5: Check the path-protection hook still blocks** — `cmd_doc_check_path` matches on `subdir in protected` where subdir is `parts[kb_idx + 2]` (`specs`) — drafts files are `kb/<repo>/specs/drafts/<slug>.md`, so `subdir == "specs"` still blocks direct writes. Add a regression test:

```python
def test_check_path_blocks_drafts(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    from reins.commands.doc_create import cmd_doc_check_path
    rc = cmd_doc_check_path("kb/testproject/specs/drafts/new-spec.md")
    assert rc == 2
```

Run: `uv run pytest tests/commands/test_doc_create.py -v` — Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "feat(create): gated doc types scaffold into the drafts/ annex"
```

### Task 4: `--include-drafts` surfacing

**Files:**
- Modify: `src/reins/commands/doc_show.py`, `src/reins/cli.py`
- Test: `tests/commands/test_doc_show.py`

- [ ] **Step 1: Write failing tests** (mirror the file's existing fixture usage)

```python
def _mk_spec(repo_dir, name, status="approved", drafts=False):
    d = repo_dir / "specs" / ("drafts" if drafts else "")
    d.mkdir(parents=True, exist_ok=True)
    (d / f"{name}.md").write_text(
        f"# {name}\n\n**Status:** {status}\n\n## Problem\n\nx\n"
    )

def test_list_excludes_drafts_by_default(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    repo_dir = kb_repo / "kb" / "testproject"
    _mk_spec(repo_dir, "landed")
    _mk_spec(repo_dir, "wip", status="draft", drafts=True)
    from reins.commands.doc_show import cmd_doc_list
    assert cmd_doc_list("spec") == 0
    out = capsys.readouterr().out
    assert "landed" in out and "wip" not in out

def test_list_include_drafts_marks_them(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    repo_dir = kb_repo / "kb" / "testproject"
    _mk_spec(repo_dir, "wip", status="draft", drafts=True)
    from reins.commands.doc_show import cmd_doc_list
    assert cmd_doc_list("spec", include_drafts=True) == 0
    out = capsys.readouterr().out
    assert "[DRAFT]" in out and "wip" in out

def test_show_finds_draft_only_with_flag(kb_repo, monkeypatch, capsys):
    monkeypatch.chdir(kb_repo)
    _mk_spec(kb_repo / "kb" / "testproject", "wip", status="draft", drafts=True)
    from reins.commands.doc_show import cmd_doc_show
    assert cmd_doc_show("spec", "wip") == 1
    assert cmd_doc_show("spec", "wip", include_drafts=True) == 0
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_doc_show.py -v -k drafts`
Expected: FAIL — `TypeError: cmd_doc_list() got an unexpected keyword argument 'include_drafts'`

- [ ] **Step 3: Implement** — in `doc_show.py`:

```python
from reins.doc_types import DRAFTS_DIR_NAME, REGISTRY

def _doc_files(
    doc_type: str, repo_dir: Path, include_drafts: bool = False,
) -> list[Path]:
    """All docs of a slug-addressed type, index files and drafts excluded.

    The default glob never descends into drafts/ (pattern has no slash for
    slug-addressed types); include_drafts adds the annex explicitly.
    """
    dt = REGISTRY[doc_type]
    pattern = re.sub(r"\{\w+\}", "*", dt.filename)
    files = sorted((repo_dir / dt.dir_path).glob(pattern))
    if include_drafts and dt.gated:
        files += sorted(
            (repo_dir / dt.dir_path / DRAFTS_DIR_NAME).glob(pattern)
        )
    return [f for f in files if f.name != "index.md"]
```

Thread the kwarg through `cmd_doc_show(doc_type, slug, full=False, include_drafts=False)` and `cmd_doc_list(doc_type, include_drafts=False)`. In `cmd_doc_list`'s output loop, mark drafts:

```python
    for f in files:
        title, status = _title_and_status(f)
        marker = "[DRAFT] " if f.parent.name == DRAFTS_DIR_NAME else ""
        line = f"{marker}{f.stem} — {title}"
```

In `cmd_doc_show`, when the slug misses and the type is gated, hint: `console.next_step(f"reins {doc_type} show {slug} --include-drafts")` after the error (only when not already including).

- [ ] **Step 4: CLI wiring** — in `cli.py` `_doc_group_with_create_title`, add to both `show` and `list` parsers:

```python
        sp.add_argument("--include-drafts", action="store_true",
                        help="Include drafts/ (unapproved) docs")
        lp = gs.add_parser("list", help=f"List {name} docs")
        lp.add_argument("--include-drafts", action="store_true",
                        help="Include drafts/ (unapproved) docs")
```

(the `list` parser must be captured in a variable now). Update `_DISPATCH` entries for spec/prd/debt/idea show+list to pass `include_drafts=getattr(a, "include_drafts", False)`.

- [ ] **Step 5: Run tests** — `uv run pytest tests/commands/test_doc_show.py tests/test_cli_shape.py -v` — Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/doc_show.py src/reins/cli.py tests/commands/test_doc_show.py
git commit -m "feat(show/list): exclude drafts by default, --include-drafts opt-in with [DRAFT] marker"
```

### Task 5: gh PR wrappers

**Files:**
- Modify: `src/reins/github.py`
- Test: `tests/test_github.py`

- [ ] **Step 1: Write failing tests** (the existing file mocks `subprocess.run`; follow its pattern — the assertions below show the contract)

```python
def test_gh_pr_create_returns_url(mock_run_gh_success):
    # mock returns stdout "https://github.com/o/r/pull/7\n"
    from reins.github import gh_pr_create
    url = gh_pr_create(
        "o/r", head="review/myrepo/spec-x", title="t", body="b",
        reviewers=["alice", "bob"],
    )
    assert url == "https://github.com/o/r/pull/7"
    # command included: --repo o/r --head review/myrepo/spec-x
    # and one --reviewer flag per reviewer

def test_gh_pr_view_parses_json(mock_run_gh_json):
    # mock stdout: '{"number":7,"state":"OPEN","reviewDecision":"APPROVED","url":"u"}'
    from reins.github import gh_pr_view
    pr = gh_pr_view("o/r", head="review/myrepo/spec-x")
    assert pr["number"] == 7 and pr["reviewDecision"] == "APPROVED"

def test_gh_pr_view_none_when_no_pr(mock_run_gh_failure):
    from reins.github import gh_pr_view
    assert gh_pr_view("o/r", head="review/myrepo/spec-x") is None
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/test_github.py -v -k pr_` — Expected: ImportError FAILs

- [ ] **Step 3: Implement** — append to `github.py`:

```python
import json

# GitHub API enum values (external contract, mirrored once here)
PR_STATE_OPEN = "OPEN"
REVIEW_DECISION_APPROVED = "APPROVED"

def gh_pr_create(
    repo: str, *, head: str, title: str, body: str,
    reviewers: list[str] | None = None,
) -> str:
    """Open a PR from *head* into the repo's default branch. Returns the URL."""
    args = [
        "pr", "create", "--repo", repo, "--head", head,
        "--title", title, "--body", body,
    ]
    for r in reviewers or []:
        args.extend(["--reviewer", r])
    return run_gh(*args).stdout.strip()

def gh_pr_view(repo: str, *, head: str) -> dict | None:
    """PR metadata for the open/merged PR whose head is *head*, or None."""
    r = run_gh(
        "pr", "view", head, "--repo", repo,
        "--json", "number,state,reviewDecision,url,latestReviews",
        check=False,
    )
    if r.returncode != 0 or not r.stdout.strip():
        return None
    return json.loads(r.stdout)

def gh_pr_merge(repo: str, number: int) -> None:
    run_gh("pr", "merge", str(number), "--repo", repo, "--squash")

def gh_pr_close(repo: str, number: int, comment: str = "") -> None:
    args = ["pr", "close", str(number), "--repo", repo]
    if comment:
        args.extend(["--comment", comment])
    run_gh(*args)
```

- [ ] **Step 4: Run tests** — `uv run pytest tests/test_github.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/github.py tests/test_github.py
git commit -m "feat(github): PR create/view/merge/close wrappers"
```

### Task 6: review core — target resolution + remote naming

**Files:**
- Create: `src/reins/review.py`
- Test: `tests/test_review.py`

- [ ] **Step 1: Write failing tests**

```python
"""Tests for the review-lane core (no gh, no network)."""
from pathlib import Path

import pytest

from reins.review import (
    ReviewTarget, gh_repo_from_url, pr_new_url, resolve_draft, review_branch,
)

def test_review_branch_is_repo_scoped():
    assert review_branch("myrepo", "spec", "my-slug") == "review/myrepo/spec-my-slug"

@pytest.mark.parametrize("url", [
    "git@github.com:owner/name-kb.git",
    "https://github.com/owner/name-kb.git",
    "https://github.com/owner/name-kb",
])
def test_gh_repo_from_url(url):
    assert gh_repo_from_url(url) == "owner/name-kb"

def test_gh_repo_from_url_non_github_is_none():
    assert gh_repo_from_url("git@gitlab.com:o/n.git") is None

def test_pr_new_url():
    assert pr_new_url("owner/kb", "review/myrepo/spec-x") == (
        "https://github.com/owner/kb/pull/new/review/myrepo/spec-x"
    )

def test_resolve_draft_by_slug(tmp_path):
    repo_dir = tmp_path / "myrepo"
    d = repo_dir / "specs" / "drafts"
    d.mkdir(parents=True)
    (d / "my-slug.md").write_text("# t\n\n**Status:** draft\n")
    t = resolve_draft("my-slug", tmp_path, "myrepo")
    assert isinstance(t, ReviewTarget)
    assert t.doc_type.key == "spec"
    assert t.draft_path == d / "my-slug.md"
    assert t.final_rel == "myrepo/specs/my-slug.md"
    assert t.branch == "review/myrepo/spec-my-slug"

def test_resolve_draft_accepts_path(tmp_path):
    repo_dir = tmp_path / "myrepo"
    d = repo_dir / "specs" / "drafts"
    d.mkdir(parents=True)
    (d / "my-slug.md").write_text("# t\n")
    t = resolve_draft(str(d / "my-slug.md"), tmp_path, "myrepo")
    assert t.slug == "my-slug"

def test_resolve_draft_missing_returns_none(tmp_path):
    (tmp_path / "myrepo").mkdir()
    assert resolve_draft("nope", tmp_path, "myrepo") is None
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/test_review.py -v` — Expected: ModuleNotFoundError FAIL

- [ ] **Step 3: Implement `src/reins/review.py`** (first half)

```python
"""Review-lane core: server-side PR refs for gated kb docs.

Pure logic — no console printing (commands/review.py owns UX). The local kb
checkout never leaves main; all ref work happens in a temp clone (same
gc.auto=0 pattern as submodule.seed_remote).
"""

from __future__ import annotations

import re
import tempfile
from dataclasses import dataclass
from pathlib import Path

from reins.docmeta import (
    FIELD_APPROVED_BY, FIELD_REVIEW_PR, FIELD_STATUS, STATUS_APPROVED,
    STATUS_IN_REVIEW, get_field, set_field,
)
from reins.doc_types import DRAFTS_DIR_NAME, DocType, gated_types
from reins.git import run_git

REVIEW_REF_PREFIX = "review/"

@dataclass(frozen=True)
class ReviewTarget:
    doc_type: DocType
    slug: str
    repo_scope: str      # kb repo-scope dir name (e.g. "myrepo")
    draft_path: Path     # absolute path in the local kb working copy
    final_rel: str       # kb-repo-relative final path ("myrepo/specs/x.md")
    branch: str          # "review/myrepo/spec-x"

def review_branch(repo_scope: str, type_key: str, slug: str) -> str:
    return f"{REVIEW_REF_PREFIX}{repo_scope}/{type_key}-{slug}"

def resolve_draft(
    slug_or_path: str, kb_dir: Path, repo_scope: str,
) -> ReviewTarget | None:
    """Find a draft by slug across gated types, or by direct path."""
    repo_dir = kb_dir / repo_scope
    p = Path(slug_or_path)
    candidates: list[tuple[DocType, Path]] = []
    for dt in gated_types():
        d = repo_dir / dt.dir_path / DRAFTS_DIR_NAME
        direct = d / f"{p.stem}.md"
        if p.suffix == ".md" and p.resolve() == direct.resolve():
            candidates.append((dt, direct))
        elif p.suffix != ".md" and (d / f"{slug_or_path}.md").is_file():
            candidates.append((dt, d / f"{slug_or_path}.md"))
    if len(candidates) != 1:
        return None  # missing, or ambiguous across types (caller reports)
    dt, draft = candidates[0]
    slug = draft.stem
    return ReviewTarget(
        doc_type=dt,
        slug=slug,
        repo_scope=repo_scope,
        draft_path=draft,
        final_rel=f"{repo_scope}/{dt.dir_path}/{slug}.md",
        branch=review_branch(repo_scope, dt.key, slug),
    )

def gh_repo_from_url(url: str) -> str | None:
    """'owner/name' from a github.com remote URL (ssh or https), else None."""
    m = re.match(
        r"(?:git@github\.com:|https://github\.com/)([^/]+/[^/]+?)(?:\.git)?/?$",
        url.strip(),
    )
    return m.group(1) if m else None

def kb_remote_url(kb_dir: Path) -> str:
    return run_git("remote", "get-url", "origin", cwd=kb_dir).stdout.strip()

def pr_new_url(gh_repo: str, branch: str) -> str:
    return f"https://github.com/{gh_repo}/pull/new/{branch}"

def candidate_text(draft_text: str) -> str:
    """The reviewable candidate: draft content with Status set to in-review."""
    return set_field(draft_text, FIELD_STATUS, STATUS_IN_REVIEW)
```

Note for ambiguity: `resolve_draft` returning `None` for both missing and ambiguous is not enough for a good error message — split it: return a list instead. Change signature to `resolve_drafts(...) -> list[ReviewTarget]` and have `resolve_draft` = single-or-None convenience used by tests above; the command layer calls `resolve_drafts` and reports "not found" (empty) vs "ambiguous, pass --type" (len > 1), filtering by `--type` when given.

```python
def resolve_drafts(
    slug_or_path: str, kb_dir: Path, repo_scope: str,
    type_key: str | None = None,
) -> list[ReviewTarget]:
    """All gated-type drafts matching slug/path (0, 1, or ambiguous many)."""
    ...  # same loop as above, appending a ReviewTarget per hit;
         # skip types when type_key is given and differs

def resolve_draft(slug_or_path, kb_dir, repo_scope):
    matches = resolve_drafts(slug_or_path, kb_dir, repo_scope)
    return matches[0] if len(matches) == 1 else None
```

(Implement the loop once, in `resolve_drafts`; build `ReviewTarget` there.)

- [ ] **Step 4: Run tests** — `uv run pytest tests/test_review.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/review.py tests/test_review.py
git commit -m "feat(review): core target resolution and remote naming"
```

### Task 7: review core — ref push, merge detection, cleanup

**Files:**
- Modify: `src/reins/review.py`
- Test: `tests/test_review.py`

- [ ] **Step 1: Write failing tests** — build a bare kb remote + local clone fixture (pattern from `tests/test_kb_seed.py` / `conftest.py` `_git_init`):

```python
import subprocess

@pytest.fixture
def kb_pair(tmp_path):
    """(bare_remote, local_kb) — local cloned from bare, one commit on main."""
    bare = tmp_path / "kb-remote.git"
    subprocess.run(["git", "init", "-q", "--bare", "-b", "main", str(bare)], check=True)
    local = tmp_path / "kb"
    subprocess.run(["git", "clone", "-q", str(bare), str(local)], check=True)
    for k, v in (("user.email", "t@t"), ("user.name", "T")):
        subprocess.run(["git", "config", k, v], cwd=local, check=True)
    d = local / "myrepo" / "specs" / "drafts"
    d.mkdir(parents=True)
    (d / "x.md").write_text("# X\n\n**Status:** draft\n\n## Problem\n\nbody\n")
    subprocess.run(["git", "add", "-A"], cwd=local, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "init"], cwd=local, check=True)
    subprocess.run(["git", "push", "-q", "origin", "main"], cwd=local, check=True)
    return bare, local

def _remote_file(bare, ref, rel):
    r = subprocess.run(
        ["git", "show", f"{ref}:{rel}"], cwd=bare,
        capture_output=True, text=True,
    )
    return r.stdout if r.returncode == 0 else None

def test_push_candidate_creates_ref_with_only_final_file(kb_pair):
    bare, local = kb_pair
    from reins.review import push_candidate, resolve_draft
    t = resolve_draft("x", local, "myrepo")
    push_candidate(local, t)
    cand = _remote_file(bare, t.branch, "myrepo/specs/x.md")
    assert cand and "**Status:** in-review" in cand
    # draft untouched on the ref (add-only: rename detection must find no delete)
    assert _remote_file(bare, t.branch, "myrepo/specs/drafts/x.md") is not None

def test_push_candidate_updates_existing_ref(kb_pair):
    bare, local = kb_pair
    from reins.review import push_candidate, resolve_draft
    t = resolve_draft("x", local, "myrepo")
    push_candidate(local, t)
    (local / "myrepo/specs/drafts/x.md").write_text(
        "# X\n\n**Status:** draft\n\n## Problem\n\nrevised\n"
    )
    push_candidate(local, t)
    assert "revised" in _remote_file(bare, t.branch, "myrepo/specs/x.md")

def test_merged_on_main_detection(kb_pair):
    bare, local = kb_pair
    from reins.review import merged_on_main, push_candidate, resolve_draft
    t = resolve_draft("x", local, "myrepo")
    assert merged_on_main(local, t) is False
    push_candidate(local, t)
    # simulate a GitHub merge: fast-forward main to the review ref
    subprocess.run(["git", "fetch", "-q", "origin", t.branch], cwd=local, check=True)
    subprocess.run(["git", "push", "-q", "origin",
                    f"origin/{t.branch}:main"], cwd=local, check=True)
    assert merged_on_main(local, t) is True

def test_cleanup_after_merge_flips_stamps_deletes(kb_pair):
    bare, local = kb_pair
    from reins.review import cleanup_after_merge, push_candidate, resolve_draft
    t = resolve_draft("x", local, "myrepo")
    push_candidate(local, t)
    subprocess.run(["git", "fetch", "-q", "origin", t.branch], cwd=local, check=True)
    subprocess.run(["git", "push", "-q", "origin",
                    f"origin/{t.branch}:main"], cwd=local, check=True)
    assert cleanup_after_merge(local, t, pr_url="https://x/pull/1") is True
    final = _remote_file(bare, "main", "myrepo/specs/x.md")
    assert "**Status:** approved" in final
    assert "**Review-PR:** https://x/pull/1" in final
    assert _remote_file(bare, "main", "myrepo/specs/drafts/x.md") is None
    # idempotent second run is a no-op
    assert cleanup_after_merge(local, t, pr_url="https://x/pull/1") is False

def test_divergence_detection(kb_pair):
    bare, local = kb_pair
    from reins.review import candidate_matches_draft, push_candidate, resolve_draft
    t = resolve_draft("x", local, "myrepo")
    push_candidate(local, t)
    assert candidate_matches_draft(local, t) is True
    (local / "myrepo/specs/drafts/x.md").write_text("# X\n\n**Status:** draft\n\nchanged\n")
    assert candidate_matches_draft(local, t) is False
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/test_review.py -v -k "push_candidate or merged or cleanup or divergence"` — Expected: ImportError FAILs

- [ ] **Step 3: Implement** — append to `review.py`:

```python
def _file_allow(url: str) -> tuple[str, ...]:
    return ("-c", "protocol.file.allow=always") if url.startswith("/") else ()

def _temp_clone(url: str):
    """Context manager yielding a temp clone path (gc disabled, user set)."""
    return tempfile.TemporaryDirectory(ignore_cleanup_errors=True)

def _clone_into(url: str, tmp: str) -> Path:
    path = Path(tmp) / "kb-review"
    run_git(
        *_file_allow(url), "clone", "-q", "--depth", "1",
        "-c", "gc.auto=0", "-c", "maintenance.auto=false",
        url, str(path),
    )
    run_git("config", "user.email", "reins@review", cwd=path)
    run_git("config", "user.name", "Reins Review", cwd=path)
    return path

def push_candidate(kb_dir: Path, target: ReviewTarget) -> None:
    """Create/update the review ref so it differs from main by exactly one
    added file: the candidate at the final path. The draft copy on main is
    untouched, so GitHub's rename detection has nothing to pair (spec:
    add-only PR trick)."""
    url = kb_remote_url(kb_dir)
    content = candidate_text(target.draft_path.read_text())
    with _temp_clone(url) as tmp:
        clone = _clone_into(url, tmp)
        run_git("checkout", "-q", "-B", target.branch, "origin/main", cwd=clone)
        final = clone / target.final_rel
        final.parent.mkdir(parents=True, exist_ok=True)
        final.write_text(content)
        run_git("add", "--", target.final_rel, cwd=clone)
        status = run_git("status", "--porcelain", cwd=clone).stdout.strip()
        if status.splitlines() != [f"A  {target.final_rel}"]:
            raise RuntimeError(
                f"review ref would touch more than the candidate file:\n{status}"
            )
        run_git("commit", "-q", "-m",
                f"review({target.doc_type.key}): {target.slug} candidate",
                cwd=clone)
        run_git(*_file_allow(url), "push", "-q", "-f", "origin",
                target.branch, cwd=clone)

def delete_review_ref(kb_dir: Path, target: ReviewTarget) -> None:
    url = kb_remote_url(kb_dir)
    run_git(*_file_allow(url), "push", "-q", "origin",
            f":refs/heads/{target.branch}", check=False, cwd=kb_dir)

def _remote_show(kb_dir: Path, ref: str, rel: str) -> str | None:
    url = kb_remote_url(kb_dir)
    r = run_git(*_file_allow(url), "fetch", "-q", "origin", ref,
                check=False, cwd=kb_dir)
    if r.returncode != 0:
        return None
    show = run_git("show", f"FETCH_HEAD:{rel}", check=False, cwd=kb_dir)
    return show.stdout if show.returncode == 0 else None

def candidate_on_ref(kb_dir: Path, target: ReviewTarget) -> str | None:
    return _remote_show(kb_dir, target.branch, target.final_rel)

def merged_on_main(kb_dir: Path, target: ReviewTarget) -> bool:
    """Candidate at the final path on origin/main ⇒ the PR merged. Pure git —
    part of the no-gh escape hatch."""
    return _remote_show(kb_dir, "main", target.final_rel) is not None

def candidate_matches_draft(kb_dir: Path, target: ReviewTarget) -> bool:
    cand = candidate_on_ref(kb_dir, target)
    if cand is None:
        return False
    return cand == candidate_text(target.draft_path.read_text())

def cleanup_after_merge(
    kb_dir: Path, target: ReviewTarget, pr_url: str,
    approved_by: str = "", retries: int = 2,
) -> bool:
    """Post-merge finalize on main: Status→approved, stamp Review-PR, delete
    the draft. Shared by `review merge` and CI `_review-cleanup`. Idempotent;
    pull–rebase–retry on push races. Returns True if it changed anything."""
    url = kb_remote_url(kb_dir)
    for _ in range(retries + 1):
        with _temp_clone(url) as tmp:
            clone = _clone_into(url, tmp)
            final = clone / target.final_rel
            draft = clone / target.repo_scope / target.doc_type.dir_path \
                / DRAFTS_DIR_NAME / f"{target.slug}.md"
            changed = False
            if final.is_file():
                text = final.read_text()
                if get_field(text, FIELD_STATUS) != STATUS_APPROVED:
                    text = set_field(text, FIELD_STATUS, STATUS_APPROVED)
                    if pr_url:
                        text = set_field(text, FIELD_REVIEW_PR, pr_url)
                    if approved_by:
                        text = set_field(text, FIELD_APPROVED_BY, approved_by)
                    final.write_text(text)
                    changed = True
            if draft.is_file():
                run_git("rm", "-q", "--", str(draft.relative_to(clone)), cwd=clone)
                changed = True
            if not changed:
                return False
            run_git("add", "-A", cwd=clone)
            run_git("commit", "-q", "-m",
                    f"review({target.doc_type.key}): approve {target.slug}, "
                    "remove draft", cwd=clone)
            push = run_git(*_file_allow(url), "push", "-q", "origin",
                           "HEAD:main", check=False, cwd=clone)
            if push.returncode == 0:
                return True
    raise RuntimeError("cleanup push kept failing after retries")
```

Note on `_temp_clone`/`_clone_into` shape: it mirrors `submodule.seed_remote`'s gc-race fix (`src/reins/submodule.py:40-58`); a shallow clone is enough because we branch from `origin/main` tip. Also stamp `Approved-by`: `gh pr view --json reviews` is a gh nicety — in `cleanup_after_merge` accept an optional `approved_by: str = ""` and set the field only when non-empty (the CI path and no-gh path pass nothing).

- [ ] **Step 4: Run tests** — `uv run pytest tests/test_review.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/review.py tests/test_review.py
git commit -m "feat(review): ref push, git-only merge detection, shared idempotent cleanup"
```

### Task 8: `reins review` CLI verbs

**Files:**
- Create: `src/reins/commands/review.py`
- Modify: `src/reins/cli.py`
- Test: `tests/commands/test_review.py`

- [ ] **Step 1: Write failing tests** — use the existing `submodule_repo` fixture (`tests/conftest.py:99` — parent repo + real kb submodule + bare remote at `tmp_path/kb-remote`). Every test wraps the command call in the same patch stack `tests/commands/test_publish.py` uses, extended for review:

```python
import subprocess
from pathlib import Path
from unittest.mock import patch

def _draft(parent: Path, slug: str = "x") -> Path:
    """Publish a draft into the fixture kb (scope 'testproject')."""
    d = parent / "kb" / "testproject" / "specs" / "drafts"
    d.mkdir(parents=True, exist_ok=True)
    f = d / f"{slug}.md"
    f.write_text(f"# {slug}\n\n**Status:** draft\n\n## Problem\n\nbody\n")
    kb = parent / "kb"
    subprocess.run(["git", "add", "-A"], cwd=kb, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "draft"], cwd=kb, check=True)
    subprocess.run(["git", "push", "-q", "origin", "main"], cwd=kb, check=True)
    return f

def _review_ctx(parent: Path):
    """Patch stack: repo location, scope name, and mode gate."""
    return (
        patch("reins.commands.review.repo_root", return_value=parent),
        patch("reins.commands.review.repo_slug", return_value="testproject"),
        patch("reins.commands.review.can_publish", return_value=True),
    )
```

Call each command inside `with _review_ctx(parent)[0], _review_ctx(parent)[1], _review_ctx(parent)[2]:` — or cleaner, make `_review_ctx` a `contextlib.ExitStack`-based context manager (engineer's choice; keep it in this test file). In the tests below, `submodule_repo` replaces the illustrative `submodule_repo` name, and each begins with `_draft(submodule_repo)`.

```python
def test_start_happy_path_prints_pr_url(submodule_repo, monkeypatch, capsys):
    _draft(submodule_repo)
    monkeypatch.setattr("reins.github.gh_available", lambda: True)
    monkeypatch.setattr("reins.github.gh_authenticated", lambda: True)
    monkeypatch.setattr(
        "reins.github.gh_pr_create",
        lambda repo, **kw: "https://github.com/o/kb/pull/12",
    )
    monkeypatch.setattr(
        "reins.review.gh_repo_from_url", lambda url: "o/kb",
    )
    from reins.commands.review import cmd_review_start
    assert cmd_review_start("x", reviewers=[]) == 0
    out = capsys.readouterr().out
    assert "https://github.com/o/kb/pull/12" in out
    draft = (submodule_repo / "kb/testproject/specs/drafts/x.md").read_text()
    assert "**Status:** in-review" in draft
    assert "**Review-PR:** https://github.com/o/kb/pull/12" in draft

def test_start_no_gh_prints_escape_hatch(submodule_repo, monkeypatch, capsys):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)
    monkeypatch.setattr("reins.review.gh_repo_from_url", lambda url: "o/kb")
    from reins.commands.review import cmd_review_start
    assert cmd_review_start("x", reviewers=[]) == 0
    out = capsys.readouterr().out
    assert "https://github.com/o/kb/pull/new/review/testproject/spec-x" in out
    assert "reins review link x" in out  # follow-up instruction

def test_start_missing_slug_errors(submodule_repo, capsys):
    from reins.commands.review import cmd_review_start
    assert cmd_review_start("nope", reviewers=[]) == 1
    assert "no draft" in capsys.readouterr().out

def test_link_records_pr(submodule_repo, capsys):
    from reins.commands.review import cmd_review_link
    assert cmd_review_link("x", "https://github.com/o/kb/pull/9") == 0
    draft = (submodule_repo / "kb/testproject/specs/drafts/x.md").read_text()
    assert "**Review-PR:** https://github.com/o/kb/pull/9" in draft

def test_cancel_stamps_gardening_signal(submodule_repo, monkeypatch, capsys):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)
    from reins.commands.review import cmd_review_cancel, cmd_review_start
    cmd_review_start("x", reviewers=[])
    assert cmd_review_cancel("x") == 0
    draft = (submodule_repo / "kb/testproject/specs/drafts/x.md").read_text()
    assert "**Status:** draft" in draft
    assert "**Review-cancelled:**" in draft

def test_restart_clears_cancellation(submodule_repo, monkeypatch):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)
    from reins.commands.review import cmd_review_cancel, cmd_review_start
    cmd_review_start("x", reviewers=[])
    cmd_review_cancel("x")
    cmd_review_start("x", reviewers=[])
    draft = (submodule_repo / "kb/testproject/specs/drafts/x.md").read_text()
    assert "**Review-cancelled:**" not in draft
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/commands/test_review.py -v` — Expected: ModuleNotFoundError FAIL

- [ ] **Step 3: Implement `src/reins/commands/review.py`**

```python
"""reins review — PR-style review lane for gated kb docs."""

from __future__ import annotations

from datetime import date
from pathlib import Path

from reins import console, github
from reins.docmeta import (
    FIELD_REVIEW_CANCELLED, FIELD_REVIEW_PR, FIELD_STATUS, STATUS_DRAFT,
    STATUS_IN_REVIEW, get_field, remove_field, set_field,
)
from reins.git import repo_root, repo_slug
from reins.kb import commit_kb, ensure_kb_on_main, require_kb_dir
from reins.mode import can_publish, get_mode
from reins.review import (
    ReviewTarget, candidate_matches_draft, cleanup_after_merge,
    delete_review_ref, gh_repo_from_url, kb_remote_url, merged_on_main,
    pr_new_url, push_candidate, resolve_drafts,
)

def _ctx(slug_or_path: str, type_key: str | None = None):
    """(root, kb_dir, target) or None after printing the error."""
    root = repo_root()
    if root is None:
        return None
    kb_dir = require_kb_dir(root)
    ensure_kb_on_main(kb_dir)
    matches = resolve_drafts(slug_or_path, kb_dir, repo_slug(), type_key)
    if not matches:
        console.error(f"no draft named '{slug_or_path}'")
        console.next_step("reins spec list --include-drafts")
        return None
    if len(matches) > 1:
        keys = ", ".join(m.doc_type.key for m in matches)
        console.error(f"'{slug_or_path}' is ambiguous across types: {keys}")
        console.next_step(f"reins review start {slug_or_path} --type <key>")
        return None
    return root, kb_dir, matches[0]

def _gh_ready() -> bool:
    return github.gh_available() and github.gh_authenticated()

def _stamp_draft(
    root, kb_dir, target: ReviewTarget, message: str,
    fields: dict[str, str | None],
) -> None:
    """Set/remove header fields on the on-main draft and commit the kb.

    Keys are docmeta FIELD_* constants; a value of None removes the field.
    """
    text = target.draft_path.read_text()
    for field, value in fields.items():
        text = remove_field(text, field) if value is None \
            else set_field(text, field, value)
    target.draft_path.write_text(text)
    commit_kb(root, message, kb_dir=kb_dir)

def cmd_review_start(
    slug: str, reviewers: list[str], type_key: str | None = None,
) -> int:
    if not can_publish():
        console.error(f"Review blocked (mode: {get_mode()}).")
        console.next_step("reins mode enable")
        return 1
    ctx = _ctx(slug, type_key)
    if ctx is None:
        return 1
    root, kb_dir, target = ctx
    push_candidate(kb_dir, target)
    gh_repo = gh_repo_from_url(kb_remote_url(kb_dir))

    pr_url: str | None = None
    if gh_repo and _gh_ready():
        text = target.draft_path.read_text()
        title = next(
            (line[2:].strip() for line in text.splitlines()
             if line.startswith("# ")),
            target.slug,
        )
        pr_url = github.gh_pr_create(
            gh_repo, head=target.branch,
            title=f"[doc-review] {target.doc_type.key}: {title}",
            body=f"Review of `{target.final_rel}` via the reins doc-review lane.",
            reviewers=reviewers,
        )
    _stamp_draft(
        root, kb_dir, target,
        f"review({target.doc_type.key}): start {target.slug}",
        {
            FIELD_STATUS: STATUS_IN_REVIEW,
            FIELD_REVIEW_PR: pr_url,        # None → removed when no gh yet
            FIELD_REVIEW_CANCELLED: None,   # restart clears the gardening marker
        },
    )
    if pr_url:
        print(pr_url)
        console.next_step(f"reins review status")
        return 0
    # No-gh escape hatch: ref is pushed; hand the human the GitHub half.
    console.warn("gh unavailable — review ref pushed, PR must be opened manually.")
    if gh_repo:
        print(pr_new_url(gh_repo, target.branch))
    console.next_step(f"reins review link {target.slug} <pr-url>")
    return 0

def cmd_review_push(slug: str, type_key: str | None = None) -> int:
    if not can_publish():
        console.error(f"Review blocked (mode: {get_mode()}).")
        return 1
    ctx = _ctx(slug, type_key)
    if ctx is None:
        return 1
    root, kb_dir, target = ctx
    push_candidate(kb_dir, target)
    console.success("Candidate updated on the review ref.")
    console.warn("Pushed changes dismiss existing approvals — reviewers re-review.")
    url = get_field(target.draft_path.read_text(), FIELD_REVIEW_PR)
    if url:
        print(url)
    return 0

def cmd_review_link(slug: str, pr_url: str, type_key: str | None = None) -> int:
    ctx = _ctx(slug, type_key)
    if ctx is None:
        return 1
    root, kb_dir, target = ctx
    _stamp_draft(
        root, kb_dir, target,
        f"review({target.doc_type.key}): link PR for {target.slug}",
        {FIELD_STATUS: STATUS_IN_REVIEW, FIELD_REVIEW_PR: pr_url},
    )
    console.success(f"Linked {pr_url}")
    return 0

def cmd_review_merge(
    slug: str, type_key: str | None = None, force: bool = False,
) -> int:
    if not can_publish():
        console.error(f"Review blocked (mode: {get_mode()}).")
        return 1
    ctx = _ctx(slug, type_key)
    if ctx is None:
        return 1
    root, kb_dir, target = ctx
    pr_url = get_field(target.draft_path.read_text(), FIELD_REVIEW_PR) or ""
    approved_by = ""
    gh_repo = gh_repo_from_url(kb_remote_url(kb_dir))

    if not merged_on_main(kb_dir, target):
        if not candidate_matches_draft(kb_dir, target) and not force:
            console.error(
                "draft has changed since the last 'review push' — the PR "
                "does not show what you'd be approving."
            )
            console.next_step(f"reins review push {target.slug}",
                              f"reins review merge {target.slug} --force")
            return 1
        if gh_repo and _gh_ready():
            pr = github.gh_pr_view(gh_repo, head=target.branch)
            if pr is None:
                console.error("no PR found for the review ref")
                console.next_step(f"reins review link {target.slug} <pr-url>")
                return 1
            if pr["reviewDecision"] != github.REVIEW_DECISION_APPROVED and not force:
                console.error(f"PR not approved (decision: {pr['reviewDecision'] or 'none'})")
                print(pr["url"])
                return 1
            github.gh_pr_merge(gh_repo, pr["number"])
            pr_url = pr["url"]
            approved_by = ",".join(sorted({
                r["author"]["login"] for r in pr.get("latestReviews", [])
                if r.get("state") == github.REVIEW_DECISION_APPROVED
            }))
        else:
            # No-gh escape hatch: human merges in the UI, we detect via git.
            console.warn("gh unavailable — merge the PR in the GitHub UI, then rerun.")
            if pr_url:
                print(pr_url)
            return 1
    if cleanup_after_merge(kb_dir, target, pr_url=pr_url, approved_by=approved_by):
        console.success(f"{target.slug} approved and landed at {target.final_rel}")
    else:
        console.info("already cleaned up — nothing to do")
    run_git_pull_kb(kb_dir)
    return 0

def run_git_pull_kb(kb_dir) -> None:
    from reins.git import file_transport_args, run_git
    fta = file_transport_args(cwd=kb_dir)
    run_git(*fta, "pull", "-q", "--no-rebase", "origin", "main",
            check=False, cwd=kb_dir)

def cmd_review_cancel(slug: str, type_key: str | None = None) -> int:
    ctx = _ctx(slug, type_key)
    if ctx is None:
        return 1
    root, kb_dir, target = ctx
    pr_url = get_field(target.draft_path.read_text(), FIELD_REVIEW_PR)
    gh_repo = gh_repo_from_url(kb_remote_url(kb_dir))
    if gh_repo and _gh_ready():
        pr = github.gh_pr_view(gh_repo, head=target.branch)
        if pr and pr["state"] == github.PR_STATE_OPEN:
            github.gh_pr_close(gh_repo, pr["number"],
                               comment="Review cancelled via reins.")
    elif pr_url:
        console.warn("gh unavailable — close the PR manually:")
        print(pr_url)
    delete_review_ref(kb_dir, target)
    _stamp_draft(
        root, kb_dir, target,
        f"review({target.doc_type.key}): cancel {target.slug}",
        {
            FIELD_STATUS: STATUS_DRAFT,
            FIELD_REVIEW_CANCELLED: date.today().isoformat(),
        },
    )
    console.success(f"review cancelled — {target.slug} back to draft")
    return 0

def cmd_review_status() -> int:
    root = repo_root()
    if root is None:
        return 1
    kb_dir = require_kb_dir(root)
    repo_dir = kb_dir / repo_slug()
    from reins.doc_types import DRAFTS_DIR_NAME, gated_types
    rows = []
    for dt in gated_types():
        d = repo_dir / dt.dir_path / DRAFTS_DIR_NAME
        for f in sorted(d.glob("*.md")) if d.is_dir() else []:
            text = f.read_text()
            rows.append((
                dt.key, f.stem,
                get_field(text, FIELD_STATUS) or STATUS_DRAFT,
                get_field(text, FIELD_REVIEW_PR) or "",
            ))
    if not rows:
        print("doc reviews: 0 open")
        return 0
    print(f"doc reviews: {len(rows)}")
    for key, slug, status, url in rows:
        line = f"{key}/{slug} [{status}]"
        if url:
            line += f" {url}"
        console.info(line)
    return 0
```

Fix while implementing: move `run_git_pull_kb` to a private `_pull_kb_main(kb_dir)` above the commands. `_stamp_draft` removing `Review-PR` when `pr_url is None` on a no-gh start is correct (nothing to record yet — `review link` records it).

- [ ] **Step 4: CLI wiring** — in `cli.py`, after the principle group:

```python
    # ── Review lane (gated kb docs) ────────────────────────
    review_p = sub.add_parser("review", help="PR-style review lane for gated kb docs")
    review_sub = review_p.add_subparsers(dest="review_command")
    review_sub.required = True

    def _slug_verb(name: str, help_text: str):
        vp = review_sub.add_parser(name, help=help_text)
        vp.add_argument("slug", help="Draft slug or path")
        vp.add_argument("--type", dest="type_key", default=None,
                        help="Doc type key when the slug is ambiguous")
        return vp

    start_p = _slug_verb("start", "Push review ref and open the PR")
    start_p.add_argument("--reviewer", action="append", default=[],
                         dest="reviewers", help="GitHub reviewer (repeatable)")
    _slug_verb("push", "Sync draft edits to the review PR")
    merge_p = _slug_verb("merge", "Merge the approved PR and land the doc")
    merge_p.add_argument("--force", action="store_true",
                         help="Merge despite divergence/approval warnings")
    _slug_verb("cancel", "Close the PR, return the draft to draft status")
    link_p = _slug_verb("link", "Record a manually-created PR")
    link_p.add_argument("pr_url", help="PR URL")
    review_sub.add_parser("status", help="All open doc reviews")
    review_sub.add_parser("setup", help="Install kb-side review automation").add_argument(
        "--force", action="store_true", help="Overwrite a modified workflow file")
```

Dispatch entries:

```python
    ("review", "start"): lambda a: _load("review", "cmd_review_start")(
        a.slug, reviewers=a.reviewers, type_key=a.type_key),
    ("review", "push"): lambda a: _load("review", "cmd_review_push")(a.slug, type_key=a.type_key),
    ("review", "merge"): lambda a: _load("review", "cmd_review_merge")(
        a.slug, type_key=a.type_key, force=a.force),
    ("review", "cancel"): lambda a: _load("review", "cmd_review_cancel")(a.slug, type_key=a.type_key),
    ("review", "link"): lambda a: _load("review", "cmd_review_link")(
        a.slug, a.pr_url, type_key=a.type_key),
    ("review", "status"): lambda _: _load("review", "cmd_review_status")(),
    ("review", "setup"): lambda a: _load("review", "cmd_review_setup")(force=a.force),
```

(`cmd_review_setup` arrives in Task 10 — stub it now returning 1 with `console.error("not implemented")` so `tests/test_cli_shape.py` stays green, or add the dispatch entry in Task 10; pick whichever keeps the CLI-shape test passing.)

- [ ] **Step 5: Run tests** — `uv run pytest tests/commands/test_review.py tests/test_cli_shape.py -v` — Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/review.py src/reins/cli.py tests/commands/test_review.py
git commit -m "feat(review): start/push/merge/cancel/link/status CLI verbs with no-gh escape hatch"
```

### Task 9: `_review-cleanup` internal command (CI entry point)

Runs **inside the kb repo** (Actions checkout), not the parent repo: cwd is the kb root.

**Files:**
- Create: `src/reins/commands/internal/review_cleanup.py`
- Modify: `src/reins/cli.py` (`_INTERNAL_COMMANDS` + `_dispatch_internal`)
- Test: `tests/commands/internal/test_review_cleanup.py`

- [ ] **Step 1: Write failing test**

```python
"""_review-cleanup runs with cwd = kb repo root (the Actions checkout)."""
import subprocess

def test_cleanup_from_ref_name(kb_pair_cloned_at_merged_state, monkeypatch):
    # fixture: bare remote + local clone where review/myrepo/spec-x was
    # fast-forwarded into main (reuse the Task 7 kb_pair fixture steps)
    bare, local = kb_pair_cloned_at_merged_state
    monkeypatch.chdir(local)
    from reins.commands.internal.review_cleanup import cmd_review_cleanup
    rc = cmd_review_cleanup(["review/myrepo/spec-x", "https://x/pull/1"])
    assert rc == 0
    final = subprocess.run(
        ["git", "show", "main:myrepo/specs/x.md"], cwd=bare,
        capture_output=True, text=True,
    ).stdout
    assert "**Status:** approved" in final

def test_cleanup_rejects_malformed_ref(monkeypatch, tmp_path):
    monkeypatch.chdir(tmp_path)
    from reins.commands.internal.review_cleanup import cmd_review_cleanup
    assert cmd_review_cleanup(["not-a-review-ref"]) == 1
```

Extract the Task 7 `kb_pair` fixture into `tests/conftest.py` (name it `kb_pair`) so both test files share it; build `kb_pair_cloned_at_merged_state` on top of it in this test file.

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/commands/internal/test_review_cleanup.py -v` — Expected: ModuleNotFoundError FAIL

- [ ] **Step 3: Implement**

```python
"""Internal: post-merge cleanup, invoked by the kb-repo CI workflow.

Usage: reins _review-cleanup <head-ref> <pr-url>
cwd must be the kb repo root (Actions checkout of main).
"""

from __future__ import annotations

import re
from pathlib import Path

from reins import console
from reins.doc_types import REGISTRY
from reins.review import REVIEW_REF_PREFIX, ReviewTarget, cleanup_after_merge, review_branch

_REF_RE = re.compile(
    rf"^{REVIEW_REF_PREFIX}(?P<scope>[^/]+)/(?P<type>[a-z]+)-(?P<slug>[a-z0-9-]+)$"
)

def cmd_review_cleanup(args: list[str]) -> int:
    if not args:
        console.error("usage: reins _review-cleanup <head-ref> [pr-url]")
        return 1
    m = _REF_RE.match(args[0])
    if m is None or m.group("type") not in REGISTRY:
        console.error(f"not a review ref: {args[0]}")
        return 1
    pr_url = args[1] if len(args) > 1 else ""
    kb_root = Path.cwd()
    dt = REGISTRY[m.group("type")]
    scope, slug = m.group("scope"), m.group("slug")
    target = ReviewTarget(
        doc_type=dt, slug=slug, repo_scope=scope,
        draft_path=kb_root / scope / dt.dir_path / "drafts" / f"{slug}.md",
        final_rel=f"{scope}/{dt.dir_path}/{slug}.md",
        branch=review_branch(scope, dt.key, slug),
    )
    changed = cleanup_after_merge(kb_root, target, pr_url=pr_url)
    console.success("cleanup done" if changed else "already clean — no-op")
    return 0
```

Wire in `cli.py`: add `"_review-cleanup"` to `_INTERNAL_COMMANDS` and a branch in `_dispatch_internal`:

```python
    if cmd == "_review-cleanup":
        from reins.commands.internal.review_cleanup import cmd_review_cleanup
        return cmd_review_cleanup(rest)
```

Note: `cleanup_after_merge` calls `kb_remote_url(kb_dir)` — in CI the checkout's `origin` is the kb repo itself, so this works unchanged.

- [ ] **Step 4: Run tests** — `uv run pytest tests/commands/internal/test_review_cleanup.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/commands/internal/review_cleanup.py src/reins/cli.py tests/
git commit -m "feat(review): _review-cleanup internal command for kb-repo CI"
```

### Task 10: workflow asset + `review setup`

**Files:**
- Create: `workflows/reins-doc-review-cleanup.yml` (repo root; bundled asset)
- Modify: `pyproject.toml` (force-include `workflows/` under `reins/_data/workflows/` — mirror how existing assets are bundled; check `[tool.hatch.build.targets.wheel.force-include]`)
- Modify: `src/reins/commands/review.py` (`cmd_review_setup`)
- Test: `tests/commands/test_review.py`, `tests/test_assets.py`

- [ ] **Step 1: Write the workflow template** at `workflows/reins-doc-review-cleanup.yml`:

```yaml
# Installed into kb repos by `reins review setup`. Do not edit in place —
# edits are overwritten by the next setup --force.
name: reins doc-review cleanup
on:
  pull_request:
    types: [closed]
jobs:
  cleanup:
    if: >-
      github.event.pull_request.merged == true &&
      startsWith(github.event.pull_request.head.ref, 'review/')
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install "reins @ git+https://github.com/mnbiehl/reins@main"
      - run: >-
          reins _review-cleanup
          "${{ github.event.pull_request.head.ref }}"
          "${{ github.event.pull_request.html_url }}"
```

- [ ] **Step 2: Write failing tests**

```python
def test_setup_installs_workflow(submodule_repo, monkeypatch, capsys):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)  # ruleset skipped
    from reins.commands.review import cmd_review_setup
    assert cmd_review_setup() == 0
    wf = submodule_repo / "kb/.github/workflows/reins-doc-review-cleanup.yml"
    assert wf.is_file()
    assert "reins _review-cleanup" in wf.read_text()
    out = capsys.readouterr().out
    assert "ruleset" in out.lower()  # reported as skipped, not silent

def test_setup_idempotent(submodule_repo, monkeypatch):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)
    from reins.commands.review import cmd_review_setup
    cmd_review_setup()
    assert cmd_review_setup() == 0  # unchanged file → success no-op

def test_setup_refuses_clobber_without_force(submodule_repo, monkeypatch, capsys):
    monkeypatch.setattr("reins.github.gh_available", lambda: False)
    from reins.commands.review import cmd_review_setup
    cmd_review_setup()
    wf = submodule_repo / "kb/.github/workflows/reins-doc-review-cleanup.yml"
    wf.write_text(wf.read_text() + "# user edit\n")
    assert cmd_review_setup() == 1
    assert cmd_review_setup(force=True) == 0
```

Also in `tests/test_assets.py`, assert `get_asset_path("workflows/reins-doc-review-cleanup.yml")` resolves.

- [ ] **Step 3: Run to verify failure** — `uv run pytest tests/commands/test_review.py -v -k setup` — Expected: AttributeError/ImportError FAIL

- [ ] **Step 4: Implement `cmd_review_setup`** in `commands/review.py`:

```python
_WORKFLOW_ASSET = "workflows/reins-doc-review-cleanup.yml"
_WORKFLOW_DEST = ".github/workflows/reins-doc-review-cleanup.yml"

_RULESET = {
    "name": "reins-doc-review",
    "target": "branch",
    "enforcement": "active",
    "conditions": {"ref_name": {"include": ["refs/heads/main"], "exclude": []}},
    "rules": [{
        "type": "pull_request",
        "parameters": {
            "required_approving_review_count": 1,
            "dismiss_stale_reviews_on_push": True,
            "require_code_owner_review": False,
            "require_last_push_approval": False,
            "required_review_thread_resolution": False,
            "allowed_merge_methods": ["squash", "merge"],
        },
    }],
    "bypass_actors": [
        # Repo admins + write-role members bypass so direct `kb publish`
        # pushes to main keep working; PR merges still get dismiss-stale.
        {"actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always"},
        {"actor_id": 4, "actor_type": "RepositoryRole", "bypass_mode": "always"},
    ],
}

def cmd_review_setup(force: bool = False) -> int:
    root = repo_root()
    if root is None:
        return 1
    kb_dir = require_kb_dir(root)
    from reins.assets import get_asset_path
    src = get_asset_path(_WORKFLOW_ASSET)
    if src is None:
        console.error(f"bundled asset missing: {_WORKFLOW_ASSET}")
        return 1
    dest = kb_dir / _WORKFLOW_DEST
    template = src.read_text()
    if dest.is_file() and dest.read_text() != template and not force:
        console.error("workflow file was modified — rerun with --force to overwrite")
        return 1
    if not dest.is_file() or dest.read_text() != template:
        dest.parent.mkdir(parents=True, exist_ok=True)
        dest.write_text(template)
        commit_kb(root, "chore(kb): install reins doc-review cleanup workflow",
                  kb_dir=kb_dir)
        console.success(f"workflow installed: kb/{_WORKFLOW_DEST}")
        console.next_step("reins kb publish")
    else:
        console.info("workflow already up to date")

    # Best-effort ruleset (dismiss stale approvals); floor is reins' own checks.
    gh_repo = gh_repo_from_url(kb_remote_url(kb_dir))
    if gh_repo and _gh_ready():
        import json as _json
        r = github.run_gh(
            "api", f"repos/{gh_repo}/rulesets",
            "--method", "POST", "--input", "-",
            check=False, input_text=_json.dumps(_RULESET),
        )
        if r.returncode == 0:
            console.success("dismiss-stale-approvals ruleset applied")
        else:
            console.warn(
                "ruleset not applied (plan/permissions?) — reins' own "
                "divergence check remains the guardrail"
            )
    else:
        console.warn("gh unavailable — ruleset skipped; apply manually if wanted")
    return 0
```

`run_gh` needs stdin support for `--input -`: extend it with an `input_text: str | None = None` kwarg (`kwargs["input"] = input_text` when set) — add a test in `tests/test_github.py`. If a ruleset with the same name already exists, the POST 422s — treat any non-zero as the warn path (idempotent enough; note in the warn when stderr mentions "already exists").

- [ ] **Step 5: Run tests** — `uv run pytest tests/commands/test_review.py tests/test_assets.py tests/test_github.py -v` — Expected: PASS

- [ ] **Step 6: Wheel bundling** — add `workflows/` to the hatchling force-include in `pyproject.toml` alongside the existing asset entries (match the existing syntax exactly), then verify: `uv build 2>/dev/null && unzip -l dist/*.whl | grep doc-review-cleanup` — Expected: the yml is listed. Remove `dist/`.

- [ ] **Step 7: Commit**

```bash
git add workflows/ pyproject.toml src/reins/commands/review.py src/reins/github.py tests/
git commit -m "feat(review): setup verb installs CI cleanup workflow + best-effort ruleset"
```

### Task 11: lint rule — plans referencing unapproved docs

**Files:**
- Create: `src/reins/linter/rules/draft_refs.py`
- Modify: `src/reins/linter/rules/__init__.py` (register in `BUILTIN_RULES`)
- Test: `tests/linter/test_draft_refs.py`

- [ ] **Step 1: Write failing tests** (mirror an existing rule test in `tests/linter/` for fixture style)

```python
from pathlib import Path

from reins.linter.rules.draft_refs import DraftRefsRule

def _kb(tmp_path: Path) -> Path:
    plan_dir = tmp_path / "kb/myrepo/exec-plans/active/feature-x"
    plan_dir.mkdir(parents=True)
    (tmp_path / "kb/myrepo/specs/drafts").mkdir(parents=True)
    (tmp_path / "kb/myrepo/specs").mkdir(exist_ok=True, parents=True)
    return plan_dir

def test_warns_on_drafts_path_reference(tmp_path):
    plan_dir = _kb(tmp_path)
    (plan_dir / "plan.md").write_text(
        "# Plan\n\nSee `kb/myrepo/specs/drafts/wip-spec.md` for details.\n"
    )
    diags = DraftRefsRule().run(tmp_path)
    assert len(diags) == 1 and "drafts" in diags[0]

def test_warns_on_in_review_spec_reference(tmp_path):
    plan_dir = _kb(tmp_path)
    spec = tmp_path / "kb/myrepo/specs/hot.md"
    spec.write_text("# Hot\n\n**Status:** in-review\n")
    (plan_dir / "plan.md").write_text("# Plan\n\nBuilds on `kb/myrepo/specs/hot.md`.\n")
    diags = DraftRefsRule().run(tmp_path)
    assert len(diags) == 1 and "in-review" in diags[0]

def test_legacy_spec_without_status_ok(tmp_path):
    plan_dir = _kb(tmp_path)
    (tmp_path / "kb/myrepo/specs/old.md").write_text("# Old\n\nno status field\n")
    (plan_dir / "plan.md").write_text("# Plan\n\nSee `kb/myrepo/specs/old.md`.\n")
    assert DraftRefsRule().run(tmp_path) == []
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/linter/test_draft_refs.py -v` — Expected: ModuleNotFoundError FAIL

- [ ] **Step 3: Implement** `src/reins/linter/rules/draft_refs.py`:

```python
"""Lint rule: active plans must not build on unapproved (draft/in-review) docs."""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from reins.config import KB_DIR_NAME
from reins.docmeta import FIELD_STATUS, STATUS_IN_REVIEW, get_field
from reins.doc_types import DRAFTS_DIR_NAME, REGISTRY
from reins.linter.rules.base import LintRule

if TYPE_CHECKING:
    from pathlib import Path

_KB_PATH_RE = re.compile(rf"{KB_DIR_NAME}/[\w./-]+\.md")

class DraftRefsRule(LintRule):
    def name(self) -> str:
        return f"{KB_DIR_NAME}/draft-refs"

    def run(self, project_root: Path) -> list[str]:
        diagnostics: list[str] = []
        kb = project_root / KB_DIR_NAME
        if not kb.is_dir():
            return diagnostics
        active_glob = f"*/{REGISTRY['plan'].dir_path}/active/*/plan.md"
        for plan in kb.glob(active_glob):
            rel = plan.relative_to(project_root)
            for n, line in enumerate(plan.read_text().splitlines(), 1):
                for ref in _KB_PATH_RE.findall(line):
                    if f"/{DRAFTS_DIR_NAME}/" in ref:
                        diagnostics.append(
                            f"{rel}:{n} — references drafts-annex doc "
                            f"'{ref}' (unapproved; building on drafts "
                            "needs explicit sign-off)"
                        )
                        continue
                    target = project_root / ref
                    if target.is_file() and \
                            get_field(target.read_text(), FIELD_STATUS) == STATUS_IN_REVIEW:
                        diagnostics.append(
                            f"{rel}:{n} — references in-review doc '{ref}' "
                            "(approval pending)"
                        )
        return diagnostics
```

Register in `src/reins/linter/rules/__init__.py`'s `BUILTIN_RULES` mapping with severity matching the other warning-level rules (copy how `docs_freshness` is registered — warning, not error).

- [ ] **Step 4: Run tests** — `uv run pytest tests/linter/ -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/linter/rules/draft_refs.py src/reins/linter/rules/__init__.py tests/linter/test_draft_refs.py
git commit -m "feat(lint): warn when active plans reference unapproved kb docs"
```

### Task 12: `kb status` "In review" section

**Files:**
- Modify: `src/reins/commands/status.py`
- Test: `tests/commands/test_status.py`

- [ ] **Step 1: Write failing test**

```python
def test_status_shows_in_review_section(status_repo, monkeypatch, capsys):
    # arrange a draft with Review-PR in the fixture kb
    d = status_repo / "kb/testproject/specs/drafts"
    d.mkdir(parents=True, exist_ok=True)
    (d / "hot.md").write_text(
        "# Hot\n\n**Status:** in-review\n"
        "**Review-PR:** https://github.com/o/kb/pull/3\n"
    )
    monkeypatch.chdir(status_repo)
    from reins.commands.status import cmd_status
    assert cmd_status() == 0
    out = capsys.readouterr().out
    assert "In review" in out
    assert "hot" in out and "pull/3" in out
```

(Use the fixture the existing `test_status.py` uses; `status_repo` here stands for it.)

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/commands/test_status.py -v -k in_review` — Expected: FAIL (no "In review" in output)

- [ ] **Step 3: Implement** — in `cmd_status`, after the active-plans block, iterate all repo scopes (same loop shape as the plan_dirs block):

```python
    # In-review / draft gated docs — frontmatter only, no gh calls
    from reins.docmeta import FIELD_REVIEW_PR, FIELD_STATUS, STATUS_DRAFT, get_field
    from reins.doc_types import DRAFTS_DIR_NAME, gated_types
    reviews = []
    for repo_dir in sorted(kb_dir.iterdir()):
        if not repo_dir.is_dir() or repo_dir.name.startswith((".", "_")):
            continue
        for dt in gated_types():
            d = repo_dir / dt.dir_path / DRAFTS_DIR_NAME
            if not d.is_dir():
                continue
            for f in sorted(d.glob("*.md")):
                text = f.read_text()
                reviews.append((
                    repo_dir.name, dt.key, f.stem,
                    get_field(text, FIELD_STATUS) or STATUS_DRAFT,
                    get_field(text, FIELD_REVIEW_PR) or "",
                ))
    if reviews:
        print()
        console.info(f"In review / drafts: {len(reviews)}")
        for scope, key, slug, status, url in reviews:
            line = f"  {scope}/{key}/{slug} [{status}]"
            if url:
                line += f" {url}"
            console.info(line)
```

Keep it out of `--compact` (the ≤10-line budget) except a single count line if trivially cheap — follow whatever `_compact_status` does for plans.

- [ ] **Step 4: Run tests** — `uv run pytest tests/commands/test_status.py -v` — Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reins/commands/status.py tests/commands/test_status.py
git commit -m "feat(status): in-review drafts section (frontmatter-only)"
```

### Task 13: skill gates + spec ref-format amendment + docs

**Files:**
- Modify: `.claude/skills/writing-plans/SKILL.md`, `.claude/skills/executing-plans/SKILL.md`
- Modify: `kb/reins/specs/doc-review-lane-pr-style-review-for-gated-kb-docs.md` (via edit — it's an existing file, hooks allow edits)
- Modify: `README.md` (kb section: drafts/ line in the tree + one paragraph)

- [ ] **Step 1: Skill gates** — in `writing-plans/SKILL.md`, add to the Reins Integration section:

```markdown
### Gated docs (review lane)

Before building a plan on a spec, check its status: a path under
`specs/drafts/` or a `**Status:** draft|in-review` header means the spec is
NOT approved. Stop and ask the user for explicit confirmation before planning
against an unapproved spec (`reins review status` shows open reviews).
```

Add the equivalent two-liner to `executing-plans/SKILL.md` (check before executing).

- [ ] **Step 2: Spec amendment** — in the spec's Design section, update the ref format sentence to match the implementation: branch `review/<repo-scope>/<type>-<slug>` (repo-scoped to prevent cross-repo slug collisions in a shared kb). One-line edit where `review/spec-<slug>` appears.

- [ ] **Step 3: README** — in the Repository Structure tree, under `specs/`, add the drafts line; in the kb section, add a short "Doc review" paragraph pointing at `reins review start` and the spec.

- [ ] **Step 4: Publish kb + commit**

```bash
uv run reins kb publish
git add kb .claude/skills README.md
git commit -m "docs: review-lane skill gates, spec ref-format amendment, README"
```

### Task 14: full verification + live acceptance run

- [ ] **Step 1: Full suite** — `uv run pytest` — Expected: all pass
- [ ] **Step 2: Linters** — `uv run ruff check && uv run pyright` — Expected: clean
- [ ] **Step 3: kb lint** — `uv run reins kb lint` — Expected: no new failures beyond the pre-existing three (docs-freshness, plan-structure, provenance)
- [ ] **Step 4: Live acceptance run** (the spec's scripted end-to-end; requires gh auth, run with the user):

```bash
uv run reins spec create "Acceptance test doc review lane"
uv run reins kb publish
uv run reins review start acceptance-test-doc-review-lane
# → PR URL printed; approve it in the GitHub UI (self-review or second account)
# → merge from the UI; watch the Actions run perform cleanup
uv run reins kb sync
uv run reins spec list        # doc listed as approved at specs/
uv run reins review status    # 0 open
```

Then delete the test spec (`reins kb git rm ...` + publish) and verify the workflow run is green. If the kb repo plan doesn't support rulesets, confirm the warn path printed during `review setup`.

- [ ] **Step 5: Update plan Status to `complete`, commit, and hand off** to finishing-a-development-branch.

## Dependencies

- Spec: `kb/reins/specs/doc-review-lane-pr-style-review-for-gated-kb-docs.md` (approved by Michael in-session, 2026-07-06; predates the lane itself).
- No overlap with other active branches (`gh-pr-crawl` is a dead plan dir).
- The unpushed beta-cleanup branch (other machine) touches hooks/skills layout (`.agents/skills/`); this plan only appends to skill files — merge order with beta-cleanup is flexible, conflicts are additive text.
