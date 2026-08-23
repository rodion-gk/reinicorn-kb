---
type: plan
title: Skills Fork + Template-Driven Docs — Implementation Plan
slug: skills-fork-template-docs
lifecycle: done
status: done
created: 2026-07-17
author: Michael Biehl
branch: skills-fork-template-docs
---

# Skills Fork + Template-Driven Docs — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fork superpowers skills into reins, build `reins doc create` CLI for template-driven doc creation, migrate harness to repo-scoped paths, and add agent enforcement hooks — so agents never hand-write harness docs.

**Architecture:** Add a `repo_slug()` helper that derives the repo name from `git remote origin`. All harness doc paths become `harness/{repo}/...`. A new `doc_create` command module generates templates with provenance frontmatter. A PreToolUse Claude Code hook blocks direct creation of harness docs, forcing agents through the CLI. Superpowers skills are forked, customized to reference `reins doc create`, and deduplicated with existing reins skills.

**Tech Stack:** Python 3.13+, argparse, pathlib, pytest, bash (hook script), Claude Code hooks API

---

### Task 1: Add `repo_slug()` helper to `git.py`

**Files:**
- Modify: `src/reins/git.py:68-70`
- Test: `tests/test_git.py`

**Step 1: Write the failing tests**

Create `tests/test_git.py`:

```python
"""Tests for reins.git helpers."""

from __future__ import annotations

from unittest.mock import patch
import subprocess

from reins.git import repo_slug

def test_repo_slug_from_ssh_url():
    with patch("reins.git.run_git") as mock:
        mock.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="git@github.com:mnbiehl/reins.git\n"
        )
        assert repo_slug() == "reins"

def test_repo_slug_from_https_url():
    with patch("reins.git.run_git") as mock:
        mock.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="https://github.com/mnbiehl/reins.git\n"
        )
        assert repo_slug() == "reins"

def test_repo_slug_strips_trailing_git():
    with patch("reins.git.run_git") as mock:
        mock.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="git@github.com:your-org/my-project.git\n"
        )
        assert repo_slug() == "my-project"

def test_repo_slug_no_git_suffix():
    with patch("reins.git.run_git") as mock:
        mock.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="https://github.com/your-org/foo\n"
        )
        assert repo_slug() == "foo"

def test_repo_slug_fallback_on_error():
    with patch("reins.git.run_git", side_effect=Exception("no remote")):
        assert repo_slug() == "unknown"
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_git.py -v`
Expected: FAIL — `ImportError: cannot import name 'repo_slug'`

**Step 3: Write minimal implementation**

Add to `src/reins/git.py` after `reins_root()` (after line 70):

```python
def repo_slug() -> str:
    """Derive the repo name from the git remote origin URL.

    Examples:
        git@github.com:mnbiehl/reins.git → "reins"
        https://github.com/mnbiehl/reins.git → "reins"

    Returns "unknown" if no remote is configured.
    """
    try:
        r = run_git("remote", "get-url", "origin", check=False)
        url = r.stdout.strip()
        if not url:
            return "unknown"
        # Strip trailing .git
        if url.endswith(".git"):
            url = url[:-4]
        # Take the last path component
        return url.rstrip("/").rsplit("/", 1)[-1].rsplit(":", 1)[-1]
    except Exception:
        return "unknown"
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_git.py -v`
Expected: All 5 tests PASS.

**Step 5: Commit**

```bash
git add src/reins/git.py tests/test_git.py
git commit -m "feat(git): add repo_slug() helper for repo-scoped paths"
```

---

### Task 2: Add `repo_harness_dir()` helper to `harness.py`

**Files:**
- Modify: `src/reins/harness.py:113-114`
- Test: `tests/test_harness.py`

**Step 1: Write the failing test**

Add to `tests/test_harness.py` (create if missing):

```python
"""Tests for reins.harness helpers."""

from __future__ import annotations

from pathlib import Path
from unittest.mock import patch

from reins.harness import repo_harness_dir

def test_repo_harness_dir_creates_directory(harness_repo: Path):
    with patch("reins.harness.repo_slug", return_value="myproject"):
        result = repo_harness_dir(harness_repo / "harness")
    assert result == harness_repo / "harness" / "myproject"
    assert result.is_dir()

def test_repo_harness_dir_is_idempotent(harness_repo: Path):
    with patch("reins.harness.repo_slug", return_value="myproject"):
        first = repo_harness_dir(harness_repo / "harness")
        second = repo_harness_dir(harness_repo / "harness")
    assert first == second
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_harness.py::test_repo_harness_dir_creates_directory -v`
Expected: FAIL — `ImportError: cannot import name 'repo_harness_dir'`

**Step 3: Write minimal implementation**

Add to `src/reins/harness.py` after `plan_dir()` (after line 114), and add the import:

```python
from reins.git import repo_slug  # add to existing imports from reins.git
```

```python
def repo_harness_dir(harness_dir: Path) -> Path:
    """Return the repo-scoped subdirectory inside the harness.

    Creates it if it doesn't exist. Path: harness/{repo_slug}/
    """
    slug = repo_slug()
    repo_dir = harness_dir / slug
    repo_dir.mkdir(parents=True, exist_ok=True)
    return repo_dir
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_harness.py -v`
Expected: All tests PASS.

**Step 5: Commit**

```bash
git add src/reins/harness.py tests/test_harness.py
git commit -m "feat(harness): add repo_harness_dir() for repo-scoped paths"
```

---

### Task 3: Migrate existing harness docs to repo-scoped paths

**Files:**
- Modify: `harness/` (move directories)

This task moves existing docs from `harness/X/` to `harness/reins/X/`. This is a harness-only change (submodule).

**Step 1: Create the repo-scoped directory and move content**

Run inside the harness submodule:

```bash
cd harness
mkdir -p reins

# Move all repo-specific directories
git mv architecture reins/architecture
git mv design-docs reins/design-docs
git mv exec-plans reins/exec-plans
git mv ideas reins/ideas
git mv product-specs reins/product-specs
git mv tech-debt reins/tech-debt
git mv golden-principles.md reins/golden-principles.md
git mv quality-scores.md reins/quality-scores.md
git mv DESIGN.md reins/DESIGN.md
git mv references reins/references
```

Keep at harness root: `agent-tasks/`, `generated/`, `ideas/index.md` (shared index).

**Step 2: Update cross-references within moved files**

All relative links within the moved files need updating. Files that reference siblings (e.g., `design-docs/index.md` linking to `core-beliefs.md`) still work because they moved together. But files referencing across directories need fixes:

- `reins/design-docs/index.md`: relative links to sibling docs should still work
- `reins/exec-plans/` files referencing `design-docs/`: update to `../design-docs/`
- `reins/golden-principles.md` references: update if any
- Any absolute-style references like `harness/design-docs/` in plan.md files: updated to `harness/reins/design-docs/`

Use search-and-replace across moved files. Key patterns:
- `harness/design-docs/` → `harness/reins/design-docs/` (done)
- `harness/exec-plans/` → `harness/reins/exec-plans/` (done)
- `harness/golden-principles.md` → `harness/reins/golden-principles.md` (done)
- `harness/architecture/` → `harness/reins/architecture/` (done)
- `harness/tech-debt/` → `harness/reins/tech-debt/` (done)
- `harness/product-specs/` → `harness/reins/product-specs/` (done)
- `harness/quality-scores.md` → `harness/reins/quality-scores.md` (done)
- `harness/ideas/` → `harness/reins/ideas/` (done)

**Step 3: Commit harness changes**

```bash
cd harness
git add -A
git commit -m "refactor: migrate docs to repo-scoped paths (harness/reins/)"
```

**Step 4: Update parent repo pointer**

```bash
cd ..
git add harness
```

---

### Task 4: Update AGENTS.md path table

**Files:**
- Modify: `AGENTS.md`

**Step 1: Update the knowledge base path table**

Change lines 12-20 of AGENTS.md:

```markdown
| What | Where |
|------|-------|
| Architecture & domains | `harness/{repo}/architecture/ARCHITECTURE.md` |
| Dependency rules | `harness/{repo}/architecture/dependency-rules.md` |
| Golden principles | `harness/{repo}/golden-principles.md` |
| Design documents | `harness/{repo}/design-docs/index.md` |
| Active execution plans | `harness/{repo}/exec-plans/active/` |
| Tech debt | `harness/{repo}/tech-debt/index.md` |
| Quality scores | `harness/{repo}/quality-scores.md` |
| Ideas | `harness/{repo}/ideas/index.md` |
| Product specs | `harness/{repo}/product-specs/index.md` |
```

Where `{repo}` is derived from `git remote origin` (e.g., `reins`).

**Step 2: Add `reins doc create` to the CLI table**

Add row to the CLI table:

```markdown
| `reins doc create <type> "title"` | Create a harness doc from template |
```

**Step 3: Update Hard Rules section**

Update paths in Hard Rules to use `{repo}` prefix:

- Rule 1: `harness/{repo}/exec-plans/active/{branch}/plan.md`
- Rule 3: `harness/{repo}/exec-plans/active/{branch}/touched-areas.md`
- Rule 4: `harness/{repo}/golden-principles.md`

**Step 4: Commit**

```bash
git add AGENTS.md
git commit -m "docs(agents): update paths for repo-scoped harness layout"
```

---

### Task 5: Update linter rules for repo-scoped paths

**Files:**
- Modify: `src/reins/linter/rules/docs_freshness.py:13-23`
- Modify: `src/reins/linter/rules/plan_structure.py:18-19`
- Modify: `src/reins/linter/rules/cross_links.py:25-28`
- Modify: `linters/rules/harness/docs-freshness.sh:24-33`
- Modify: `linters/rules/harness/plan-structure.sh:13`
- Test: existing lint tests should still pass

**Step 1: Update Python linter — docs_freshness.py**

Replace hardcoded `KEY_DOCS` list (lines 13-23) to use dynamic repo discovery:

```python
def _key_docs(project_root: Path) -> list[str]:
    """Build KEY_DOCS list dynamically based on repo-scoped harness dirs."""
    harness = project_root / "harness"
    if not harness.is_dir():
        return ["AGENTS.md"]

    docs = ["AGENTS.md"]
    for repo_dir in sorted(harness.iterdir()):
        if not repo_dir.is_dir() or repo_dir.name.startswith((".", "_")):
            continue
        prefix = f"harness/{repo_dir.name}"
        docs.extend([
            f"{prefix}/architecture/ARCHITECTURE.md",
            f"{prefix}/architecture/dependency-rules.md",
            f"{prefix}/golden-principles.md",
            f"{prefix}/quality-scores.md",
            f"{prefix}/tech-debt/index.md",
            f"{prefix}/design-docs/index.md",
            f"{prefix}/product-specs/index.md",
        ])
        design_md = repo_dir / "DESIGN.md"
        if design_md.is_file():
            docs.append(f"{prefix}/DESIGN.md")

    return docs
```

Update `run()` to call `_key_docs(project_root)` instead of using the module-level constant.

**Step 2: Update Python linter — plan_structure.py**

Change line 19 to discover repo-scoped plan directories:

```python
def run(self, project_root: Path) -> list[str]:
    harness = project_root / "harness"
    if not harness.is_dir():
        return []

    diagnostics: list[str] = []

    # Check all repo-scoped active dirs
    for repo_dir in sorted(harness.iterdir()):
        if not repo_dir.is_dir() or repo_dir.name.startswith((".", "_")):
            continue
        active_dir = repo_dir / "exec-plans" / "active"
        if not active_dir.is_dir():
            continue

        for plan_dir in sorted(active_dir.iterdir()):
            # ... same validation logic, but with updated rel paths
            rel_prefix = f"harness/{repo_dir.name}/exec-plans/active/{plan_dir.name}"
            # ...
```

**Step 3: Update shell linters similarly**

In `docs-freshness.sh`, replace the static `KEY_DOCS` array with dynamic discovery:

```bash
# Discover repo-scoped directories
KEY_DOCS=("AGENTS.md")
for repo_dir in "${PROJECT_ROOT}"/harness/*/; do
  [ -d "$repo_dir" ] || continue
  name=$(basename "$repo_dir")
  [[ "$name" == .* || "$name" == _* ]] && continue
  KEY_DOCS+=(
    "harness/${name}/architecture/ARCHITECTURE.md"
    "harness/${name}/architecture/dependency-rules.md"
    "harness/${name}/golden-principles.md"
    "harness/${name}/quality-scores.md"
    "harness/${name}/tech-debt/index.md"
    "harness/${name}/design-docs/index.md"
    "harness/${name}/product-specs/index.md"
  )
  [ -f "${PROJECT_ROOT}/harness/${name}/DESIGN.md" ] && KEY_DOCS+=("harness/${name}/DESIGN.md")
done
```

In `plan-structure.sh`, change line 13 to iterate over repo dirs:

```bash
for repo_dir in "${PROJECT_ROOT}"/harness/*/; do
  [ -d "$repo_dir" ] || continue
  name=$(basename "$repo_dir")
  [[ "$name" == .* || "$name" == _* ]] && continue
  ACTIVE_DIR="${repo_dir}exec-plans/active"
  [ -d "$ACTIVE_DIR" ] || continue
  # ... same plan checking logic with updated paths
done
```

**Step 4: Run all linter tests**

Run: `uv run pytest tests/ -v -k lint`
Expected: All PASS.

**Step 5: Run linters against the repo**

Run: `uv run reins lint`
Expected: No errors (warnings acceptable).

**Step 6: Commit**

```bash
git add src/reins/linter/ linters/rules/
git commit -m "fix(linters): support repo-scoped harness paths"
```

---

### Task 6: Update `idea.py` to use repo-scoped paths

**Files:**
- Modify: `src/reins/commands/idea.py:33`
- Test: `tests/commands/test_idea.py`

**Step 1: Write the failing test**

Add to `tests/commands/test_idea.py`:

```python
def test_idea_uses_repo_scoped_path(harness_repo: Path, capsys):
    with patch("reins.commands.idea.repo_root", return_value=harness_repo), \
         patch("reins.commands.idea.run_git") as mock_git, \
         patch("reins.commands.idea.commit_harness") as mock_commit, \
         patch("reins.commands.idea.repo_slug", return_value="myproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_idea("test repo scoping")

    assert result == 0
    ideas_dir = harness_repo / "harness" / "myproject" / "ideas" / "test-user"
    assert ideas_dir.is_dir()
    files = list(ideas_dir.glob("*.md"))
    assert len(files) == 1
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/commands/test_idea.py::test_idea_uses_repo_scoped_path -v`
Expected: FAIL — file created at `harness/reins/ideas/` not `harness/myproject/ideas/`

**Step 3: Update idea.py**

Change line 33 from:

```python
    idea_dir = harness_dir / "ideas" / username
```

To:

```python
    from reins.git import repo_slug
    idea_dir = harness_dir / repo_slug() / "ideas" / username
```

**Step 4: Run all idea tests**

Run: `uv run pytest tests/commands/test_idea.py -v`
Expected: All PASS (update existing test to also patch `repo_slug`).

**Step 5: Commit**

```bash
git add src/reins/commands/idea.py tests/commands/test_idea.py
git commit -m "feat(idea): use repo-scoped path (harness/{repo}/ideas/)"
```

---

### Task 7: Update `plan.py` to use repo-scoped paths

**Files:**
- Modify: `src/reins/commands/plan.py`
- Modify: `src/reins/harness.py:113-114` (update `plan_dir()`)
- Test: existing plan tests

**Step 1: Update `plan_dir()` in harness.py**

Change `plan_dir()` (line 113-114) from:

```python
def plan_dir(harness: Path, branch: str) -> Path:
    return harness / "exec-plans" / "active" / branch
```

To:

```python
def plan_dir(harness: Path, branch: str) -> Path:
    from reins.git import repo_slug
    return harness / repo_slug() / "exec-plans" / "active" / branch
```

**Step 2: Update `check_overlap()` similarly**

Change line 128 from:

```python
    active_dir = h_dir / "exec-plans" / "active"
```

To use repo-scoped path. Since overlap should check across all repos:

```python
    # Check all repo-scoped active dirs for overlap
    found_overlap = False
    for repo_dir in sorted(h_dir.iterdir()):
        if not repo_dir.is_dir() or repo_dir.name.startswith((".", "_")):
            continue
        active_dir = repo_dir / "exec-plans" / "active"
        if not active_dir.is_dir():
            continue
        # ... existing overlap logic
```

**Step 3: Run plan tests**

Run: `uv run pytest tests/commands/test_plan.py -v`
Expected: All PASS (may need to update test fixtures to create repo-scoped dirs).

**Step 4: Commit**

```bash
git add src/reins/harness.py src/reins/commands/plan.py tests/
git commit -m "feat(plan): use repo-scoped exec-plan paths"
```

---

### Task 8: Update remaining commands for repo-scoped paths

**Files:**
- Modify: `src/reins/commands/status.py`
- Modify: `src/reins/commands/internal/post_checkout.py`
- Modify: `src/reins/commands/internal/pre_push.py`
- Modify: `src/reins/commands/internal/post_merge.py`

**Step 1: Audit all harness path references**

Search for hardcoded `harness/` subdir references in all command files:

```
grep -rn "exec-plans\|design-docs\|golden-principles\|ideas\|tech-debt\|product-specs\|quality-scores\|architecture" src/reins/commands/
```

**Step 2: Update each reference to go through `repo_harness_dir()` or `repo_slug()`**

The pattern for each command: replace `harness_dir / "some-subdir"` with `repo_harness_dir(harness_dir) / "some-subdir"`.

**Step 3: Run full test suite**

Run: `uv run pytest tests/ -v`
Expected: All PASS.

**Step 4: Commit**

```bash
git add src/reins/commands/
git commit -m "refactor(commands): use repo-scoped harness paths throughout"
```

---

### Task 9: Update `conftest.py` fixtures for repo-scoped layout

**Files:**
- Modify: `tests/conftest.py:41-65`

**Step 1: Update `harness_repo` fixture**

The fixture needs to create the repo-scoped directory structure. Change the minimal harness structure (lines 41-65) to include a repo subdirectory:

```python
    harness = repo / "harness"
    harness.mkdir()

    # Repo-scoped structure
    repo_sub = harness / "testproject"
    repo_sub.mkdir()
    (repo_sub / "exec-plans").mkdir()
    (repo_sub / "exec-plans" / "active").mkdir()
    (repo_sub / "exec-plans" / "_template").mkdir()

    # Template files
    (repo_sub / "exec-plans" / "_template" / "plan.md").write_text(...)
    # etc.
```

Also patch `repo_slug` to return `"testproject"` in fixtures or add it to conftest.

**Step 2: Run full test suite**

Run: `uv run pytest tests/ -v`
Expected: All PASS.

**Step 3: Commit**

```bash
git add tests/conftest.py
git commit -m "test(fixtures): update harness_repo for repo-scoped layout"
```

---

### Task 10: Build `reins doc create` command — core module

**Files:**
- Create: `src/reins/commands/doc_create.py`
- Test: `tests/commands/test_doc_create.py`

**Step 1: Write the failing tests**

Create `tests/commands/test_doc_create.py`:

```python
"""Tests for reins doc create command."""

from __future__ import annotations

import subprocess
from pathlib import Path
from unittest.mock import patch

from reins.commands.doc_create import cmd_doc_create

def test_create_design_doc(harness_repo: Path):
    with patch("reins.commands.doc_create.repo_root", return_value=harness_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_harness") as mock_commit, \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_doc_create("design", "My Cool Feature")

    assert result == 0
    doc = harness_repo / "harness" / "testproject" / "design-docs" / "my-cool-feature.md"
    assert doc.is_file()
    content = doc.read_text()
    assert "# My Cool Feature" in content
    assert "**Status:** draft" in content
    assert "**Author:** Test User" in content
    assert "**Origin:** ai-assisted" in content
    assert "**Human-validated:** false" in content
    assert "## Problem" in content
    assert "## Design Goals" in content
    assert "## Non-Goals" in content
    mock_commit.assert_called_once()

def test_create_spec_doc(harness_repo: Path):
    with patch("reins.commands.doc_create.repo_root", return_value=harness_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_harness") as mock_commit, \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_doc_create("spec", "User Authentication")

    assert result == 0
    doc = harness_repo / "harness" / "testproject" / "product-specs" / "user-authentication.md"
    assert doc.is_file()
    content = doc.read_text()
    assert "## User Stories" in content
    assert "## Acceptance Criteria" in content
    assert "## Out of Scope" in content

def test_create_debt_doc(harness_repo: Path):
    with patch("reins.commands.doc_create.repo_root", return_value=harness_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_harness") as mock_commit, \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_doc_create("debt", "Missing validation in API layer")

    assert result == 0
    doc = harness_repo / "harness" / "testproject" / "tech-debt" / "by-domain" / "missing-validation-in-api-layer.md"
    assert doc.is_file()
    content = doc.read_text()
    assert "**Severity:**" in content
    assert "**Remediation:**" in content

def test_create_retro_doc(harness_repo: Path):
    with patch("reins.commands.doc_create.repo_root", return_value=harness_repo), \
         patch("reins.commands.doc_create.run_git") as mock_git, \
         patch("reins.commands.doc_create.commit_harness") as mock_commit, \
         patch("reins.commands.doc_create.repo_slug", return_value="testproject"), \
         patch("reins.commands.doc_create.current_branch", return_value="feature/cool-feature"):
        mock_git.return_value = subprocess.CompletedProcess(
            args=[], returncode=0, stdout="Test User\n"
        )
        result = cmd_doc_create("retro", "Cool Feature Retro")

    assert result == 0
    doc = harness_repo / "harness" / "testproject" / "exec-plans" / "completed" / "feature/cool-feature" / "retro.md"
    assert doc.is_file()
    content = doc.read_text()
    assert "## What Worked" in content
    assert "## What Didn't" in content

def test_create_idea_delegates_to_idea_command(harness_repo: Path):
    with patch("reins.commands.doc_create.cmd_idea") as mock_idea:
        mock_idea.return_value = 0
        result = cmd_doc_create("idea", "some cool idea")

    assert result == 0
    mock_idea.assert_called_once_with("some cool idea")

def test_create_invalid_type():
    result = cmd_doc_create("invalid", "test")
    assert result == 1

def test_create_empty_title():
    result = cmd_doc_create("design", "")
    assert result == 1
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/test_doc_create.py -v`
Expected: FAIL — `ModuleNotFoundError`

**Step 3: Write the implementation**

Create `src/reins/commands/doc_create.py`:

```python
"""reins doc create — template-driven harness doc creation."""

from __future__ import annotations

import re
from datetime import date

from reins import console
from reins.git import current_branch, repo_root, repo_slug, run_git
from reins.harness import commit_harness, require_harness_dir

_DOC_TYPES = {"design", "plan", "idea", "spec", "debt", "retro", "principle"}

def _get_author() -> str:
    try:
        return run_git("config", "user.name").stdout.strip()
    except Exception:
        return "unknown"

def _slugify(text: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", text.lower())[:60].rstrip("-")

def _provenance(title: str, author: str, status: str = "draft") -> str:
    return (
        f"# {title}\n"
        f"\n"
        f"**Date:** {date.today().isoformat()}\n"
        f"**Author:** {author}\n"
        f"**Status:** {status}\n"
        f"**Origin:** ai-assisted\n"
        f"**Human-validated:** false\n"
    )

def _create_design(repo_dir: Path, title: str, author: str) -> Path:
    slug = _slugify(title)
    target = repo_dir / "design-docs" / f"{slug}.md"
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(
        _provenance(title, author)
        + "\n## Problem\n\n_Describe the problem._\n"
        "\n## Design Goals\n\n_What must be true when this is done._\n"
        "\n## Design\n\n_How it works._\n"
        "\n## Non-Goals\n\n_What this explicitly does not cover._\n"
    )
    return target

def _create_plan(repo_dir: Path, title: str, author: str) -> Path:
    branch = current_branch() or "unknown"
    target_dir = repo_dir / "exec-plans" / "active" / branch
    target_dir.mkdir(parents=True, exist_ok=True)
    plan_file = target_dir / "plan.md"
    plan_file.write_text(
        _provenance(title, author)
        + "\n## Goal\n\n_What this branch is building or fixing._\n"
        "\n## Acceptance Criteria\n\n- [ ] _Criterion 1_\n"
        "\n## Tasks\n\n- [ ] _Task 1_\n"
    )
    # Also create touched-areas.md
    (target_dir / "touched-areas.md").write_text("# Touched Areas\n")
    (target_dir / "progress.md").write_text("# Progress\n")
    (target_dir / "decisions.md").write_text("# Decisions\n")
    return plan_file

def _create_spec(repo_dir: Path, title: str, author: str) -> Path:
    slug = _slugify(title)
    target = repo_dir / "product-specs" / f"{slug}.md"
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(
        _provenance(title, author)
        + "\n## Overview\n\n_One-paragraph summary._\n"
        "\n## User Stories\n\n- As a [role], I want [goal] so that [benefit].\n"
        "\n## Acceptance Criteria\n\n- [ ] _Criterion 1_\n"
        "\n## Out of Scope\n\n_What this spec explicitly does not cover._\n"
        "\n## Open Questions\n\n_Unresolved decisions._\n"
    )
    return target

def _create_debt(repo_dir: Path, title: str, author: str) -> Path:
    slug = _slugify(title)
    target = repo_dir / "tech-debt" / "by-domain" / f"{slug}.md"
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(
        _provenance(title, author)
        + "\n**Severity:** medium\n"
        "**Domain:** _domain_\n"
        "\n## Impact\n\n_What this debt causes._\n"
        "\n## Remediation\n\n_How to fix it._\n"
    )
    return target

def _create_retro(repo_dir: Path, title: str, author: str) -> Path:
    branch = current_branch() or "unknown"
    target_dir = repo_dir / "exec-plans" / "completed" / branch
    target_dir.mkdir(parents=True, exist_ok=True)
    target = target_dir / "retro.md"
    target.write_text(
        _provenance(title, author)
        + "\n## Outcome Assessment\n\n_Did we achieve what we set out to do?_\n"
        "\n## What Worked\n\n- _Item_\n"
        "\n## What Didn't\n\n- _Item_\n"
        "\n## Learnings\n\n- _Lesson_\n"
        "\n## Follow-ups\n\n- [ ] _Action item_\n"
    )
    return target

def _create_principle(repo_dir: Path, title: str, author: str) -> Path:
    target = repo_dir / "golden-principles.md"
    if not target.is_file():
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text("# Golden Principles\n\n")

    # Append new principle
    content = target.read_text()
    # Count existing principles to number the new one
    import re as _re
    existing = _re.findall(r'^\d+\.', content, _re.MULTILINE)
    num = len(existing) + 1

    target.write_text(
        content.rstrip()
        + f"\n\n{num}. **{title}**\n"
        f"   - _Rule description_\n"
        f"   - Prevents: _What this rule prevents_\n"
    )
    return target

_CREATORS = {
    "design": _create_design,
    "plan": _create_plan,
    "spec": _create_spec,
    "debt": _create_debt,
    "retro": _create_retro,
    "principle": _create_principle,
}

def cmd_doc_create(doc_type: str, title: str) -> int:
    if not title.strip():
        console.error("Title is required.")
        return 1

    if doc_type not in _DOC_TYPES:
        console.error(
            f"Unknown doc type '{doc_type}'. "
            f"Valid types: {', '.join(sorted(_DOC_TYPES))}"
        )
        return 1

    # Delegate idea to existing command
    if doc_type == "idea":
        from reins.commands.idea import cmd_idea
        return cmd_idea(title)

    root = repo_root()
    if root is None:
        return 1
    harness_dir = require_harness_dir(root)

    slug = repo_slug()
    repo_dir = harness_dir / slug
    repo_dir.mkdir(parents=True, exist_ok=True)

    author = _get_author()

    creator = _CREATORS[doc_type]
    filepath = creator(repo_dir, title, author)

    console.success(f"Created: {filepath}")
    commit_harness(root, f"doc({doc_type}): {_slugify(title)}")

    return 0
```

Add the missing import at the top:

```python
from pathlib import Path
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/commands/test_doc_create.py -v`
Expected: All 7 tests PASS.

**Step 5: Commit**

```bash
git add src/reins/commands/doc_create.py tests/commands/test_doc_create.py
git commit -m "feat(doc): add reins doc create command with templates"
```

---

### Task 11: Wire `doc create` into CLI

**Files:**
- Modify: `src/reins/cli.py:19-68` and `src/reins/cli.py:75-151`

**Step 1: Add the subparser (in `_build_parser()`)**

After the `idea` subparser (line 40), add:

```python
    # Doc creation
    doc_p = sub.add_parser("doc", help="Harness document management")
    doc_sub = doc_p.add_subparsers(dest="doc_command")
    doc_sub.required = True
    doc_create_p = doc_sub.add_parser(
        "create", help="Create a harness doc from template"
    )
    doc_create_p.add_argument(
        "doc_type",
        choices=["design", "plan", "idea", "spec", "debt", "retro", "principle"],
        help="Document type",
    )
    doc_create_p.add_argument("title", nargs="+", help="Document title")
```

**Step 2: Add the dispatch (in `_dispatch()`)**

After the `idea` dispatch (line 108), add:

```python
    if cmd == "doc":
        dc = args.doc_command
        if dc == "create":
            from reins.commands.doc_create import cmd_doc_create
            return cmd_doc_create(args.doc_type, " ".join(args.title))
```

**Step 3: Test CLI integration**

Run: `uv run reins doc create design "Test Design" --help` (verify argparse works)
Run: `uv run pytest tests/ -v`
Expected: All PASS.

**Step 4: Commit**

```bash
git add src/reins/cli.py
git commit -m "feat(cli): wire up reins doc create subcommand"
```

---

### Task 12: Add agent enforcement hook

**Files:**
- Create: `.claude/hooks/enforce-doc-templates.sh`
- Modify: `.claude/settings.json` (create if missing)

**Step 1: Write the hook script**

Create `.claude/hooks/enforce-doc-templates.sh`:

```bash
#!/usr/bin/env bash
# enforce-doc-templates.sh — PreToolUse hook for Write|Edit
#
# Blocks direct creation of new harness doc files, forcing agents to use
# 'reins doc create <type> "title"' instead.
# Allows editing existing files (filling in template sections).

INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_params.file_path // .tool_params.filePath // empty')

[ -z "$FILE" ] && exit 0

# Normalize: only check harness doc paths
case "$FILE" in
  */harness/*/design-docs/*.md | \
  */harness/*/product-specs/*.md | \
  */harness/*/exec-plans/*/plan.md | \
  */harness/*/exec-plans/*/retro.md | \
  */harness/*/tech-debt/*.md)
    # Allow edits to existing files (template fill-in)
    [ -f "$FILE" ] && exit 0
    echo "BLOCKED: Use 'reins doc create <type> \"title\"' instead of writing harness docs directly."
    echo "Available types: design, plan, spec, debt, retro, principle"
    echo "Example: reins doc create design \"My Feature\""
    exit 2
    ;;
  */harness/*/ideas/*.md)
    [ -f "$FILE" ] && exit 0
    echo "BLOCKED: Use 'reins idea \"your idea\"' or 'reins doc create idea \"title\"'."
    exit 2
    ;;
esac

exit 0
```

**Step 2: Make it executable**

```bash
chmod +x .claude/hooks/enforce-doc-templates.sh
```

**Step 3: Create `.claude/settings.json`**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "command": ".claude/hooks/enforce-doc-templates.sh"
      }
    ]
  }
}
```

**Step 4: Test the hook manually**

```bash
echo '{"tool_params":{"file_path":"~/dev/reins/harness/reins/design-docs/nonexistent.md"}}' | .claude/hooks/enforce-doc-templates.sh
# Expected: exit 2, "BLOCKED" message

echo '{"tool_params":{"file_path":"~/dev/reins/harness/reins/design-docs/skills-fork-and-template-docs.md"}}' | .claude/hooks/enforce-doc-templates.sh
# Expected: exit 0 (file exists, editing allowed)

echo '{"tool_params":{"file_path":"~/dev/reins/src/reins/cli.py"}}' | .claude/hooks/enforce-doc-templates.sh
# Expected: exit 0 (not a harness doc path)
```

**Step 5: Commit**

```bash
git add .claude/hooks/enforce-doc-templates.sh .claude/settings.json
git commit -m "feat(hooks): add agent enforcement hook for doc templates"
```

---

### Task 13: Fork superpowers skills into reins

**Files:**
- Create: `.claude/skills/` (copy all superpowers skills)
- Create: `.claude/skills/ATTRIBUTION.md`

**Step 1: Copy all superpowers skills**

```bash
# Remove existing reins skills that will be replaced
rm .claude/skills/capture-idea.md

# Copy superpowers skills
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/brainstorming .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/writing-plans .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/executing-plans .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/test-driven-development .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/systematic-debugging .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/verification-before-completion .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/requesting-code-review .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/receiving-code-review .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/dispatching-parallel-agents .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/subagent-driven-development .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/finishing-a-development-branch .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/using-git-worktrees .claude/skills/
cp -r ~/.claude/plugins/cache/claude-plugins-official/superpowers/4.3.1/skills/writing-skills .claude/skills/
```

**Step 2: Create attribution file**

Create `.claude/skills/ATTRIBUTION.md`:

```markdown
# Skills Attribution

Several skills in this directory are forked from the
[superpowers](https://github.com/anthropics/superpowers) Claude Code plugin
by Jesse Vincent, licensed under the MIT License.

**Forked from:** superpowers v4.3.1
**Fork date:** 2026-02-23
**License:** MIT (Copyright 2025 Jesse Vincent)

## Forked Skills

- brainstorming
- writing-plans
- executing-plans
- test-driven-development
- systematic-debugging
- verification-before-completion
- requesting-code-review
- receiving-code-review
- dispatching-parallel-agents
- subagent-driven-development
- finishing-a-development-branch
- using-git-worktrees
- writing-skills

## Reins-Native Skills

- analyze-codebase
- create-exec-plan (merged with writing-plans)
- publish-plan
- review-pr (merged with requesting-code-review)
- sync-context
- update-harness
- using-reins (replaces using-superpowers)
```

**Step 3: Do NOT copy `using-superpowers`**

We'll create `using-reins` in the next task instead.

**Step 4: Commit**

```bash
git add .claude/skills/
git commit -m "feat(skills): fork superpowers skills into reins (MIT, Jesse Vincent)"
```

---

### Task 14: Create `using-reins` intro skill

**Files:**
- Create: `.claude/skills/using-reins/SKILL.md`
- Remove: `.claude/skills/capture-idea.md` (superseded)

**Step 1: Create the intro skill**

Create `.claude/skills/using-reins/SKILL.md` based on `using-superpowers` but customized for reins:

- Replace all references to `superpowers` with `reins`
- Add `reins doc create` as the mandatory doc creation method
- Reference harness paths (`harness/{repo}/...`)
- Reference golden principles
- Add `reins` CLI commands to the skill priority section

Key differences from using-superpowers:
- Adds a "Doc Creation" section: "All harness docs MUST be created via `reins doc create <type> \"title\"`. Never hand-write harness docs."
- References `harness/{repo}/` paths instead of `docs/plans/`
- Lists reins-specific skills alongside forked ones
- Removes `docs/plans/` references entirely

**Step 2: Commit**

```bash
git add .claude/skills/using-reins/ .claude/skills/capture-idea.md
git commit -m "feat(skills): add using-reins intro skill, remove capture-idea"
```

---

### Task 15: Customize forked skills for reins conventions

**Files:**
- Modify: `.claude/skills/brainstorming/SKILL.md`
- Modify: `.claude/skills/writing-plans/SKILL.md`
- Modify: `.claude/skills/executing-plans/SKILL.md`
- Modify: All other forked skill SKILL.md files

This is the largest task — apply reins customizations across all forked skills.

**Step 1: Customize brainstorming skill**

Key changes to `brainstorming/SKILL.md`:
- Step 5 (Write design doc): Change `docs/plans/YYYY-MM-DD-<topic>-design.md` to `reins doc create design "<topic>"`
- Step 6 (Transition): Keep invoking `writing-plans` but note it will use `reins doc create plan`
- Add: "Before exploring context, check `harness/{repo}/golden-principles.md` for project rules."
- Remove: Any `docs/plans/` references

**Step 2: Customize writing-plans skill**

Key changes to `writing-plans/SKILL.md`:
- "Save plans to" line: Change `docs/plans/YYYY-MM-DD-<feature-name>.md` to: "Save plans via: `reins doc create plan \"<feature-name>\"`"
- Plan header: Keep structure but note the file is created by `reins doc create`
- Add: "After creating the plan, run `reins publish` to share with team."

**Step 3: Merge writing-plans with create-exec-plan**

The existing `create-exec-plan.md` skill tells agents to run `reins plan create`. The forked `writing-plans` tells agents to write detailed implementation plans. These serve different purposes:

- `reins plan create` → creates the exec plan directory structure
- `writing-plans` → writes the detailed task-by-task plan content

Keep both but update `create-exec-plan.md` to reference `reins doc create plan` and remove overlap.

**Step 4: Merge requesting-code-review with review-pr**

Keep `review-pr.md` as the reins-native review skill (it references golden principles, dependency rules, etc.). Update `requesting-code-review/SKILL.md` to delegate to `review-pr` skill for reins projects.

**Step 5: Customize remaining skills**

For each of these skills, search for `docs/plans/` and replace with appropriate `reins doc create` or `harness/{repo}/` references:

- `executing-plans/SKILL.md` — update plan location reference
- `test-driven-development/SKILL.md` — add golden principle #7 reference
- `systematic-debugging/SKILL.md` — add golden principle #4 reference (agent-readable errors)
- `verification-before-completion/SKILL.md` — add `reins lint` as verification step
- `finishing-a-development-branch/SKILL.md` — add `reins publish` step
- `subagent-driven-development/SKILL.md` — update plan location references
- `dispatching-parallel-agents/SKILL.md` — no changes needed
- `using-git-worktrees/SKILL.md` — no changes needed
- `receiving-code-review/SKILL.md` — no changes needed
- `writing-skills/SKILL.md` — update save location for skills

**Step 6: Commit**

```bash
git add .claude/skills/
git commit -m "feat(skills): customize forked skills for reins conventions"
```

---

### Task 16: Update harness asset bundling for `reins attach`

**Files:**
- Modify: `pyproject.toml` (wheel force-include)
- Modify: `src/reins/commands/attach.py` (if skills copy logic needs updating)

**Step 1: Verify `pyproject.toml` includes the new skills**

Check that `[tool.hatch.build.targets.wheel.force-include]` includes `.claude/skills` — it should already work since it includes the whole directory.

**Step 2: Verify `reins attach` copies the updated skills**

Run attach in a test repo to confirm the forked skills are included.

**Step 3: Commit if changes needed**

```bash
git add pyproject.toml src/reins/commands/attach.py
git commit -m "chore(packaging): verify skill bundling includes forked skills"
```

---

### Task 17: Run full test suite and linters

**Step 1: Run all tests**

Run: `uv run pytest tests/ -v`
Expected: All tests PASS.

**Step 2: Run linters**

Run: `uv run reins lint`
Expected: No errors.

**Step 3: Run harness cross-link check**

After the migration, verify no broken links:

```bash
uv run reins lint
```

**Step 4: Fix any failures**

If tests or linters fail, fix and commit:

```bash
git commit -m "fix: address test/lint feedback from skills fork"
```

---

### Task 18: Manual smoke test

**Step 1: Test doc creation**

```bash
uv run reins doc create design "Smoke Test Design"
uv run reins doc create spec "Smoke Test Spec"
uv run reins doc create debt "Smoke Test Debt Item"
uv run reins idea "smoke test idea"
```

Verify each creates a file at the correct repo-scoped path with correct frontmatter.

**Step 2: Test enforcement hook**

In a Claude Code session, try to Write a new file to `harness/reins/design-docs/test.md`. Verify it's blocked with the "use reins doc create" message.

Then try to Edit the existing `harness/reins/design-docs/skills-fork-and-template-docs.md`. Verify it succeeds.

**Step 3: Test skills**

Verify the `.claude/skills/` directory has all expected skills and they load correctly in Claude Code.

**Step 4: Commit harness state**

```bash
cd harness && git add -A && git commit -m "chore: smoke test artifacts" && cd ..
git add harness
git commit -m "chore: update harness pointer after smoke test"
```

---

## Acceptance Criteria

- [ ] `repo_slug()` derives repo name from git remote origin
- [ ] Harness docs live at `harness/{repo}/...` (migration complete)
- [ ] AGENTS.md path table reflects repo-scoped paths
- [ ] All linter rules support repo-scoped paths dynamically
- [ ] `reins doc create <type> "title"` generates correct template for all 7 types
- [ ] `reins idea` uses repo-scoped path
- [ ] Enforcement hook blocks new harness doc creation via Write/Edit
- [ ] Enforcement hook allows editing existing harness docs
- [ ] All superpowers skills forked with MIT attribution
- [ ] `using-reins` intro skill replaces `using-superpowers`
- [ ] Forked skills reference `reins doc create` instead of `docs/plans/`
- [ ] Forked skills reference golden principles where relevant
- [ ] Existing reins skills deduplicated with forked skills
- [ ] `reins attach` bundles the updated skill set
- [ ] All tests pass
- [ ] All linters pass

## Tasks

- [ ] Task 1: Add `repo_slug()` helper
- [ ] Task 2: Add `repo_harness_dir()` helper
- [ ] Task 3: Migrate existing harness docs
- [ ] Task 4: Update AGENTS.md
- [ ] Task 5: Update linter rules
- [ ] Task 6: Update `idea.py` for repo-scoped paths
- [ ] Task 7: Update `plan.py` for repo-scoped paths
- [ ] Task 8: Update remaining commands
- [ ] Task 9: Update test fixtures
- [ ] Task 10: Build `doc create` command
- [ ] Task 11: Wire `doc create` into CLI
- [ ] Task 12: Add enforcement hook
- [ ] Task 13: Fork superpowers skills
- [ ] Task 14: Create `using-reins` skill
- [ ] Task 15: Customize forked skills
- [ ] Task 16: Update asset bundling
- [ ] Task 17: Full test suite + linters
- [ ] Task 18: Manual smoke test
