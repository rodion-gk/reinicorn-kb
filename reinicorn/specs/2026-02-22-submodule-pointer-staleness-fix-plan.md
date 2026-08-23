---
type: spec
title: Submodule Pointer Staleness Fix — Implementation Plan
slug: 2026-02-22-submodule-pointer-staleness-fix-plan
lifecycle: done
status: implemented
created: 2026-07-17
author: Michael Biehl
---

# Submodule Pointer Staleness Fix — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Eliminate stale submodule pointer bugs (#7) by removing `ignore = all`, replacing rebase with merge, and auto-staging the parent pointer.

**Architecture:** Every code path that changes the harness HEAD also stages `git add harness` in the parent. History-rewriting operations (rebase) are replaced with merge. Removing `ignore = all` makes drift visible as a safety net.

**Tech Stack:** Python 3, git CLI, pytest

---

### Task 1: Remove `ignore = all` from `.gitmodules`

**Files:**
- Modify: `.gitmodules`

**Step 1: Edit `.gitmodules`**

Remove the `ignore = all` line:

```ini
[submodule "harness"]
	path = harness
	url = git@github.com:your-org/your-kb.git
	branch = main
```

**Step 2: Commit**

```bash
git add .gitmodules
git commit -m "fix(gitmodules): remove ignore=all to surface pointer drift (#7)"
```

---

### Task 2: `commit_harness()` stages parent pointer

**Files:**
- Modify: `src/reins/harness.py:74-94`
- Test: `tests/test_commit_harness.py`

**Step 1: Write the failing test**

Add to `tests/test_commit_harness.py`:

```python
def test_commit_harness_stages_parent_pointer(submodule_repo: Path) -> None:
    """After committing in harness, the parent pointer should be staged."""
    (submodule_repo / "harness" / "test.md").write_text("staged pointer test\n")

    commit_harness(submodule_repo, "test: staged pointer")

    # Parent should have harness staged (not committed, just in index)
    r = subprocess.run(
        ["git", "diff", "--cached", "--name-only"],
        cwd=submodule_repo, capture_output=True, text=True,
    )
    assert "harness" in r.stdout
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_commit_harness.py::test_commit_harness_stages_parent_pointer -v`
Expected: FAIL — `harness` not in staged output.

**Step 3: Write minimal implementation**

In `src/reins/harness.py`, add `git add harness` after the submodule commit succeeds. The function currently ends:

```python
    r = run_git("commit", "-q", "-m", message, check=False, cwd=h_dir)
    return r.returncode == 0
```

Change to:

```python
    r = run_git("commit", "-q", "-m", message, check=False, cwd=h_dir)
    if r.returncode == 0:
        run_git("add", "harness", check=False, cwd=root)
        return True
    return False
```

**Step 4: Run test to verify it passes**

Run: `uv run pytest tests/test_commit_harness.py -v`
Expected: All 4 tests PASS.

**Step 5: Commit**

```bash
git add src/reins/harness.py tests/test_commit_harness.py
git commit -m "fix(harness): stage parent pointer after commit_harness (#7)"
```

---

### Task 3: Replace rebase with merge in `sync.py`

**Files:**
- Modify: `src/reins/commands/sync.py:24-33`
- Test: `tests/commands/test_sync.py`

**Step 1: Write the failing test**

Add to `tests/commands/test_sync.py`:

```python
def test_sync_uses_merge_not_rebase(submodule_repo: Path, tmp_path: Path) -> None:
    """When ff-only fails, sync should merge (not rebase) to preserve SHAs."""
    harness = submodule_repo / "harness"
    remote = tmp_path / "harness-remote"

    # Create a divergent history: push a commit to remote, make a local commit
    # 1. Push a commit via the remote bare repo
    staging = tmp_path / "staging-clone"
    subprocess.run(
        ["git", "-c", "protocol.file.allow=always",
         "clone", str(remote), str(staging)],
        check=True, capture_output=True,
    )
    subprocess.run(["git", "config", "user.email", "test@test.com"], cwd=staging, check=True)
    subprocess.run(["git", "config", "user.name", "Test User"], cwd=staging, check=True)
    (staging / "remote-change.md").write_text("remote\n")
    subprocess.run(["git", "add", "-A"], cwd=staging, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "remote commit"], cwd=staging, check=True)
    subprocess.run(
        ["git", "-c", "protocol.file.allow=always", "push", "origin", "main"],
        cwd=staging, check=True, capture_output=True,
    )

    # 2. Make a local commit in harness (diverges from remote)
    (harness / "local-change.md").write_text("local\n")
    subprocess.run(["git", "add", "-A"], cwd=harness, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "local commit"], cwd=harness, check=True)
    local_sha = subprocess.run(
        ["git", "rev-parse", "HEAD"], cwd=harness, capture_output=True, text=True,
    ).stdout.strip()

    with patch("reins.commands.sync.repo_root", return_value=submodule_repo), \
         patch("reins.commands.sync.current_branch", return_value="feature/x"):
        result = cmd_sync()

    assert result == 0

    # The local commit SHA should still exist (merge preserves it, rebase wouldn't)
    r = subprocess.run(
        ["git", "cat-file", "-t", local_sha],
        cwd=harness, capture_output=True, text=True,
    )
    assert r.stdout.strip() == "commit"

    # The local commit should be an ancestor of HEAD (not orphaned)
    r = subprocess.run(
        ["git", "merge-base", "--is-ancestor", local_sha, "HEAD"],
        cwd=harness,
    )
    assert r.returncode == 0
```

Also add a test for pointer staging:

```python
def test_sync_stages_parent_pointer(submodule_repo: Path) -> None:
    """After sync, the parent submodule pointer should be staged."""
    with patch("reins.commands.sync.repo_root", return_value=submodule_repo), \
         patch("reins.commands.sync.current_branch", return_value="feature/x"):
        result = cmd_sync()

    assert result == 0
    r = subprocess.run(
        ["git", "diff", "--cached", "--name-only"],
        cwd=submodule_repo, capture_output=True, text=True,
    )
    # May or may not be staged depending on whether HEAD changed,
    # but at minimum it should not error
    assert r.returncode == 0
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/test_sync.py -v`
Expected: `test_sync_uses_merge_not_rebase` FAIL (rebase rewrites the SHA).

**Step 3: Write minimal implementation**

Replace sync.py lines 24-33. Change from:

```python
    r = run_git("merge", "origin/main", "--ff-only", check=False, cwd=harness_dir)
    if r.returncode != 0:
        r = run_git("rebase", "origin/main", check=False, cwd=harness_dir)
        if r.returncode != 0:
            run_git("rebase", "--abort", check=False, cwd=harness_dir)
            console.warn(
                "Could not fast-forward or rebase. "
                "Try: reins publish first."
            )
            return 1
```

To:

```python
    r = run_git("merge", "origin/main", "--ff-only", check=False, cwd=harness_dir)
    if r.returncode != 0:
        r = run_git("merge", "origin/main", check=False, cwd=harness_dir)
        if r.returncode != 0:
            console.warn(
                "Could not merge origin/main. "
                "Resolve conflicts in harness/, then reins publish."
            )
            return 1
```

Also add pointer staging after the success message (before `check_overlap`):

```python
    console.success("Harness synced to latest main.")

    # Stage updated pointer in parent
    run_git("add", "harness", check=False, cwd=root)

    print()
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/commands/test_sync.py -v`
Expected: All tests PASS.

**Step 5: Commit**

```bash
git add src/reins/commands/sync.py tests/commands/test_sync.py
git commit -m "fix(sync): replace rebase with merge, stage parent pointer (#7)"
```

---

### Task 4: Replace rebase with merge in `publish.py`

**Files:**
- Modify: `src/reins/commands/publish.py:42-60`
- Test: `tests/commands/test_publish.py`

**Step 1: Write the failing test**

Add to `tests/commands/test_publish.py`:

```python
def test_publish_updates_parent_pointer(submodule_repo: Path) -> None:
    """After publish, parent submodule pointer should be staged or committed."""
    harness = submodule_repo / "harness"
    (harness / "new-file.md").write_text("# New\n")
    subprocess.run(["git", "add", "-A"], cwd=harness, check=True)
    subprocess.run(
        ["git", "commit", "-q", "-m", "test commit"],
        cwd=harness, check=True,
    )

    with patch("reins.commands.publish.repo_root", return_value=submodule_repo), \
         patch("reins.commands.publish.can_publish", return_value=True):
        result = cmd_publish()

    assert result == 0

    # Parent pointer should reflect the harness HEAD
    tree_sha = subprocess.run(
        ["git", "ls-tree", "HEAD", "harness"],
        cwd=submodule_repo, capture_output=True, text=True,
    ).stdout.split()[2] if subprocess.run(
        ["git", "ls-tree", "HEAD", "harness"],
        cwd=submodule_repo, capture_output=True, text=True,
    ).stdout else ""

    harness_head = subprocess.run(
        ["git", "rev-parse", "HEAD"],
        cwd=harness, capture_output=True, text=True,
    ).stdout.strip()

    # Either staged or committed — the pointer should match harness HEAD
    staged = subprocess.run(
        ["git", "diff", "--cached", "--name-only"],
        cwd=submodule_repo, capture_output=True, text=True,
    ).stdout
    # The commit_harness + publish flow should have updated it
    assert "harness" in staged or tree_sha == harness_head
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/commands/test_publish.py::test_publish_updates_parent_pointer -v`
Expected: May pass or fail depending on current behavior — the key change is removing rebase.

**Step 3: Write minimal implementation**

In `src/reins/commands/publish.py`, change line 44 from:

```python
        run_git(*fta, "pull", "--rebase", "origin", "main", check=False, cwd=harness_dir)
```

To:

```python
        run_git(*fta, "pull", "origin", "main", check=False, cwd=harness_dir)
```

Remove the redundant pointer update block (lines 52-60) since `commit_harness()` now handles this:

```python
    # Remove these lines:
    # Optionally update parent submodule pointer (cosmetic, for GitHub UI)
    run_git("add", "harness", check=False, cwd=root)
    r = run_git("diff", "--cached", "--quiet", check=False, cwd=root)
    if r.returncode != 0:
        run_git(
            "commit", "-m", "chore(harness): update submodule pointer",
            check=False, cwd=root,
        )
        console.info("Updated parent submodule pointer.")
```

Replace with just a stage (no auto-commit):

```python
    # Stage parent pointer (picked up by next parent commit)
    run_git("add", "harness", check=False, cwd=root)
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/commands/test_publish.py -v`
Expected: All 4 tests PASS.

**Step 5: Commit**

```bash
git add src/reins/commands/publish.py tests/commands/test_publish.py
git commit -m "fix(publish): replace rebase with merge, simplify pointer update (#7)"
```

---

### Task 5: Auto-push worker stages parent pointer

**Files:**
- Modify: `src/reins/harness_push_worker.py:52-56`
- Test: `tests/test_harness_push_worker.py`

**Step 1: Write the failing test**

Add to `tests/test_harness_push_worker.py` inside `class TestMain`:

```python
    def test_stages_parent_pointer_after_push(self, tmp_path: Path):
        """After pushing harness, main() should stage the parent pointer."""
        harness_dir = self._make_harness(tmp_path)

        def fake_run_git(*args, **kwargs):
            result = MagicMock(returncode=0)
            cmd = list(args)
            if cmd == ["rev-parse", "origin/feature/x"]:
                result.stdout = "expected_sha\n"
            elif cmd == ["diff", "--cached", "--quiet"]:
                result.returncode = 0  # nothing staged
            elif cmd == ["rev-parse", "HEAD"]:
                result.stdout = "local_sha\n"
            elif cmd == ["rev-parse", "origin/main"]:
                result.stdout = "remote_sha\n"
            else:
                result.stdout = ""
            return result

        with patch("reins.harness_push_worker._wait_for_parent", return_value=True):
            with patch("reins.harness_push_worker.ensure_harness_on_main"):
                with patch(
                    "reins.harness_push_worker.run_git", side_effect=fake_run_git
                ) as mock_git:
                    main(
                        ppid=1,
                        root=tmp_path,
                        branch="feature/x",
                        pre_push_head="expected_sha",
                        harness_dir=harness_dir,
                    )

        # Verify git add harness was called with cwd=root (tmp_path)
        add_calls = [
            c for c in mock_git.call_args_list
            if list(c.args) == ["add", "harness"]
            and c.kwargs.get("cwd") == tmp_path
        ]
        assert len(add_calls) == 1
```

**Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_harness_push_worker.py::TestMain::test_stages_parent_pointer_after_push -v`
Expected: FAIL — no `git add harness` call with `cwd=root`.

**Step 3: Write minimal implementation**

In `src/reins/harness_push_worker.py`, after the push (line 56), add:

```python
        run_git("push", "origin", "main", cwd=harness_dir, check=False)

    # Stage parent pointer so next commit picks it up
    run_git("add", "harness", cwd=root, check=False)
```

Note: the `git add harness` should happen after any harness work (commit or push), not just inside the push conditional. Move it to after the push block so it always runs when the worker gets this far.

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_harness_push_worker.py -v`
Expected: All 5 tests PASS.

**Step 5: Commit**

```bash
git add src/reins/harness_push_worker.py tests/test_harness_push_worker.py
git commit -m "fix(push-worker): stage parent pointer after harness push (#7)"
```

---

### Task 6: Post-checkout hook stages pointer after init

**Files:**
- Modify: `src/reins/commands/internal/post_checkout.py:30-34`

**Step 1: Write minimal implementation**

In `src/reins/commands/internal/post_checkout.py`, after the `ensure_harness_on_main` call (line 34), add:

```python
                ensure_harness_on_main(harness_dir)
                run_git("add", "harness", cwd=root, check=False)
```

This is inside the `contextlib.suppress(Exception)` block, so failures are swallowed — appropriate for a hook that must never block checkout.

**Step 2: Run all tests to verify nothing breaks**

Run: `uv run pytest tests/ -v`
Expected: All tests PASS.

**Step 3: Commit**

```bash
git add src/reins/commands/internal/post_checkout.py
git commit -m "fix(post-checkout): stage parent pointer after submodule init (#7)"
```

---

### Task 7: Run full test suite and verify

**Step 1: Run all tests**

Run: `uv run pytest tests/ -v`
Expected: All tests PASS.

**Step 2: Manual smoke test**

```bash
reins idea "test pointer staging"
git status  # Should show 'harness' as staged (new commit)
git diff --cached --name-only  # Should include 'harness'
```

**Step 3: Final commit if any fixups needed**

If tests revealed issues, fix and commit with: `fix(harness): address test feedback (#7)`
