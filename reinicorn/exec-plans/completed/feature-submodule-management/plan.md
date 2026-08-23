---
type: plan
title: 'Execution Plan: feature/submodule-management'
slug: feature-submodule-management
lifecycle: done
status: complete
created: 2026-02-17
author: mnbiehl
branch: feature/submodule-management
ticket: N/A
---

# Execution Plan: feature/submodule-management

## Goal

Redesign harness submodule management so the submodule always stays on its `main` branch, every harness-modifying command auto-commits locally, and agents are prevented from running raw git commands against the submodule.

## Design Doc

[`harness/reins/design-docs/submodule-management.md`](../../../specs/submodule-management.md)

## Acceptance Criteria

- [ ] `.gitmodules` has `branch = main` and `ignore = all`
- [ ] post-checkout hook no longer runs `git submodule update` — only initializes on fresh clone/worktree, then checks out `main` branch
- [ ] post-merge hook no longer runs `git submodule update` — keeps stale plan archival
- [ ] `commit_harness()` utility exists in `reins.harness` and auto-commits inside the submodule
- [ ] `idea`, `plan create`, `plan complete`, `agent run` all call `commit_harness()` after writing
- [ ] `sync` fetches + merges instead of detaching HEAD
- [ ] `publish` pushes submodule `main` to remote, optionally updates parent pointer
- [ ] All existing tests pass + new tests for `commit_harness`, `sync`, `publish`
- [ ] `reins --help`, `reins sync`, `reins publish` work end-to-end

## Tasks

### Task 1: Update `.gitmodules`

**Files:**
- Modify: `.gitmodules`

**Steps:**

1. Add `branch = main` and `ignore = all` to the `[submodule "harness"]` section:

```ini
[submodule "harness"]
    path = harness
    url = git@github.com:your-org/your-kb.git
    branch = main
    ignore = all
```

2. Run `git submodule sync` to propagate the config.

3. Commit: `chore: configure submodule to track main with ignore=all`

---

### Task 2: Create `commit_harness()` utility

**Files:**
- Modify: `tests/conftest.py` — add shared `submodule_repo` fixture
- Modify: `src/reins/harness.py` — add `commit_harness()` function
- Create: `tests/test_commit_harness.py`

**Step 1: Add shared `submodule_repo` fixture to `tests/conftest.py`**

This fixture is reused by Tasks 4 and 5. Add it to `tests/conftest.py` alongside the existing `harness_repo` fixture:

```python
@pytest.fixture
def submodule_repo(tmp_path: Path) -> Path:
    """Create a parent repo with a real harness submodule on main branch.

    Returns the parent repo root. The submodule has a remote at
    tmp_path/harness-remote that can be used for push/fetch tests.
    """
    # Create the "remote" harness repo
    remote = tmp_path / "harness-remote"
    remote.mkdir()
    subprocess.run(["git", "init", "-q", str(remote)], check=True)
    subprocess.run(["git", "config", "user.email", "test@test.com"], cwd=remote, check=True)
    subprocess.run(["git", "config", "user.name", "Test"], cwd=remote, check=True)
    (remote / "README.md").write_text("# Harness\n")
    subprocess.run(["git", "add", "-A"], cwd=remote, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "init"], cwd=remote, check=True)

    # Create the parent repo with harness as submodule
    parent = tmp_path / "parent"
    parent.mkdir()
    subprocess.run(["git", "init", "-q", str(parent)], check=True)
    subprocess.run(["git", "config", "user.email", "test@test.com"], cwd=parent, check=True)
    subprocess.run(["git", "config", "user.name", "Test"], cwd=parent, check=True)
    subprocess.run(
        ["git", "submodule", "add", str(remote), "harness"],
        cwd=parent, check=True, capture_output=True,
    )
    subprocess.run(["git", "add", "-A"], cwd=parent, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "init"], cwd=parent, check=True)

    # Put harness on main branch (not detached)
    harness = parent / "harness"
    subprocess.run(["git", "checkout", "-q", "main"], cwd=harness, check=False)
    subprocess.run(["git", "checkout", "-q", "-b", "main"], cwd=harness, check=False)

    return parent
```

**Step 2: Write failing tests**

Test file: `tests/test_commit_harness.py` (uses the shared `submodule_repo` fixture from conftest):

```python
"""Tests for commit_harness() auto-commit utility."""

from __future__ import annotations

import subprocess
from pathlib import Path

from reins.harness import commit_harness

def test_commit_harness_commits_new_file(submodule_repo: Path) -> None:
    (submodule_repo / "harness" / "ideas" / "test.md").parent.mkdir(parents=True, exist_ok=True)
    (submodule_repo / "harness" / "ideas" / "test.md").write_text("# Test idea\n")

    result = commit_harness(submodule_repo, "test: add idea")

    assert result is True
    # Verify commit exists in submodule
    log = subprocess.run(
        ["git", "log", "--oneline", "-1"],
        cwd=submodule_repo / "harness", capture_output=True, text=True,
    )
    assert "test: add idea" in log.stdout

def test_commit_harness_returns_false_when_nothing_to_commit(submodule_repo: Path) -> None:
    result = commit_harness(submodule_repo, "nothing here")
    assert result is False

def test_commit_harness_noop_for_inline_layout(tmp_path: Path) -> None:
    """Inline layout (no .gitmodules) should no-op."""
    repo = tmp_path / "inline"
    repo.mkdir()
    subprocess.run(["git", "init", "-q", str(repo)], check=True)
    (repo / "harness").mkdir()
    (repo / "harness" / "file.md").write_text("inline\n")

    result = commit_harness(repo, "should not commit")
    assert result is False

def test_commit_harness_fixes_detached_head(submodule_repo: Path) -> None:
    """If harness is on detached HEAD, commit_harness should checkout main first."""
    harness = submodule_repo / "harness"
    # Detach HEAD
    head = subprocess.run(
        ["git", "rev-parse", "HEAD"], cwd=harness, capture_output=True, text=True,
    ).stdout.strip()
    subprocess.run(["git", "checkout", "-q", head], cwd=harness, check=True)

    # Write a file and commit
    (harness / "test.md").write_text("detached test\n")
    result = commit_harness(submodule_repo, "fix: from detached")
    assert result is True

    # Should be back on main
    branch = subprocess.run(
        ["git", "symbolic-ref", "--short", "HEAD"],
        cwd=harness, capture_output=True, text=True,
    ).stdout.strip()
    assert branch == "main"
```

**Step 2: Run tests — verify they fail**

```bash
uv run pytest tests/test_commit_harness.py -v
```

Expected: ImportError — `commit_harness` doesn't exist yet.

**Step 3: Implement `commit_harness()`**

Add to `src/reins/harness.py`:

```python
def commit_harness(root: Path, message: str) -> bool:
    """Auto-commit all changes inside the harness submodule.

    Returns True if a commit was made, False if nothing to commit
    or harness is not a submodule.
    """
    if harness_layout(root) != "submodule":
        return False

    harness_dir = root / "harness"
    if not harness_dir.is_dir():
        return False

    from reins.git import run_git

    # Ensure we're on main branch, not detached HEAD
    try:
        r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=harness_dir)
        if r.returncode != 0 or not r.stdout.strip():
            run_git("checkout", "main", check=False, cwd=harness_dir)
    except Exception:
        pass

    # Stage all changes
    run_git("add", "-A", cwd=harness_dir, check=False)

    # Check if there's anything to commit
    r = run_git("diff", "--cached", "--quiet", check=False, cwd=harness_dir)
    if r.returncode == 0:
        return False  # Nothing staged

    # Commit
    r = run_git("commit", "-q", "-m", message, check=False, cwd=harness_dir)
    return r.returncode == 0
```

**Step 4: Run tests — verify they pass**

```bash
uv run pytest tests/test_commit_harness.py -v
```

**Step 5: Commit**

```bash
git add src/reins/harness.py tests/test_commit_harness.py
git commit -m "feat(harness): add commit_harness() auto-commit utility"
```

---

### Task 3: Wire `commit_harness()` into harness-modifying commands

**Files:**
- Modify: `src/reins/commands/idea.py`
- Modify: `src/reins/commands/plan.py`
- Modify: `src/reins/commands/agent.py`
- Modify: `tests/commands/test_idea.py`
- Modify: `tests/commands/test_plan.py`

**Step 1: Update `idea.py`**

Replace the current `git add` + `git commit` block (lines 60-68) that commits in the parent repo with a call to `commit_harness()`:

```python
# At the top, add import:
from reins.harness import commit_harness, require_harness

# Replace lines 60-68 with:
    commit_harness(root, f"idea: {slug}")
```

Remove the `run_git` import if no longer used.

**Step 2: Update `plan.py`**

In `cmd_plan_create()`, after `console.success(...)` at the end (line 105), add:

```python
    from reins.harness import commit_harness
    commit_harness(root, f"plan: create {branch}")
```

In `cmd_plan_complete()`, after the `shutil.move()` call (line 180), add:

```python
    from reins.harness import commit_harness
    commit_harness(root, f"plan: complete {branch}")
```

**Step 3: Update `agent.py`**

In `cmd_agent_run()`, after writing `job_meta` (line 112), add:

```python
    from reins.harness import commit_harness
    commit_harness(root, f"agent: dispatch {task_name}")
```

**Step 4: Update existing tests**

In `tests/commands/test_idea.py`: the test currently likely checks that `run_git` was called with `add` and `commit`. Update the mock/assertion to expect `commit_harness` to be called instead. Use `unittest.mock.patch("reins.commands.idea.commit_harness")`.

In `tests/commands/test_plan.py`: add assertions that `commit_harness` is called after `plan create` and `plan complete`.

**Step 5: Run all tests**

```bash
uv run pytest -v
```

**Step 6: Commit**

```bash
git add src/reins/commands/idea.py src/reins/commands/plan.py src/reins/commands/agent.py tests/commands/test_idea.py tests/commands/test_plan.py
git commit -m "feat(commands): wire commit_harness into idea, plan, agent"
```

---

### Task 4: Rewrite `sync.py` — fetch + merge instead of detaching HEAD

**Files:**
- Modify: `src/reins/commands/sync.py`
- Create: `tests/commands/test_sync.py`

**Step 1: Write failing tests**

Uses the shared `submodule_repo` fixture from `tests/conftest.py` (added in Task 2).

```python
"""Tests for reins sync command."""

from __future__ import annotations

import subprocess
from pathlib import Path
from unittest.mock import patch

from reins.commands.sync import cmd_sync

def test_sync_stays_on_main_branch(submodule_repo: Path) -> None:
    """After sync, harness should be on main branch, not detached HEAD."""
    with patch("reins.commands.sync.repo_root", return_value=submodule_repo):
        with patch("reins.commands.sync.current_branch", return_value="feature/x"):
            cmd_sync()

    branch = subprocess.run(
        ["git", "symbolic-ref", "--short", "HEAD"],
        cwd=submodule_repo / "harness", capture_output=True, text=True,
    ).stdout.strip()
    assert branch == "main"
```

**Step 2: Run tests — verify they fail**

**Step 3: Rewrite `sync.py`**

Replace the submodule update logic with fetch + merge:

```python
def cmd_sync() -> int:
    root = repo_root()
    if root is None:
        return 1
    layout = require_harness(root)

    console.header("Syncing harness...")
    print()

    if layout == "submodule":
        console.info("Layout: submodule")
        harness_dir = root / "harness"

        # Ensure we're on main branch
        r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=harness_dir)
        if r.returncode != 0 or r.stdout.strip() != "main":
            run_git("checkout", "main", check=False, cwd=harness_dir)

        # Fetch and merge latest
        run_git("fetch", "origin", "main", check=False, cwd=harness_dir)
        r = run_git("merge", "origin/main", "--ff-only", check=False, cwd=harness_dir)
        if r.returncode != 0:
            # Try rebase if ff-only fails
            r = run_git("rebase", "origin/main", check=False, cwd=harness_dir)
            if r.returncode != 0:
                run_git("rebase", "--abort", check=False, cwd=harness_dir)
                console.warn("Could not fast-forward or rebase. Try: reins publish first.")
                return 1

        console.success("Harness synced to latest main.")
    else:
        console.info("Layout: inline (harness is part of this repo)")
        console.info("Run 'git pull' to get latest changes from remote.")

    print()
    branch = current_branch()
    if branch:
        check_overlap(branch, root)

    return 0
```

**Step 4: Run tests — verify they pass**

```bash
uv run pytest tests/commands/test_sync.py -v
```

**Step 5: Commit**

```bash
git add src/reins/commands/sync.py tests/commands/test_sync.py
git commit -m "feat(sync): fetch+merge instead of submodule update"
```

---

### Task 5: Rewrite `publish.py` — simplified push flow

**Files:**
- Modify: `src/reins/commands/publish.py`
- Create: `tests/commands/test_publish.py`

**Step 1: Write failing tests**

Uses the shared `submodule_repo` fixture from `tests/conftest.py` (added in Task 2).

Test that publish:
- Pushes submodule `main` to remote
- Retries once with `pull --rebase` on failure
- No longer commits in the parent repo by default

**Step 2: Rewrite `publish.py`**

```python
def cmd_publish() -> int:
    if not can_publish():
        mode = get_mode()
        console.warn(f"Publishing blocked (mode: {mode}).")
        if mode == "incognito":
            console.info("Run 'reins incognito' to toggle off.")
        elif mode == "disabled":
            console.info("Run 'reins enable' to re-enable.")
        return 1

    root = repo_root()
    if root is None:
        return 1
    layout = require_harness(root)

    console.header("Publishing harness changes...")
    print()

    if layout != "submodule":
        console.info("Harness is inline — changes push with your next git push.")
        return 0

    harness_dir = root / "harness"

    # Ensure on main
    r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=harness_dir)
    if r.returncode != 0 or r.stdout.strip() != "main":
        run_git("checkout", "main", check=False, cwd=harness_dir)

    # Push
    push = run_git("push", "origin", "main", check=False, cwd=harness_dir)
    if push.returncode != 0:
        console.info("Push failed, pulling latest and retrying...")
        run_git("pull", "--rebase", "origin", "main", check=False, cwd=harness_dir)
        push = run_git("push", "origin", "main", check=False, cwd=harness_dir)
        if push.returncode != 0:
            console.error("Push failed after retry. Resolve manually.")
            return 1

    console.success("Harness pushed to remote main.")

    # Optionally update parent submodule pointer (cosmetic, for GitHub UI)
    run_git("add", "harness", check=False, cwd=root)
    r = run_git("diff", "--cached", "--quiet", check=False, cwd=root)
    if r.returncode != 0:
        run_git(
            "commit", "-m", "chore(harness): update submodule pointer",
            check=False, cwd=root,
        )
        console.info("Updated parent submodule pointer.")

    print()
    branch = current_branch()
    if branch:
        check_overlap(branch, root)
    return 0
```

**Step 3: Run tests**

```bash
uv run pytest tests/commands/test_publish.py -v
```

**Step 4: Commit**

```bash
git add src/reins/commands/publish.py tests/commands/test_publish.py
git commit -m "feat(publish): simplified push to submodule main"
```

---

### Task 6: Remove `git submodule update` from hooks

**Files:**
- Modify: `src/reins/commands/internal/post_checkout.py`
- Modify: `src/reins/commands/internal/post_merge.py`
- Modify: `.git/hooks/post-checkout` (bash hook)
- Modify: `.git/hooks/post-merge` (bash hook)
- Modify: `tests/commands/test_post_merge.py`

**Step 1: Update Python `post_checkout.py`**

Remove the `subprocess.Popen(["git", "submodule", "update", ...])` block (lines 28-35). Replace with a lightweight init-only check:

```python
    # Ensure harness exists (fresh clone / new worktree only)
    harness_dir = root / "harness"
    gitmodules = root / ".gitmodules"
    if gitmodules.is_file() and "harness" in gitmodules.read_text():
        if not any(harness_dir.iterdir()) if harness_dir.is_dir() else True:
            # Empty or missing — first-time init
            with contextlib.suppress(Exception):
                subprocess.run(
                    ["git", "submodule", "update", "--init", "harness"],
                    cwd=root, capture_output=True, check=False,
                )
                subprocess.run(
                    ["git", "checkout", "main"],
                    cwd=harness_dir, capture_output=True, check=False,
                )
```

**Step 2: Update Python `post_merge.py`**

Remove the `subprocess.Popen(["git", "submodule", "update", ...])` block (lines 17-24). Keep the `_archive_stale_plans()` call.

**Step 3: Update bash hooks**

`post-checkout`: Replace the submodule update block (lines 41-54) with the same init-only logic.

`post-merge`: Remove the submodule update block (lines 15-17). Keep exit 0.

**Step 4: Run all tests**

```bash
uv run pytest -v
```

**Step 5: Commit**

```bash
git add src/reins/commands/internal/post_checkout.py src/reins/commands/internal/post_merge.py .git/hooks/post-checkout .git/hooks/post-merge tests/commands/test_post_merge.py
git commit -m "fix(hooks): stop clobbering harness with submodule update"
```

---

### Task 7: Full integration test

**Steps:**

1. Run full test suite: `uv run pytest -v`
2. Run linter: `uv run ruff check src/ tests/`
3. Run type checker: `uv run pyright src/`
4. Manual smoke test:
   - `uv run reins --help`
   - `uv run reins sync`
   - `uv run reins publish`
   - `uv run reins idea "test idea for smoke test"` then verify commit exists in harness submodule
   - `uv run reins plan status`

5. Commit any remaining fixes.

## Dependencies

- Design doc: `harness/reins/design-docs/submodule-management.md`
- Existing idea: `harness/reins/ideas/mnbiehl/2026-02-17-redesign-submodule-management-stop-using-raw-git-submodule-c.md`
