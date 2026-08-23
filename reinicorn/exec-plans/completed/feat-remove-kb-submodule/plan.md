---
type: plan
title: 'Execution Plan: feat/remove-kb-submodule'
slug: feat-remove-kb-submodule
lifecycle: done
status: complete
created: 2026-08-14
author: Michael Biehl
branch: feat/remove-kb-submodule
ticket: N/A
spec: reinicorn/specs/remove-the-kb-submodule.md
---

# Remove the kb Submodule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the kb git submodule with a gitignored plain clone at `kb/`, per the approved spec `reinicorn/specs/remove-the-kb-submodule.md` — no pointer, no drift, no bump PRs, with automatic migration for existing repos.

**Architecture:** `get_kb_dir()` stays the single discovery seam and switches from `.gitmodules` parsing to "`<root>/kb` containing `.git`". All pointer plumbing (`stage_kb_pointer`, `kb_gitlink`) is deleted. `ensure_kb_on_main()` becomes a fetch-first, report-failure guard that `commit_kb()` obeys. The boundary is `.gitignore` + a new `pre-commit` hook + a CI check. Existing repos migrate in place via `rcorn init`/`rcorn update`.

**Tech Stack:** Python 3 (uv, pytest, ruff, pyright), bash git hooks, GitHub Actions.

## Global Constraints

- **Spec:** `kb/reinicorn/specs/remove-the-kb-submodule.md` (approved). Where this plan and the spec disagree, the spec wins.
- **Decision (Michael, 2026-08-14):** the spec-approval pre-push gate re-anchors on the **kb clone's committed HEAD** (not fetched origin/main, not left dormant).
- **P2:** never hard-code `"kb"` in Python — use `KB_DIR_NAME` (`reinicorn/config.py`) / `REGISTRY`. (Bash hooks and workflow YAML may name `kb` literally; the location is convention per spec Non-Goals.)
- **P3:** no deprecated aliases or compat shims — replaced code is deleted outright (`setup_submodule`, `stage_kb_pointer`, `kb_gitlink`, `_parse_kb_submodule_path`, `_has_kb_submodule_path`, `_gitmodules_url` all go).
- **Golden principle 4:** every error names what/where/how-to-fix.
- **Golden principle 14 (full gate):** before claiming any task done: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`. The final task also runs `uv run rcorn kb lint`.
- Commit messages: conventional commits, one commit per task (plus intermediate commits where a task says so).
- The repo working copy still has the submodule layout until Task 10 — the new `get_kb_dir()` accepts it (a submodule checkout's `kb/.git` file satisfies the clone test), so `rcorn` keeps working locally mid-branch.

---

### Task 1: `get_kb_dir()` detects a kb clone; fixtures move to the clone layout

**Files:**
- Modify: `src/reinicorn/kb.py:29-91` (`_parse_kb_submodule_path`, `get_kb_dir`, `require_kb_dir`)
- Modify: `tests/conftest.py` (fixtures `kb_repo`, new `kb_clone_repo`)
- Test: `tests/test_kb.py`

**Interfaces:**
- Consumes: `KB_DIR_NAME` from `reinicorn.config`.
- Produces: `get_kb_dir(root: Path | None = None) -> Path | None` — `<root>/kb` iff `(kb/.git).exists()` (file or dir; submodule checkouts still pass during transition). `require_kb_dir(root) -> Path` — same signature, new error text. Fixture `kb_clone_repo(tmp_path) -> Path` returning a parent-repo root whose `kb/` is a real clone of a bare remote at `tmp_path/kb-remote`. Every later task's tests use `kb_clone_repo` for kb operations; `submodule_repo` survives **only** for Task 9's migration tests.

- [ ] **Step 1: Write the failing tests**

In `tests/test_kb.py`, delete the tests covering `.gitmodules` parsing and the path-traversal guard (the guard exists solely because `.gitmodules` was attacker-controllable; both are gone). Add:

```python
def test_get_kb_dir_requires_git_entry(tmp_path):
    """A kb/ directory without .git is not a kb."""
    root = tmp_path / "repo"
    (root / "kb").mkdir(parents=True)
    assert get_kb_dir(root) is None


def test_get_kb_dir_detects_clone(kb_clone_repo):
    assert get_kb_dir(kb_clone_repo) == kb_clone_repo / "kb"


def test_get_kb_dir_accepts_gitfile(tmp_path):
    """A .git *file* (submodule/worktree checkout) also counts — transition safety."""
    root = tmp_path / "repo"
    kb = root / "kb"
    kb.mkdir(parents=True)
    (kb / ".git").write_text("gitdir: ../.git/modules/kb\n")
    assert get_kb_dir(root) == kb


def test_require_kb_dir_error_names_both_paths(tmp_path, capsys):
    root = tmp_path / "repo"
    root.mkdir()
    with pytest.raises(SystemExit):
        require_kb_dir(root)
    out = capsys.readouterr().out
    assert "rcorn kb sync" in out
    assert "rcorn init" in out
```

- [ ] **Step 2: Update `tests/conftest.py`**

In `kb_repo`: make `kb/` its own git repo and gitignore it (replace the `.gitmodules` write at lines 96-101):

```python
    # kb/ is a plain nested git repo, gitignored by the parent (clone layout)
    _git_init(kb)
    (repo / ".gitignore").write_text("kb/\n")
```

and after the template files are written, commit inside the kb so it has a HEAD:

```python
    _git_commit(kb, "kb init")
```

(the existing `_git_commit(repo)` at the end stays — with `.gitignore` in place it no longer stages anything under `kb/`).

Add the new fixture after `submodule_repo`:

```python
@pytest.fixture
def kb_clone_repo(tmp_path: Path) -> Path:
    """Parent repo with kb/ as an ordinary clone of a local bare remote.

    The clone layout every kb-operation test uses. The remote lives at
    tmp_path/kb-remote for push/fetch assertions; `submodule_repo` remains
    only for migration tests.
    """
    staging = tmp_path / "kb-staging"
    staging.mkdir()
    _git_init(staging)
    (staging / "README.md").write_text("# Kb\n")
    _git_commit(staging, "init")

    remote = tmp_path / "kb-remote"
    run_git("-c", "protocol.file.allow=always",
            "clone", "--bare", str(staging), str(remote))

    parent = tmp_path / "parent"
    parent.mkdir()
    _git_init(parent)
    (parent / ".gitignore").write_text("kb/\n")
    (parent / ".reinicorn-config").write_text(
        f'REINICORN_KB_REMOTE="{remote}"\n'
    )
    _git_commit(parent, "init")

    run_git("-c", "protocol.file.allow=always",
            "clone", str(remote), str(parent / "kb"))
    kb = parent / "kb"
    run_git("config", "user.email", "test@test.com", cwd=kb)
    run_git("config", "user.name", "Test User", cwd=kb)
    run_git("config", "protocol.file.allow", "always", cwd=kb)
    return parent
```

- [ ] **Step 3: Run the new tests to verify they fail**

Run: `uv run pytest tests/test_kb.py -v`
Expected: the new tests FAIL (`get_kb_dir` still parses `.gitmodules`); pre-existing gitmodules tests are gone.

- [ ] **Step 4: Implement**

In `src/reinicorn/kb.py`, delete `_parse_kb_submodule_path` entirely and replace `get_kb_dir`/`require_kb_dir`:

```python
def get_kb_dir(root: Path | None = None) -> Path | None:
    """Return the kb clone directory, or None when no kb exists yet.

    The kb is an ordinary git clone at <root>/kb. A directory without a
    .git entry is not a kb — a leftover empty dir must not be treated as
    one. `.git` may be a file (transitional submodule checkout, linked
    worktree) or a directory (plain clone); both are real kbs.
    """
    if root is None:
        root = repo_root()
        if root is None:
            return None
    candidate = root / KB_DIR_NAME
    if (candidate / ".git").exists():
        return candidate
    return None


def require_kb_dir(root: Path | None = None) -> Path:
    """Return the kb directory, or print an error and raise SystemExit(1)."""
    kb_dir = get_kb_dir(root)
    if kb_dir is None:
        console.error(
            f"No kb found at {KB_DIR_NAME}/.\n"
            "  If this repo already uses Reinicorn (teammate clone): "
            "run 'rcorn kb sync' to clone it.\n"
            "  If Reinicorn was never set up here: run 'rcorn init'."
        )
        raise SystemExit(1)
    return kb_dir
```

The module docstring's "submodule" wording and the file's remaining "submodule" comment mentions (e.g. `overlapping_branches`) become "clone"/"kb".

- [ ] **Step 5: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS. If tests elsewhere relied on the traversal guard or `.gitmodules` parsing, fix them now (they may only need the `kb_repo` fixture change).

- [ ] **Step 6: Commit**

```bash
git add src/reinicorn/kb.py tests/conftest.py tests/test_kb.py
git commit -m "feat(kb): detect the kb as a plain clone, not a .gitmodules entry"
```

---

### Task 2: `ensure_kb_on_main()` fetches, reports, never moves backwards; `commit_kb()` refuses on failure

**Files:**
- Modify: `src/reinicorn/kb.py` (`ensure_kb_on_main`, `commit_kb`)
- Modify: `src/reinicorn/commands/sync.py:30`, `src/reinicorn/commands/publish.py:33`, `src/reinicorn/commands/review.py:109` (call sites now check the result)
- Test: `tests/test_commit_kb.py`, `tests/test_kb.py`

**Interfaces:**
- Consumes: `file_transport_args`, `run_git`, `report_failure` from `reinicorn.git`.
- Produces: `checkout_kb_main(kb_dir: Path) -> bool` (fetch best-effort + get HEAD onto `main`; no fast-forward) and `ensure_kb_on_main(kb_dir: Path) -> bool` (checkout_kb_main + fast-forward local `main` to `origin/main`; False when HEAD cannot reach `main` or local `main` has diverged from a fetched `origin/main`). `commit_kb(...)` unchanged signature, but returns False without staging when `ensure_kb_on_main` fails.

- [ ] **Step 1: Write the failing tests**

In `tests/test_kb.py` (uses `kb_clone_repo`):

```python
def _detach(kb: Path) -> None:
    run_git("checkout", "-q", "--detach", "HEAD", cwd=kb)


def test_ensure_kb_on_main_fast_forwards_stale_local_main(kb_clone_repo, tmp_path):
    """From a detached HEAD, checkout main must not revert to a stale local main."""
    kb = kb_clone_repo / "kb"
    # Advance the remote past local main via a second clone
    other = tmp_path / "other"
    run_git("clone", "-q", str(tmp_path / "kb-remote"), str(other))
    run_git("config", "user.email", "t@t", cwd=other)
    run_git("config", "user.name", "T", cwd=other)
    (other / "README.md").write_text("# Kb v2\n")
    run_git("add", "-A", cwd=other)
    run_git("commit", "-q", "-m", "v2", cwd=other)
    run_git("push", "-q", "origin", "main", cwd=other)
    _detach(kb)

    assert ensure_kb_on_main(kb) is True
    assert (kb / "README.md").read_text() == "# Kb v2\n"  # not reverted to v1


def test_ensure_kb_on_main_reports_failed_checkout(kb_clone_repo):
    """A checkout that cannot land on main returns False instead of lying."""
    kb = kb_clone_repo / "kb"
    _detach(kb)
    # Uncommitted change conflicting with main blocks the checkout
    run_git("rm", "-q", "README.md", cwd=kb)
    (kb / "README.md").write_text("conflicting\n")
    run_git("add", "README.md", cwd=kb)
    run_git("commit", "-q", "-m", "detached edit", cwd=kb)
    (kb / "README.md").write_text("dirty\n")

    assert ensure_kb_on_main(kb) is False


def test_commit_kb_refuses_off_main(kb_clone_repo, monkeypatch):
    """No commit lands on a detached HEAD — the work stays in the worktree."""
    kb = kb_clone_repo / "kb"
    _detach(kb)
    run_git("rm", "-q", "README.md", cwd=kb)
    run_git("commit", "-q", "-m", "conflict setup", cwd=kb)
    (kb / "README.md").write_text("draft\n")

    assert commit_kb(kb_clone_repo, "doc: draft", kb_dir=kb) is False
    r = run_git("log", "--oneline", "-1", cwd=kb)
    assert "doc: draft" not in r.stdout
    assert (kb / "README.md").read_text() == "draft\n"  # work intact
```

In `tests/test_commit_kb.py`, port existing cases from `submodule_repo` to `kb_clone_repo` (mechanical: the kb path is still `<root>/kb`; drop any assertions about a staged parent pointer — those move to Task 3).

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_kb.py tests/test_commit_kb.py -v`
Expected: new tests FAIL (`ensure_kb_on_main` returns None today and never fetches).

- [ ] **Step 3: Implement in `src/reinicorn/kb.py`**

```python
def checkout_kb_main(kb_dir: Path) -> bool:
    """Fetch origin/main (best effort) and put kb HEAD on the main branch.

    The fetch comes first so "main" can mean origin/main rather than a
    possibly-stale local ref; offline is survivable (local commits publish
    later), so a failed fetch warns and continues. A failed checkout is
    reported and returns False — committing into a detached HEAD makes
    work look saved when it is not.
    """
    fta = file_transport_args(cwd=kb_dir)
    fetch = run_git(*fta, "fetch", "origin", "main", check=False, cwd=kb_dir)
    if fetch.returncode != 0:
        console.warn(
            "Could not fetch kb origin/main — continuing with the local ref."
        )

    r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=kb_dir)
    if r.returncode != 0 or r.stdout.strip() != "main":
        co = run_git("checkout", "main", check=False, cwd=kb_dir)
        if co.returncode != 0:
            report_failure("put the kb on its main branch", co, warn=True)
            return False
    return True


def ensure_kb_on_main(kb_dir: Path) -> bool:
    """Put the kb on an up-to-date main. Returns False when it cannot.

    "Up to date" means: HEAD is main, and local main is not behind a
    fetched origin/main (fast-forwarded here; ahead-only is fine — those
    are unpublished doc commits). Never moves the working tree backwards
    and never discards uncommitted kb work.
    """
    if not checkout_kb_main(kb_dir):
        return False
    # Only meaningful when the fetch above succeeded; --ff-only on an
    # already-ahead main is a no-op ("Already up to date").
    has_remote_ref = run_git(
        "rev-parse", "--verify", "-q", "origin/main", check=False, cwd=kb_dir,
    )
    if has_remote_ref.returncode != 0:
        return True  # nothing fetched to compare against
    ff = run_git("merge", "--ff-only", "origin/main", check=False, cwd=kb_dir)
    if ff.returncode != 0:
        console.error(
            "Kb main has diverged from origin/main and cannot fast-forward.\n"
            f"  Where: {kb_dir}\n"
            "  How to fix: run 'rcorn kb sync' to merge origin/main first."
        )
        return False
    return True
```

In `commit_kb`, replace the bare `ensure_kb_on_main(resolved)` call (line 139) with:

```python
    if not ensure_kb_on_main(resolved):
        console.error(
            "Refusing to commit kb changes: the kb is not on an up-to-date "
            "main (see above).\n"
            f"  Where: {resolved}\n"
            "  Your edits are still in the kb working tree — nothing is lost.\n"
            "  How to fix: resolve the state above, then rerun this command."
        )
        return False
```

Call sites:
- `sync.py:30`: replace `ensure_kb_on_main(kb_dir)` with `checkout_kb_main(kb_dir)` and bail on False (`return 1`) — sync's own merge below is the divergence repair path, so it must not require an already-fast-forwardable main. Import changes accordingly.
- `publish.py:33` and `review.py:109`: `if not ensure_kb_on_main(kb_dir): return 1` (review raises/returns per its surrounding error style — match it).

- [ ] **Step 4: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/reinicorn/kb.py src/reinicorn/commands/sync.py src/reinicorn/commands/publish.py src/reinicorn/commands/review.py tests/
git commit -m "fix(kb): ensure_kb_on_main fetches first, reports failure, never moves backwards"
```

---

### Task 3: Delete the pointer plumbing

**Files:**
- Modify: `src/reinicorn/kb.py` (delete `stage_kb_pointer`; `commit_kb` line 155 drops the call)
- Modify: `src/reinicorn/commands/sync.py:63`, `src/reinicorn/commands/publish.py:47`, `src/reinicorn/commands/internal/post_checkout.py:75` (delete calls + imports; sync's "Stage updated pointer" comment goes too)
- Test: `tests/test_commit_kb.py`, `tests/commands/test_sync.py`, `tests/commands/test_publish.py`, `tests/commands/internal/test_post_checkout.py`

**Interfaces:**
- Consumes: nothing new.
- Produces: `stage_kb_pointer` no longer exists; no caller stages anything in the parent repo. Later tasks must not reintroduce a parent-index write.

- [ ] **Step 1: Write the failing test**

```python
def test_commit_kb_stages_nothing_in_parent(kb_clone_repo):
    kb = kb_clone_repo / "kb"
    (kb / "note.md").write_text("hi\n")
    assert commit_kb(kb_clone_repo, "doc: note") is True
    r = run_git("diff", "--cached", "--name-only", cwd=kb_clone_repo)
    assert r.stdout.strip() == ""
```

- [ ] **Step 2: Run it to verify current behavior**

Run: `uv run pytest tests/test_commit_kb.py -v`
Expected: the new test PASSES already in the clone layout only because `kb/` is gitignored — it exists as a regression tripwire. Any test in the four listed files that asserts a staged `kb` pointer FAILS after Step 3 and must be deleted, not adapted.

- [ ] **Step 3: Delete `stage_kb_pointer` and all four call sites; remove dead imports.**

- [ ] **Step 4: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS (ruff F401 will catch any leftover import).

- [ ] **Step 5: Commit**

```bash
git add -A src/reinicorn tests
git commit -m "feat(kb): delete stage_kb_pointer — no pointer exists to stage"
```

---

### Task 4: Pre-push pushes an ahead kb; spec gate anchors on kb HEAD; `kb_gitlink` deleted

**Files:**
- Modify: `src/reinicorn/commands/internal/pre_push.py` (`_ensure_kb_pushed`, `cmd_pre_push`, module docstring, fail-closed message)
- Modify: `src/reinicorn/commands/internal/spec_gate.py` (anchor on kb HEAD)
- Modify: `src/reinicorn/kb.py` (delete `kb_gitlink`)
- Test: `tests/commands/internal/test_pre_push.py`

**Interfaces:**
- Consumes: `get_kb_dir`, `get_mode`, `tracked_paths_at`, `doc_text_at`.
- Produces: `_ensure_kb_pushed(root: Path) -> int` (branches parameter dropped — no per-branch pointer exists). `ensure_plan_spec_approved(root, branches)` keeps its signature (branches still select which plans to check) but resolves docs from the kb clone's committed `HEAD`. `kb_gitlink` no longer exists.

- [ ] **Step 1: Write the failing tests**

```python
def test_pre_push_pushes_ahead_kb(kb_clone_repo, monkeypatch):
    """Local kb main ahead of origin/main gets pushed before the parent push."""
    kb = kb_clone_repo / "kb"
    (kb / "doc.md").write_text("x\n")
    run_git("add", "-A", cwd=kb)
    run_git("commit", "-q", "-m", "local doc", cwd=kb)
    monkeypatch.chdir(kb_clone_repo)

    assert _ensure_kb_pushed(kb_clone_repo) == 0
    r = run_git("rev-list", "--count", "origin/main..main", cwd=kb)
    assert r.stdout.strip() == "0"  # nothing left unpublished


def test_pre_push_noop_when_kb_current(kb_clone_repo, monkeypatch):
    monkeypatch.chdir(kb_clone_repo)
    assert _ensure_kb_pushed(kb_clone_repo) == 0


def test_spec_gate_reads_kb_head(kb_clone_repo, monkeypatch):
    """An unapproved spec committed to the kb blocks the branch push."""
    # Arrange a plan on kb HEAD declaring a draft spec, then push a branch
    # named to match it; assert ensure_plan_spec_approved(...) == 1.
    # Mirror the existing gitlink-based test's doc setup, swapping the
    # "commit to kb + point the branch at it" arrangement for a plain
    # kb commit (the clone's HEAD is the anchor now).
```

Port the existing spec-gate tests: wherever they commit a plan/spec into the kb and update the parent gitlink, now just commit into the kb clone — assertions are unchanged. A staged-but-uncommitted spec must still NOT satisfy the gate (keep that test; it passes because `rev-parse HEAD` ignores the index).

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/commands/internal/test_pre_push.py -v`
Expected: FAIL (`_ensure_kb_pushed` still takes `branches` and consults gitlinks).

- [ ] **Step 3: Implement**

`pre_push.py` — replace `_ensure_kb_pushed` (keep the failure-reporting block verbatim where noted):

```python
def _ensure_kb_pushed(root: Path) -> int:
    """Push local kb main when it is ahead of origin/main.

    The invariant is simply "the kb is always published" — no pointer is
    consulted, because none exists. Runs synchronously BEFORE the parent
    push so reviewers and CI can always read the docs the pushed work
    references. Returns non-zero only when unpublished kb commits cannot
    be pushed.
    """
    kb_dir = get_kb_dir(root)
    if kb_dir is None:
        return 0

    mode = get_mode(root)
    if mode in ("incognito", "disabled"):
        return 0

    run_git("fetch", "origin", "main", "--quiet", cwd=kb_dir, check=False)
    ahead = run_git(
        "rev-list", "--count", "origin/main..main", check=False, cwd=kb_dir,
    )
    if ahead.returncode != 0 or ahead.stdout.strip() in ("", "0"):
        # No origin/main to compare against (brand-new kb, offline first
        # fetch) fails open: this guard protects publication, and blocking
        # every push on an unverifiable comparison would brick the repo.
        return 0

    print("\U0001f984 Kb has unpushed commits, pushing now...")
    r = run_git("push", "origin", "main", check=False, cwd=kb_dir)
    if r.returncode != 0:
        print("\n❌ " + "\n".join(explain_failure("push the kb", r)))
        print(
            "\n   The parent push would reference kb docs that are not\n"
            "   published, so reviewers and CI could not read them.\n"
            "   Fix: rcorn kb publish (or bypass once with git push --no-verify)\n",
            flush=True,
        )
        return 1

    print("\U0001f984 Kb pushed successfully.")
    return 0
```

In `cmd_pre_push`: call `_ensure_kb_pushed(root)` (no branches); update the fail-closed message ("dangling kb submodule pointer" → "unpublished kb commits") and the module docstring ("kb submodule sync" → "kb publication"). The `(kb_dir / ".git").exists()` guard is now inside `get_kb_dir` — drop the duplicate.

`spec_gate.py` — replace the per-branch gitlink resolution (lines 64-83):

```python
        head = run_git("rev-parse", "--verify", "-q", "HEAD",
                       check=False, cwd=kb_dir)
        if head.returncode != 0:
            return 0  # empty kb clone: nothing committed to gate on
        rev = head.stdout.strip()
        tracked = tracked_paths_at(kb_dir, rev)

        for branch in branches:
            if not branch:
                continue
            plan_rel = branch_doc_path("plan", Path(scope), branch).as_posix()
            if plan_rel not in tracked:
                continue
            plan_path = f"{KB_DIR_NAME}/{plan_rel}"
            rc = _check_plan(
                doc_text_at(kb_dir, rev, plan_rel),
                plan_path, scope, kb_dir, rev, tracked,
            )
            if rc != 0:
                return rc
```

Update the module + function docstrings: everything is read from the kb clone's committed HEAD (always `main`; `_ensure_kb_pushed` just published it) — never from the kb index or worktree, so a staged-but-uncommitted spec still cannot satisfy the gate. Remove the `kb_gitlink` import; delete `kb_gitlink` from `kb.py`.

- [ ] **Step 4: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/reinicorn/commands/internal/pre_push.py src/reinicorn/commands/internal/spec_gate.py src/reinicorn/kb.py tests/
git commit -m "feat(hooks): pre-push publishes an ahead kb; spec gate anchors on kb HEAD"
```

---

### Task 5: `setup_kb_clone()` replaces `setup_submodule()`; init records the remote and gitignores `kb/`

**Files:**
- Create: `src/reinicorn/kb_setup.py` (from `src/reinicorn/submodule.py`; delete the old file)
- Modify: `src/reinicorn/commands/init.py` (`_has_kb_submodule_path` deleted, `_init_path`, teammate path, remote recording, summary text)
- Test: `tests/test_kb_setup.py` (from `tests/test_submodule.py`), `tests/test_init_*.py`, `tests/test_multi_repo_init.py`

**Interfaces:**
- Consumes: `is_remote_empty`, `seed_remote`, `generate_seed_tree`, `validate_git_url`, `scratch_clone` (all kept as-is).
- Produces: `kb_setup.KbSetupError` (replaces `SubmoduleError`), `setup_kb_clone(target_dir: Path, url: str, repo_slug: str | None = None) -> bool`, `cleanup_failed_kb(target_dir: Path) -> None`, `ensure_kb_gitignored(root: Path) -> bool` (idempotent; returns True when it added the entry). Tasks 6 and 9 import `setup_kb_clone` / `ensure_kb_gitignored`.

- [ ] **Step 1: Write the failing tests**

`tests/test_kb_setup.py` (port `test_submodule.py`'s empty-remote/seed/URL-validation cases verbatim with the new names, then add):

```python
def test_setup_kb_clone_creates_plain_clone(tmp_path):
    bare = tmp_path / "remote.git"
    run_git("init", "-q", "--bare", "-b", "main", str(bare))
    target = tmp_path / "proj"
    target.mkdir()
    _git_init(target)

    assert setup_kb_clone(target, str(bare.resolve()), repo_slug="proj") is True
    kb = target / "kb"
    assert (kb / ".git").is_dir()                      # a clone, not a submodule
    assert not (target / ".gitmodules").exists()
    assert "kb/" in (target / ".gitignore").read_text()
    r = run_git("symbolic-ref", "--short", "HEAD", cwd=kb)
    assert r.stdout.strip() == "main"


def test_ensure_kb_gitignored_idempotent(tmp_path):
    root = tmp_path
    (root / ".gitignore").write_text("*.pyc\n")
    assert ensure_kb_gitignored(root) is True
    assert ensure_kb_gitignored(root) is False
    assert (root / ".gitignore").read_text() == "*.pyc\nkb/\n"
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_kb_setup.py -v`
Expected: FAIL with import errors (module does not exist yet).

- [ ] **Step 3: Implement `src/reinicorn/kb_setup.py`**

Copy `submodule.py`, then:
- Rename `SubmoduleError` → `KbSetupError`, `cleanup_failed_submodule` → `cleanup_failed_kb` (drop its `.git/modules` cleanup? **No** — keep it: a migration or old checkout can leave `.git/modules/kb` behind and a fresh setup must not trip on it).
- Replace the `submodule add` + two `git config -f .gitmodules` calls in `setup_submodule` with:

```python
    file_allow = ("-c", "protocol.file.allow=always") if url.startswith("/") else ()
    r = run_git(*file_allow, "clone", url, str(kb_dir), check=False)
    if r.returncode != 0:
        cleanup_failed_kb(target_dir)
        raise KbSetupError("\n".join(explain_failure(
            "clone the kb", r,
            detail=[
                f"URL: {url}",
                "How to fix: Check the URL is correct and you have access.",
            ],
        )))

    ensure_kb_gitignored(target_dir)
    console.success("Kb cloned (tracking main)")
    return True
```

- Rename the function to `setup_kb_clone`; the stale-state check becomes `if kb_dir.is_dir() and not (kb_dir / ".git").exists():`. The "already registered" retry branch (old lines 114-119) is deleted — a clone either succeeds or is cleaned up.
- Add:

```python
def ensure_kb_gitignored(root: Path) -> bool:
    """Add `kb/` to the repo .gitignore. Returns True when it was added."""
    gitignore = root / ".gitignore"
    entry = f"{KB_DIR_NAME}/"
    existing = gitignore.read_text() if gitignore.is_file() else ""
    if entry in existing.splitlines():
        return False
    text = existing if existing.endswith("\n") or not existing else existing + "\n"
    gitignore.write_text(text + entry + "\n")
    return True
```

Delete `src/reinicorn/submodule.py` and `tests/test_submodule.py`.

- [ ] **Step 4: Update `src/reinicorn/commands/init.py`**

- Delete `_has_kb_submodule_path`. `_init_path` becomes:

```python
def _init_path(cwd: Path) -> Literal["full", "assets_only", "hooks_only"]:
    """Classify the setup path without changing repository state."""
    if not ((cwd / KB_DIR_NAME) / ".git").exists():
        return "full"
    if (cwd / MANIFEST_PATH).is_file():
        return "hooks_only"
    return "assets_only"
```

- Teammate path (old lines 193-195): replace `git submodule update --init` with a clone:

```python
        if not kb_dir.is_dir() or not any(kb_dir.iterdir()):
            from reinicorn.kb_remote import resolve_kb_remote_url
            url = resolve_kb_remote_url(cwd)
            if not url:
                console.error(
                    f"No kb at {KB_DIR_NAME}/ and no recorded remote.\n"
                    f"  How to fix: set {KB_REMOTE_KEY} in .reinicorn-config, "
                    "or run 'rcorn init --kb-url <url>'."
                )
                return 1
            console.info("Cloning kb...")
            setup_kb_clone(cwd, url)
```

(NOTE: `_init_path` returning non-"full" requires `kb/.git` to exist, so this branch is now only reachable on the "full" path's teammate variant — verify while implementing and drop the branch if it is genuinely dead; the `resolve_kb_remote_url`-driven clone lives in `rcorn kb sync` (Task 6) either way.)
- Full path: `setup_submodule(cwd, kb_url, repo_slug=slug)` → `setup_kb_clone(cwd, kb_url, repo_slug=slug)`, catching `KbSetupError`. After success, record the remote:

```python
    from reinicorn.kb_remote import KB_REMOTE_KEY
    config_set(KB_REMOTE_KEY, kb_url, cwd)
```

- `_print_full_summary` step 4: `git add .gitmodules kb ...` → `git add .gitignore AGENTS.md .agents .claude .cursor .github .reinicorn CLAUDE.md .reinicorn-config`.
- Module docstring: "submodule + AGENTS + skills + hooks" → "kb clone + AGENTS + skills + hooks".

- [ ] **Step 5: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS. `tests/test_init_*.py` / `test_multi_repo_init.py` assertions about `.gitmodules` flip to asserting `kb/.git` exists, `.gitignore` contains `kb/`, and `.reinicorn-config` records `REINICORN_KB_REMOTE`.

- [ ] **Step 6: Commit**

```bash
git add -A src/reinicorn tests
git commit -m "feat(init): kb is set up as a gitignored clone; remote recorded in .reinicorn-config"
```

---

### Task 6: Bootstrap — `kb sync` clones when absent; home + post-checkout follow; `.gitmodules` fallback deleted

**Files:**
- Modify: `src/reinicorn/commands/sync.py` (clone-if-missing), `src/reinicorn/commands/home.py:41-46`, `src/reinicorn/commands/internal/post_checkout.py` (`_init_kb`, `_kb_reference_args`, `cmd_post_checkout`), `src/reinicorn/kb_remote.py` (delete `_gitmodules_url`)
- Test: `tests/commands/test_sync.py`, `tests/test_post_checkout_worktree.py`, `tests/commands/internal/test_post_checkout.py`, `tests/test_kb_remote.py`

**Interfaces:**
- Consumes: `setup_kb_clone`, `KbSetupError`, `resolve_kb_remote_url`, `_main_checkout_root` (already in `kb_remote.py`).
- Produces: `cmd_sync` clones before its fetch/merge when `get_kb_dir` is None. `configured_kb_remote_url(root)` reads config only. `post_checkout._clone_reference_args(root) -> list[str]` replaces `_kb_reference_args`.

- [ ] **Step 1: Write the failing tests**

```python
def test_sync_clones_when_kb_absent(kb_clone_repo):
    """A teammate clone (config present, kb/ missing) gets a kb from one command."""
    shutil.rmtree(kb_clone_repo / "kb")
    with chdir(kb_clone_repo):          # match the file's existing cwd idiom
        assert cmd_sync() == 0
    assert (kb_clone_repo / "kb" / ".git").exists()


def test_home_reports_missing_clone(kb_clone_repo, capsys):
    shutil.rmtree(kb_clone_repo / "kb")
    with chdir(kb_clone_repo):
        cmd_home()
    out = capsys.readouterr().out
    assert "not cloned yet" in out
    assert "rcorn kb sync" in out
    assert "submodule" not in out
```

Port `tests/test_post_checkout_worktree.py`: a new linked worktree must end up with a kb clone on `main` whose origin matches the main checkout's kb origin (assertions stay; the arrangement uses `kb_clone_repo` + `git worktree add`).

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/commands/test_sync.py tests/test_post_checkout_worktree.py -v`
Expected: FAIL (`require_kb_dir` exits when kb/ is missing; home prints the submodule hint).

- [ ] **Step 3: Implement**

`sync.py` — before `require_kb_dir`:

```python
    kb_dir = get_kb_dir(root)
    if kb_dir is None:
        url = resolve_kb_remote_url(root)
        if not url:
            console.error(
                f"No kb at {KB_DIR_NAME}/ and no recorded remote to clone from.\n"
                f"  How to fix: run 'rcorn init' to set up the kb."
            )
            return 1
        console.progress("No kb clone found — cloning...")
        try:
            setup_kb_clone(root, url)
        except KbSetupError as e:
            console.error(str(e))
            return 1
    kb_dir = require_kb_dir(root)
```

`home.py` lines 41-46:

```python
    if not kb_dir.is_dir():
        print("kb: not cloned yet")
        console.next_step("rcorn kb sync")
        return 0
```

Wait — with the new `get_kb_dir`, a missing clone returns `None`, so this branch folds into the one above it. Correct shape: `get_kb_dir` returning `None` while `.reinicorn-config` records `REINICORN_KB_REMOTE` means "not cloned yet" → `rcorn kb sync`; with no recorded remote it stays "not set up in this repo" → `rcorn init`:

```python
    kb_dir = get_kb_dir(root)
    if kb_dir is None:
        if configured_kb_remote_url(root):
            print("kb: not cloned yet")
            console.next_step("rcorn kb sync")
        else:
            print("kb: not set up in this repo")
            console.next_step("rcorn init", "rcorn help")
        return 0
```

`post_checkout.py`:
- `_kb_reference_args` → `_clone_reference_args(root)`: borrow objects from the main checkout's kb clone (worktrees share nothing anymore via `.git/modules`):

```python
def _clone_reference_args(root: Path) -> list[str]:
    """`git clone` args that borrow kb objects from the main checkout's kb.

    A new linked worktree clones its own kb; --reference-if-able borrows
    objects from the main checkout's clone when one exists (no network
    cost for history), --dissociate copies them so the new kb never
    depends on the other clone staying alive. Falls back to a plain
    clone when there is nothing to borrow.
    """
    from reinicorn.kb_remote import _main_checkout_root
    ref = _main_checkout_root(root) / KB_DIR_NAME
    if (ref / ".git").exists():
        return ["--reference-if-able", str(ref), "--dissociate"]
    return []
```

- `_init_kb(root)` (drop the `kb_dir` parameter — it derives it): resolve URL first (unchanged comment), then `run_git("clone", *_clone_reference_args(root), remote, str(root / KB_DIR_NAME), check=False)` guarded by the same report-don't-raise structure; `apply_kb_remote_url` stays (inherited URL may differ from the recorded one); `checkout_kb_main(kb_dir)` replaces `ensure_kb_on_main` + the deleted `stage_kb_pointer`.
- `cmd_post_checkout` (lines 97-102): the trigger is now "no kb, but a remote is recorded":

```python
    if get_kb_dir(root) is None and resolve_kb_remote_url(root):
        _init_kb(root)
```

`kb_remote.py`: delete `_gitmodules_url`; `configured_kb_remote_url` returns `config_get(KB_REMOTE_KEY, root=root)`; trim the module docstring's submodule paragraph.

- [ ] **Step 4: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A src/reinicorn tests
git commit -m "feat(kb): one-command bootstrap — sync and post-checkout clone the kb when absent"
```

---

### Task 7: `repo_root()` walks up from a kb toplevel instead of asking for a superproject

**Files:**
- Modify: `src/reinicorn/git.py:202-223` (`repo_root`)
- Test: `tests/test_git.py`

**Interfaces:**
- Consumes: `CONFIG_FILE_NAME` from `reinicorn.identity`, `KB_DIR_NAME` from `reinicorn.config` (import both lazily inside the function — `config.py` imports `reinicorn.git` lazily already; keep the cycle impossible).
- Produces: `repo_root(quiet: bool = False) -> Path | None`, same contract; from inside `kb/` it returns the parent project root.

- [ ] **Step 1: Write the failing test**

```python
def test_repo_root_resolves_parent_from_inside_kb(kb_clone_repo, monkeypatch):
    """A command run with cwd inside kb/ must resolve the parent project,
    not the kb — otherwise docs land in kb/kb/ (spec §7). Nothing else
    covers this regression."""
    monkeypatch.chdir(kb_clone_repo / "kb")
    assert repo_root() == kb_clone_repo


def test_repo_root_keeps_plain_repos(tmp_path, monkeypatch):
    """A repo that happens to BE named kb but has no parent config is itself."""
    repo = tmp_path / "kb"
    repo.mkdir()
    _init_repo(repo)  # the file's existing init helper
    monkeypatch.chdir(repo)
    assert repo_root() == repo
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_git.py -v`
Expected: first test FAILS — `--show-superproject-working-tree` returns empty for a plain nested clone, so `repo_root()` returns the kb itself.

- [ ] **Step 3: Implement — replace the superproject block in `repo_root`:**

```python
    try:
        r = run_git("rev-parse", "--show-toplevel")
        root = Path(r.stdout.strip())

        # Inside kb/ the toplevel is the kb clone itself (a plain nested
        # repo has no superproject to ask git about). A toplevel named kb
        # whose parent carries a .reinicorn-config is the kb; resolve to
        # the parent so commands act on the real project.
        from reinicorn.config import KB_DIR_NAME
        from reinicorn.identity import CONFIG_FILE_NAME
        if root.name == KB_DIR_NAME and (root.parent / CONFIG_FILE_NAME).is_file():
            root = root.parent

        return root
    except (subprocess.CalledProcessError, FileNotFoundError):
        ...  # unchanged
```

Also update `repo_slug`'s docstring line about "running from inside a submodule".

- [ ] **Step 4: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/reinicorn/git.py tests/test_git.py
git commit -m "fix(git): resolve the parent project from inside the kb clone"
```

---

### Task 8: `pre-commit` hook refuses staged kb/ paths

**Files:**
- Create: `hooks/pre-commit`
- Modify: `src/reinicorn/hooks_health.py:18` (`HOOK_NAMES`)
- Test: `tests/test_hooks_health.py`, `tests/commands/` hook-install coverage (wherever `HOOK_NAMES` counts are asserted), new `tests/test_pre_commit_hook.py`

**Interfaces:**
- Consumes: the hook template conventions in `hooks/` (bash, marker-appendable: first line `#!/usr/bin/env bash`, logic below).
- Produces: `HOOK_NAMES = ("post-checkout", "post-merge", "pre-push", "pre-commit")`; `hooks/pre-commit` blocks any staged non-deletion path at or under `kb/`.

- [ ] **Step 1: Write the failing test** (`tests/test_pre_commit_hook.py`)

```python
"""The pre-commit hook is the boundary layer .gitignore can't be:
`git add -f` defeats .gitignore; this refuses at commit time. Runs the
hook script directly via bash — no rcorn on PATH required, because a
guard that needs rcorn fails open exactly when it matters."""
import subprocess
from pathlib import Path

from reinicorn.git import reinicorn_root, run_git

HOOK = reinicorn_root() / "hooks" / "pre-commit"


def _run_hook(cwd: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        ["bash", str(HOOK)], cwd=cwd, capture_output=True, text=True,
    )


def test_blocks_staged_kb_file(kb_clone_repo):
    (kb_clone_repo / "kb" / "smuggled.md").write_text("x\n")
    run_git("add", "-f", "kb/smuggled.md", cwd=kb_clone_repo)
    r = _run_hook(kb_clone_repo)
    assert r.returncode == 1
    assert "rcorn kb publish" in r.stdout + r.stderr


def test_allows_clean_commit(kb_clone_repo):
    (kb_clone_repo / "src.py").write_text("x\n")
    run_git("add", "src.py", cwd=kb_clone_repo)
    assert _run_hook(kb_clone_repo).returncode == 0


def test_allows_kb_deletion(submodule_repo):
    """Migration's `git rm --cached kb` stages a DELETION of the gitlink —
    the hook must let the migration commit through."""
    run_git("rm", "--cached", "kb", cwd=submodule_repo)
    assert _run_hook(submodule_repo).returncode == 0
```

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_pre_commit_hook.py -v`
Expected: FAIL (no `hooks/pre-commit` file).

- [ ] **Step 3: Write `hooks/pre-commit`**

```bash
#!/usr/bin/env bash
# =============================================================================
# pre-commit — Reinicorn git hook
#
# Refuses to commit any path at or under kb/ into the parent repo. The kb
# is a separate repository that publishes through `rcorn kb publish`;
# .gitignore alone is a convention that `git add -f` and a stray
# `git submodule add` both defeat.
#
# Deletions are allowed (--diff-filter=d): the submodule->clone migration
# commits the REMOVAL of the old kb gitlink through this hook.
#
# Self-contained bash on purpose: a boundary guard that needs rcorn on
# PATH fails open exactly when it matters.
# =============================================================================

staged=$(git diff --cached --name-only --diff-filter=d -- kb 2>/dev/null)
if [ -n "$staged" ]; then
    echo "❌ Commit blocked: staged paths under kb/ belong to the shared kb, not this repo." >&2
    echo "$staged" | head -5 | sed 's/^/   /' >&2
    echo "   Publish kb changes with: rcorn kb publish" >&2
    echo "   Unstage with: git restore --staged -- kb" >&2
    exit 1
fi

exit 0
```

- [ ] **Step 4: Add `"pre-commit"` to `HOOK_NAMES` in `hooks_health.py` and fix any test asserting the old 3-hook count. Run `shellcheck hooks/pre-commit` (lint-kb runs shellcheck in CI).**

- [ ] **Step 5: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add hooks/pre-commit src/reinicorn/hooks_health.py tests/
git commit -m "feat(hooks): pre-commit guard refuses staged kb/ paths"
```

---

### Task 9: In-place migration via `rcorn init` / `rcorn update`

**Files:**
- Create: `src/reinicorn/kb_migrate.py`
- Modify: `src/reinicorn/commands/init.py` (call before `_init_path`), `src/reinicorn/commands/update.py` (call before the version early-return)
- Create: `upgrades/v0.2.md`
- Test: `tests/test_kb_migrate.py` (uses `submodule_repo` — this is why that fixture survives)

**Interfaces:**
- Consumes: `submodule_repo` fixture, `resolve_kb_remote_url`, `setup_kb_clone`, `ensure_kb_gitignored`, `KB_DIR_NAME`.
- Produces: `detect_submodule_layout(root: Path) -> bool` (index gitlink OR `.gitmodules` entry), `kb_unpublished_reason(kb_dir: Path) -> str | None`, `migrate_submodule_to_clone(root: Path) -> bool`.

- [ ] **Step 1: Write the failing tests**

```python
def test_detect_by_gitmodules(submodule_repo):
    assert detect_submodule_layout(submodule_repo) is True


def test_detect_orphan_gitlink(submodule_repo):
    """A tracked 160000 kb entry with no .gitmodules still migrates (spec §10)."""
    (submodule_repo / ".gitmodules").unlink()
    run_git("add", ".gitmodules", cwd=submodule_repo)
    run_git("commit", "-q", "-m", "orphan the gitlink", cwd=submodule_repo)
    assert detect_submodule_layout(submodule_repo) is True


def test_detect_clone_layout_is_false(kb_clone_repo):
    assert detect_submodule_layout(kb_clone_repo) is False


def test_migration_refuses_uncommitted_kb_work(submodule_repo):
    (submodule_repo / "kb" / "draft.md").write_text("unpublished draft\n")
    assert migrate_submodule_to_clone(submodule_repo) is False
    assert (submodule_repo / "kb" / "draft.md").read_text() == "unpublished draft\n"
    # Nothing destructive ran: still a submodule
    assert detect_submodule_layout(submodule_repo) is True


def test_migration_refuses_unpushed_kb_commits(submodule_repo):
    kb = submodule_repo / "kb"
    (kb / "draft.md").write_text("committed, unpushed\n")
    run_git("add", "-A", cwd=kb)
    run_git("commit", "-q", "-m", "local only", cwd=kb)
    assert migrate_submodule_to_clone(submodule_repo) is False
    assert detect_submodule_layout(submodule_repo) is True


def test_migration_converts_clean_repo(submodule_repo):
    assert migrate_submodule_to_clone(submodule_repo) is True
    kb = submodule_repo / "kb"
    assert (kb / ".git").is_dir()                       # plain clone now
    assert not (submodule_repo / ".gitmodules").exists()
    assert "kb/" in (submodule_repo / ".gitignore").read_text()
    # gitlink removal is staged for the user to commit
    r = run_git("diff", "--cached", "--name-only", cwd=submodule_repo)
    assert "kb" in r.stdout.splitlines()
    # submodule config gone
    r = run_git("config", "--get", "submodule.kb.url",
                check=False, cwd=submodule_repo)
    assert r.returncode != 0


def test_migration_handles_orphan_gitlink(submodule_repo):
    (submodule_repo / ".gitmodules").unlink()
    run_git("add", ".gitmodules", cwd=submodule_repo)
    run_git("commit", "-q", "-m", "orphan", cwd=submodule_repo)
    assert migrate_submodule_to_clone(submodule_repo) is True
    assert (submodule_repo / "kb" / ".git").is_dir()
```

(`submodule_repo` has no `.reinicorn-config`; the migration falls back to the old kb clone's own origin URL for the re-clone — assert that works, since real pre-migration repos may predate `REINICORN_KB_REMOTE` too.)

- [ ] **Step 2: Run to verify failure**

Run: `uv run pytest tests/test_kb_migrate.py -v`
Expected: FAIL (module missing).

- [ ] **Step 3: Implement `src/reinicorn/kb_migrate.py`**

```python
"""Migrate a repo from the kb-submodule layout to the plain-clone layout.

Detection inspects the index for a mode-160000 kb entry, not just
`.gitmodules` — an orphan gitlink with no `.gitmodules` section (or a
malformed `.gitmodules`) migrates too. The unpublished-work check runs
before anything destructive: every later step assumes the old kb
worktree is disposable, and losing a draft to a migration would be
unforgivable.
"""

from __future__ import annotations

import shutil
from pathlib import Path

from reinicorn import console
from reinicorn.config import KB_DIR_NAME
from reinicorn.git import run_git
from reinicorn.kb_remote import resolve_kb_remote_url
from reinicorn.kb_setup import KbSetupError, ensure_kb_gitignored, setup_kb_clone

_GITLINK_MODE = "160000"


def detect_submodule_layout(root: Path) -> bool:
    """True when the index tracks a kb gitlink or .gitmodules declares one."""
    r = run_git("ls-files", "-s", "--", KB_DIR_NAME, check=False, cwd=root)
    for line in r.stdout.splitlines():
        mode, _, rest = line.partition(" ")
        path = rest.split("\t", 1)[-1] if "\t" in rest else ""
        if mode == _GITLINK_MODE and path == KB_DIR_NAME:
            return True
    gitmodules = root / ".gitmodules"
    return (
        gitmodules.is_file()
        and f'[submodule "{KB_DIR_NAME}"]' in gitmodules.read_text()
    )


def kb_unpublished_reason(kb_dir: Path) -> str | None:
    """Why the old kb worktree cannot be discarded, or None when it can."""
    if not (kb_dir / ".git").exists():
        return None  # nothing checked out — nothing to lose
    dirty = run_git("status", "--porcelain", check=False, cwd=kb_dir)
    if dirty.stdout.strip():
        return "it has uncommitted changes"
    run_git("fetch", "origin", "main", check=False, cwd=kb_dir)
    ahead = run_git(
        "rev-list", "--count", "origin/main..HEAD", check=False, cwd=kb_dir,
    )
    if ahead.returncode != 0:
        return "its commits cannot be verified against origin/main (fetch failed?)"
    if ahead.stdout.strip() != "0":
        return "it has commits that are not on origin/main"
    return None


def migrate_submodule_to_clone(root: Path) -> bool:
    """Convert the kb submodule at KB_DIR_NAME into a gitignored plain clone.

    Step order mirrors spec §10: the refuse-if-unpublished check runs
    before any destructive command. Leaves the gitlink removal staged and
    prints exactly what to commit — never commits on the user's behalf.
    """
    kb_dir = root / KB_DIR_NAME

    reason = kb_unpublished_reason(kb_dir)
    if reason is not None:
        console.error(
            f"Refusing to migrate the kb submodule: {reason}.\n"
            f"  Where: {kb_dir}\n"
            "  How to fix: publish it first (rcorn kb publish), then rerun "
            "this command."
        )
        return False

    # Resolve the clone URL before dismantling anything that records it.
    url = resolve_kb_remote_url(root)
    if not url:
        console.error(
            "Cannot migrate: no kb remote URL could be resolved.\n"
            "  How to fix: set REINICORN_KB_REMOTE in .reinicorn-config, "
            "then rerun."
        )
        return False

    console.progress("Migrating kb from submodule to plain clone...")

    registered = run_git(
        "config", "--get", f"submodule.{KB_DIR_NAME}.url",
        check=False, cwd=root,
    ).returncode == 0
    if registered:
        run_git("submodule", "deinit", "-f", KB_DIR_NAME, check=False, cwd=root)
    run_git("rm", "-q", "--cached", "-f", KB_DIR_NAME, check=False, cwd=root)

    _strip_gitmodules_section(root)
    run_git(
        "config", "--remove-section", f"submodule.{KB_DIR_NAME}",
        check=False, cwd=root,
    )

    modules = _git_common_dir(root) / "modules" / KB_DIR_NAME
    if modules.exists():
        backup = modules.with_name(f"{KB_DIR_NAME}.pre-clone-migration")
        if backup.exists():
            shutil.rmtree(backup)
        shutil.move(str(modules), str(backup))
    if kb_dir.exists():
        shutil.rmtree(kb_dir)

    try:
        setup_kb_clone(root, url)
    except KbSetupError as e:
        console.error(str(e))
        console.next_step("rcorn kb sync")
        return False

    ensure_kb_gitignored(root)

    console.success("Kb migrated to a plain clone.")
    print()
    console.info("The gitlink removal is staged. Commit it yourself:")
    # .gitmodules edits need staging only when the file survived (other
    # submodules); its full deletion was already staged by the strip helper.
    extra = " .gitmodules" if (root / ".gitmodules").is_file() else ""
    console.info(f"  git add .gitignore{extra}")
    console.info("  git commit -m 'chore: migrate kb from submodule to clone'")
    return True


def _strip_gitmodules_section(root: Path) -> None:
    """Remove the kb section from .gitmodules; delete the file when empty."""
    gitmodules = root / ".gitmodules"
    if not gitmodules.is_file():
        return
    kept: list[str] = []
    in_kb = False
    for line in gitmodules.read_text().splitlines():
        stripped = line.strip()
        if stripped.startswith("["):
            in_kb = stripped == f'[submodule "{KB_DIR_NAME}"]'
        if not in_kb:
            kept.append(line)
    if any(line.strip() for line in kept):
        gitmodules.write_text("\n".join(kept) + "\n")
    else:
        gitmodules.unlink()
        run_git("rm", "-q", "--cached", "--ignore-unmatch", ".gitmodules",
                check=False, cwd=root)


def _git_common_dir(root: Path) -> Path:
    r = run_git("rev-parse", "--git-common-dir", check=False, cwd=root)
    common = Path(r.stdout.strip() or ".git")
    return common if common.is_absolute() else root / common
```

(While implementing, fix the f-string ruff will flag in the commit-hint line — plain string.)

- [ ] **Step 4: Wire in**

`init.py` — at the top of `cmd_init` right after the git-repo check:

```python
    from reinicorn.kb_migrate import detect_submodule_layout, migrate_submodule_to_clone
    if detect_submodule_layout(cwd):
        console.info("Detected the old kb submodule layout.")
        if not migrate_submodule_to_clone(cwd):
            return 1
```

`update.py` — in `cmd_update` right after `read_manifest` succeeds (BEFORE the `manifest_version == pkg_version` early-return, so an up-to-date repo still migrates):

```python
    from reinicorn.kb_migrate import detect_submodule_layout, migrate_submodule_to_clone
    if detect_submodule_layout(repo_root):
        console.info("Detected the old kb submodule layout.")
        if not migrate_submodule_to_clone(repo_root):
            return 1
```

- [ ] **Step 5: Write `upgrades/v0.2.md`**

```markdown
# v0.2 — the kb submodule is gone

The kb is now an ordinary gitignored clone at `kb/`, not a git submodule.
No pointer to bump, no drift, no `submodules: recursive` in CI.

## What happens on `rcorn update` / `rcorn init`

Repos still on the submodule layout are migrated in place:

1. The migration REFUSES to run while the old `kb/` has uncommitted or
   unpushed work — publish it first (`rcorn kb publish`).
2. The submodule is deregistered, the gitlink and `.gitmodules` entry are
   removed (removal left staged for you to commit), `.git/modules/kb` is
   set aside as `.git/modules/kb.pre-clone-migration`, and `kb/` is
   re-cloned fresh from `REINICORN_KB_REMOTE`.
3. `kb/` is added to `.gitignore` and a `pre-commit` hook now refuses any
   staged path under `kb/` (run `rcorn hooks install` to get it).

Commit the staged changes when it finishes:

    git add .gitignore .gitmodules
    git commit -m "chore: migrate kb from submodule to clone"

## CI

Drop `submodules: recursive` from `actions/checkout` and clone instead:

    - name: Clone kb
      run: git clone --depth 1 <your-kb-url> kb
```

- [ ] **Step 6: Run the full gate**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/reinicorn/kb_migrate.py src/reinicorn/commands/init.py src/reinicorn/commands/update.py upgrades/v0.2.md tests/test_kb_migrate.py
git commit -m "feat(migrate): rcorn init/update convert the kb submodule to a clone in place"
```

---

### Task 10: Migrate this repository itself

**Files:**
- Delete (tracked): `.gitmodules`, the `kb` gitlink
- Modify: `.gitignore`, `.reinicorn-config` (comment text for `REINICORN_KB_REMOTE`)

**Interfaces:**
- Consumes: Task 9's migration (run it for real).
- Produces: this repo on the clone layout — Task 11's CI check requires it.

- [ ] **Step 1: Confirm the kb has nothing unpublished**

Run: `cd kb && git status --porcelain && git log origin/main..HEAD --oneline; cd ..`
Expected: both empty. (Publish first if not: `rcorn kb publish`.)

- [ ] **Step 2: Run the real migration**

Run: `uv run rcorn update` (or `uv run python -c "from pathlib import Path; from reinicorn.kb_migrate import migrate_submodule_to_clone; migrate_submodule_to_clone(Path.cwd())"` if the manifest version matches and update exits early before you added the pre-check — it should not, per Task 9).
Expected: "Kb migrated to a plain clone." — `.gitmodules` gone, gitlink removal staged, `kb/` a fresh clone on `main`, `kb/` in `.gitignore`.

- [ ] **Step 3: Update the `.reinicorn-config` comment**

Replace the `REINICORN_KB_REMOTE` comment block:

```
# Kb remote URL. The kb is an ordinary clone at kb/ (gitignored);
# sync/publish and the bootstrap clone read this to know where to push
# and pull.
```

- [ ] **Step 4: Verify and commit**

Run: `git status --short` — expect staged `D .gitmodules`, `D kb` (gitlink), modified `.gitignore` + `.reinicorn-config`; then `git ls-files -- kb` — expect empty; then `uv run rcorn kb status` still works.

```bash
git add .gitignore .reinicorn-config
git commit -m "chore: migrate this repo's kb from submodule to clone"
```

(The pre-commit hook from Task 8 must allow this commit — it stages only deletions under `kb`. If it blocks, that is a Task 8 bug; fix there, not with --no-verify.)

---

### Task 11: CI reads current kb main; boundary check

**Files:**
- Modify: `.github/workflows/test.yml`, `.github/workflows/lint-kb.yml`, `.github/workflows/lint-architecture.yml`, `.github/workflows/doc-gardening.yml`

**Interfaces:**
- Consumes: the public kb URL `https://github.com/crystldm/reinicorn-kb.git` (unauthenticated clone; fork PRs work unchanged).
- Produces: no workflow uses `submodules: recursive`; every job that reads `kb/` clones it; `test.yml` fails when the parent tree tracks anything under `kb/`.

- [ ] **Step 1: Edit all four workflows**

In each, remove `submodules: recursive` from the `actions/checkout` `with:` block (keep `persist-credentials: false`, and doc-gardening's `fetch-depth: 0`), and add immediately after the checkout step:

```yaml
      - name: Clone kb
        run: git clone --depth 1 https://github.com/crystldm/reinicorn-kb.git kb
```

**Exception — `doc-gardening.yml` clones with full history** (staleness ages come from `git -C kb log`, which `--depth 1` would falsify):

```yaml
      - name: Clone kb
        run: git clone https://github.com/crystldm/reinicorn-kb.git kb
```

Update doc-gardening's comment "Kb files live in a submodule" → "The kb is cloned separately".

- [ ] **Step 2: Add the boundary check to `test.yml`** (before the lint step)

```yaml
      # The server-side kb boundary: .gitignore and the pre-commit hook are
      # client-side and bypassable (git add -f, --no-verify). Anything
      # tracked under kb/ — file or 160000 gitlink — fails the build.
      - name: Kb boundary check
        run: |
          tracked=$(git ls-files -- kb)
          if [ -n "$tracked" ]; then
            echo "::error::Tracked paths under kb/ — the kb publishes via its own repo (rcorn kb publish):"
            echo "$tracked"
            exit 1
          fi
```

(NOTE: the clone step must run AFTER this check or the check must run against the checkout only — `git ls-files` reads the index, not the worktree, so order does not matter for correctness; keep the boundary check first for a faster fail.)

- [ ] **Step 3: Validate workflow syntax**

Run: `uv run python -c "import yaml,glob; [yaml.safe_load(open(f)) for f in glob.glob('.github/workflows/*.yml')]; print('ok')"`
Expected: `ok`.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows
git commit -m "ci: clone the kb instead of submodule checkout; enforce the kb boundary"
```

---

### Task 12: Sweep the remaining submodule references; full gate

**Files:**
- Modify: `README.md` (intro line 61, tree line 90, line 177, section "KB as a submodule" 248-292 incl. the further-reading link), `AGENTS.md:19`, `src/reinicorn/commands/kb_git.py` (docstrings/comments), `src/reinicorn/linter/spec_refs.py:107-109` (comment), `linters/rules/scripts/shellcheck.sh:5` (comment), any hits the grep below still finds
- Test: none new (prose/comments), full suite + kb lint as the gate

**Interfaces:** none — prose only. Do not touch `kb_migrate.py`/`kb_setup.py`'s own "submodule" wording (they are ABOUT the old layout) or the spec/history docs in `kb/`.

- [ ] **Step 1: Find every stale reference**

Run: `grep -rn "submodule\|gitmodules" --include='*.py' --include='*.md' --include='*.yml' --include='*.sh' src tests hooks editor-hooks linters .github README.md AGENTS.md CLAUDE.md | grep -vi "kb_migrate\|kb_setup\|test_kb_migrate\|pre-clone-migration\|upgrades/"`

- [ ] **Step 2: Rewrite each hit.** README's "KB as a submodule" section becomes "KB as a shared clone": the kb is an ordinary gitignored clone at `kb/` tracking `main`; `rcorn kb sync` bootstraps it on a fresh clone; multi-repo support is unchanged (several repos share one kb, each under its own scope directory); the boundary is `.gitignore` + the pre-commit hook + CI. Replace the git-submodules further-reading link with nothing (or a link to the spec in the kb). `AGENTS.md:19` → "never manage the kb checkout with raw Git". `spec_refs.py` comment: the kb is a plain clone in every checkout — the two enumerable cases collapse, simplify the wording (code unchanged).

- [ ] **Step 3: Full gate, including kb lint**

Run: `uv run ruff check src/reinicorn tests && uv run pyright src/reinicorn && uv run pytest tests/ && uv run rcorn kb lint`
Expected: all PASS.

- [ ] **Step 4: Verify the whole change against the spec's Design Goals**

- No pointer: `git ls-files -- kb` empty, `grep -rn stage_kb_pointer src` empty.
- CI reads current main: no `submodules: recursive` in `.github/workflows/`.
- One-command bootstrap: fresh-clone test from Task 6 green.
- Boundary: Task 8 hook test + Task 11 CI step green.
- `commit_kb` refuses off-main: Task 2 tests green.
- Migration automatic: Task 9 tests green; this repo migrated (Task 10).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "docs: the kb is a shared clone, not a submodule"
```

---

## Dependencies

- Spec `reinicorn/specs/remove-the-kb-submodule.md` (approved, PR #8).
- Tasks are ordered to keep every commit green; Task 10 (repo self-migration) must precede Task 11 (CI boundary check would fail on the tracked gitlink).
- After merge: teammates run `rcorn update` (or `rcorn init`) once to migrate their clones; announce the `upgrades/v0.2.md` note.
