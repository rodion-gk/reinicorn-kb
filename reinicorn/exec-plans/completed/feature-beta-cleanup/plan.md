---
type: plan
title: 'Execution Plan: feature/beta-cleanup'
slug: feature-beta-cleanup
lifecycle: done
status: complete
created: 2026-07-05
author: Michael Biehl
branch: feature/beta-cleanup
ticket: N/A
spec: kb/reins/specs/beta-cleanup-focus-repo-on-kb-cli-skills-cut-extraneous-feat.md
---

# Execution Plan: feature/beta-cleanup

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Goal

Strip the repo to the first beta candidate — a Skill Set + CLI for AI knowledgebase management and team spec-driven development — by deleting extraneous features, repairing/wiring all hooks, moving skills to the open-standard `.agents/skills/` location, adding Codex (dropping Gemini), scaffolding only workflow docs we enforce, and cleaning the kb.

**Architecture:** Pure subtraction plus small repairs in the existing `src/reins` CLI. No new modules except tests. Kb changes go through the reins CLI (`idea create`, `kb publish`); kb file moves use plain `mv` (never raw `git` in `kb/`).

**Tech Stack:** Python 3.12+ (stdlib only), pytest, bash hooks, hatchling wheel packaging.

## Working conventions (read first)

- Run all commands from the repo root. Tests: `uv run pytest <path> -v` (or `pytest` if the venv is active). Full suite: `uv run pytest`.
- NEVER run raw `git` inside `kb/` — use `reins kb publish` to commit+push kb changes, `reins kb git <args>` only if raw git is unavoidable.
- After each kb-touching task, `reins kb publish` then commit the parent-repo `kb` pointer bump: `git add kb && git commit -m "chore(kb): <what>"`.
- Commit messages end with:

```
Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
```

## Acceptance Criteria

- [ ] `scripts/`, `extensions/`, `tools/`, `src/reins/extensions.py`, `tests/test_extensions.py`, and the four loose legacy skill files are gone; no stray references remain (`grep -ri` clean).
- [ ] Each cut feature has an idea doc in `kb/reins/ideas/`.
- [ ] `hooks/post-checkout` is a thin stub delegating to `reins _post-checkout`; the suggestion text says `reins plan create`; `_post-checkout` and `_post-merge` have tests.
- [ ] `reins hooks install` installs BOTH editor hooks (enforce-doc-templates for Write|Edit, block-raw-kb-git for Bash) into Claude/Cursor/Copilot configs, with tests.
- [ ] Gemini support removed; Codex in the init picker (AGENTS.md-native, no extra file); tests updated.
- [ ] Skills install canonically to `.agents/skills/` with `.claude/skills` symlink (copy fallback); repo itself converted; wheel still ships skills; update/manifest paths updated.
- [ ] `plan create` scaffolds only `plan.md`; `retro create` writes into the active plan dir with registry-aligned sections; `plan complete` warns on missing/empty retro; finishing-a-development-branch skill has a retro step.
- [ ] Kb: agent-tasks/agent-jobs gone, dead active plans archived, superseded/implemented specs marked, cleanup-queue pruned.
- [ ] README rewritten around the beta; HELLO.md renamed HUMANS.md; GETTING-STARTED verified.
- [ ] Full `uv run pytest`, `uv run ruff check`, `uv run pyright`, and `reins kb lint` pass.

## Tasks

### Task 1: Delete extraneous features from the code repo

**Files:**
- Delete: `scripts/` (4 files), `extensions/` (all), `tools/` (all), `src/reins/extensions.py`, `tests/test_extensions.py`
- Delete: `.claude/skills/analyze-codebase.md`, `.claude/skills/publish-plan.md`, `.claude/skills/sync-context.md`, `.claude/skills/update-kb.md`

- [ ] **Step 1: Confirm nothing imports the extensions module**

Run: `grep -rn "extensions" src/reins --include="*.py" -l`
Expected: only `src/reins/extensions.py` itself.

- [ ] **Step 2: Delete**

```bash
git rm -r -q scripts extensions tools src/reins/extensions.py tests/test_extensions.py \
  .claude/skills/analyze-codebase.md .claude/skills/publish-plan.md \
  .claude/skills/sync-context.md .claude/skills/update-kb.md
```

- [ ] **Step 3: Verify the suite still passes and no stray references**

Run: `uv run pytest -q`
Expected: all pass (test_extensions.py is gone).

Run: `grep -rn "apply-variant\|attach-to-repo\|init-greenfield\|setup-hooks\|block-dangerous\|extensions/" src tests hooks editor-hooks GETTING-STARTED.md AGENTS.md pyproject.toml`
Expected: no output. README hits are fine (rewritten in Task 12). If `tests/run-all.sh` or docs reference deleted paths, remove those lines.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat!: remove legacy scripts, extension system, and orphaned tools for beta"
```

### Task 2: File idea docs for the cut features

- [ ] **Step 1: Create four idea docs**

```bash
reins idea create "Pip-free shell bootstrap path for environments without Python tooling (replaced scripts/ deleted for beta)"
reins idea create "Extension overlay system for language stacks and integrations (extensions/ deleted for beta; manifest.json copy/append/hooks design in git history)"
reins idea create "block-dangerous-commands standalone Go guard tool (tools/ deleted for beta; see spec block-dangerous-commands-go-rewrite)"
reins idea create "Revive analyze-codebase attach workflow as a proper skill (loose skill file deleted for beta)"
```

- [ ] **Step 2: Publish and bump pointer**

```bash
reins kb publish
git add kb && git commit -m "chore(kb): file idea docs for features cut in beta cleanup"
```

### Task 3: post-checkout hook — thin stub + fixed suggestion + tests

**Files:**
- Modify: `hooks/post-checkout` (replace entirely)
- Modify: `src/reins/commands/internal/post_checkout.py:58-71`
- Test: `tests/commands/internal/test_post_checkout.py` (new)

- [ ] **Step 1: Write failing tests**

Create `tests/commands/internal/test_post_checkout.py`. Uses the existing `submodule_repo` fixture from `tests/conftest.py` (same as `test_pre_push.py`).

```python
"""Tests for reins _post-checkout — submodule init + new-branch suggestion."""

from __future__ import annotations

import subprocess
from pathlib import Path

from reins.commands.internal.post_checkout import cmd_post_checkout

def test_file_checkout_is_noop(submodule_repo: Path, monkeypatch, capsys):
    """checkout_type '0' (file checkout) → exit 0, no output."""
    monkeypatch.chdir(submodule_repo)
    assert cmd_post_checkout(["a", "b", "0"]) == 0
    assert capsys.readouterr().out == ""

def test_disabled_mode_is_noop(submodule_repo: Path, monkeypatch, capsys):
    """Disabled mode → no suggestion printed."""
    (submodule_repo / ".reins-mode").write_text("disabled")
    monkeypatch.chdir(submodule_repo)
    assert cmd_post_checkout(["a", "b", "1"]) == 0
    assert capsys.readouterr().out == ""

def test_new_branch_suggests_plan_create(submodule_repo: Path, monkeypatch, capsys):
    """New branch without upstream → suggests 'reins plan create'."""
    subprocess.run(
        ["git", "checkout", "-q", "-b", "feature-new-thing"],
        cwd=submodule_repo, check=True,
    )
    monkeypatch.chdir(submodule_repo)
    assert cmd_post_checkout(["a", "b", "1"]) == 0
    out = capsys.readouterr().out
    assert "feature-new-thing" in out
    assert "reins plan create" in out
    assert "/create-exec-plan" not in out

def test_ticket_id_detected_in_branch(submodule_repo: Path, monkeypatch, capsys):
    """Branch containing a JIRA-style ticket id → id shown in suggestion."""
    subprocess.run(
        ["git", "checkout", "-q", "-b", "feature/ABC-123-do-thing"],
        cwd=submodule_repo, check=True,
    )
    monkeypatch.chdir(submodule_repo)
    assert cmd_post_checkout(["a", "b", "1"]) == 0
    out = capsys.readouterr().out
    assert "ABC-123" in out
    assert "reins plan create" in out
```

- [ ] **Step 2: Run to verify the suggestion tests fail**

Run: `uv run pytest tests/commands/internal/test_post_checkout.py -v`
Expected: `test_new_branch_suggests_plan_create` and `test_ticket_id_detected_in_branch` FAIL (output still says `/create-exec-plan`); the two no-op tests pass.

- [ ] **Step 3: Fix the suggestion text in Python**

In `src/reins/commands/internal/post_checkout.py`, replace lines 58-71 with:

```python
        print()
        if ticket_id:
            print(f"reins: new branch '{branch}' (ticket: {ticket_id})")
            print(
                f"  Run 'reins plan create' to set up an execution plan "
                f"with {ticket_id} context."
            )
        else:
            print(f"reins: new branch '{branch}'")
            print(
                "  Run 'reins plan create' to set up an execution plan "
                "for this work."
            )
        print()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/commands/internal/test_post_checkout.py -v`
Expected: 4 passed.

- [ ] **Step 5: Replace the bash hook with a thin stub**

Replace the entire content of `hooks/post-checkout` with:

```bash
#!/usr/bin/env bash
# =============================================================================
# post-checkout — reins git hook
#
# Runs after `git checkout` and `git worktree add`. Responsibilities:
#   1. Ensure kb submodule exists (fresh clone / new worktree)
#   2. On new branch checkout, suggest `reins plan create`
#
# Delegates all logic to `reins _post-checkout` (Python).
#
# Arguments (provided by git):
#   $1 = previous HEAD ref
#   $2 = new HEAD ref
#   $3 = flag: "1" = branch checkout, "0" = file checkout
# =============================================================================

if command -v reins &>/dev/null; then
    reins _post-checkout "$1" "$2" "$3" 2>/dev/null || true
fi

exit 0
```

Note: `cmd_post_checkout` reads `args[2]` for the checkout flag, so the hook must pass all three git args through — verify `_dispatch_internal` in `src/reins/cli.py:230-251` forwards the remaining argv list to the handler.

- [ ] **Step 6: Verify the stub works end-to-end**

```bash
bash hooks/post-checkout HEAD HEAD 1
```

Expected: prints the `reins plan create` suggestion for the current branch if it has no upstream, or nothing. Must NOT error.

- [ ] **Step 7: Commit**

```bash
git add hooks/post-checkout src/reins/commands/internal/post_checkout.py tests/commands/internal/test_post_checkout.py
git commit -m "fix(hooks): post-checkout delegates to Python, suggests 'reins plan create'"
```

### Task 4: Tests for `_post-merge`

**Files:**
- Read first: `src/reins/commands/internal/post_merge.py` (76 lines — archives stale active plans whose branch is merged)
- Test: `tests/commands/internal/test_post_merge.py` (new)

- [ ] **Step 1: Read the implementation**

Read `src/reins/commands/internal/post_merge.py` fully. It exposes `cmd_post_merge(args)`; identify its stale-plan-archival helper and mode gating. The tests below assume it archives `active/<branch>/` to `completed/` for branches merged into the current branch and no-ops when mode is disabled; adjust setup details to match the real contract.

- [ ] **Step 2: Write tests**

Create `tests/commands/internal/test_post_merge.py`:

```python
"""Tests for reins _post-merge — stale plan archival."""

from __future__ import annotations

import subprocess
from pathlib import Path

from reins.commands.internal.post_merge import cmd_post_merge

def _mk_active_plan(repo: Path, slug: str, branch_dir: str) -> Path:
    active = repo / "kb" / slug / "exec-plans" / "active" / branch_dir
    active.mkdir(parents=True)
    (active / "plan.md").write_text(
        f"# Execution Plan: {branch_dir}\n\n**Status:** complete\n"
    )
    subprocess.run(["git", "add", "-A"], cwd=repo / "kb", check=True)
    subprocess.run(["git", "commit", "-q", "-m", "plan"], cwd=repo / "kb", check=True)
    return active

def test_disabled_mode_is_noop(submodule_repo: Path, monkeypatch):
    (submodule_repo / ".reins-mode").write_text("disabled")
    monkeypatch.chdir(submodule_repo)
    assert cmd_post_merge([]) == 0

def test_merged_branch_plan_archived(submodule_repo: Path, monkeypatch):
    """A plan for a branch already merged into HEAD moves to completed/."""
    slug = "unknown"  # repo_slug() for a repo with no remote
    subprocess.run(["git", "checkout", "-q", "-b", "feature-done"], cwd=submodule_repo, check=True)
    (submodule_repo / "f.txt").write_text("x\n")
    subprocess.run(["git", "add", "f.txt"], cwd=submodule_repo, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "work"], cwd=submodule_repo, check=True)
    subprocess.run(["git", "checkout", "-q", "main"], cwd=submodule_repo, check=True)
    subprocess.run(["git", "merge", "-q", "feature-done"], cwd=submodule_repo, check=True)

    _mk_active_plan(submodule_repo, slug, "feature-done")
    monkeypatch.chdir(submodule_repo)

    assert cmd_post_merge([]) == 0
    assert not (submodule_repo / "kb" / slug / "exec-plans" / "active" / "feature-done").exists()
    assert (
        submodule_repo / "kb" / slug / "exec-plans" / "completed" / "feature-done" / "plan.md"
    ).is_file()

def test_unmerged_branch_plan_kept(submodule_repo: Path, monkeypatch):
    """A plan whose branch is NOT merged stays in active/."""
    slug = "unknown"
    subprocess.run(["git", "checkout", "-q", "-b", "feature-wip"], cwd=submodule_repo, check=True)
    (submodule_repo / "g.txt").write_text("y\n")
    subprocess.run(["git", "add", "g.txt"], cwd=submodule_repo, check=True)
    subprocess.run(["git", "commit", "-q", "-m", "wip"], cwd=submodule_repo, check=True)
    subprocess.run(["git", "checkout", "-q", "main"], cwd=submodule_repo, check=True)

    _mk_active_plan(submodule_repo, slug, "feature-wip")
    monkeypatch.chdir(submodule_repo)

    assert cmd_post_merge([]) == 0
    assert (
        submodule_repo / "kb" / slug / "exec-plans" / "active" / "feature-wip" / "plan.md"
    ).is_file()
```

- [ ] **Step 3: Run, adapt setup to actual behavior, make green**

Run: `uv run pytest tests/commands/internal/test_post_merge.py -v`
If a test fails because the real behavior differs (e.g. it only archives on the default branch), adjust the TEST setup to satisfy the real contract — do not change `post_merge.py` unless it is actually broken. If it IS broken (crashes), fix minimally and note it in the PR description.

- [ ] **Step 4: Commit**

```bash
git add tests/commands/internal/test_post_merge.py
git commit -m "test(hooks): cover _post-merge stale plan archival"
```

### Task 5: Wire block-raw-kb-git.sh into `reins hooks install`

**Files:**
- Modify: `src/reins/commands/hooks_install.py:14-24` (constants), `:97-129` (`_install_editor_hooks`)
- Test: `tests/test_init_hook.py` (extend)

- [ ] **Step 1: Write failing test**

Append to `tests/test_init_hook.py` (read `test_init_installs_session_hook` first and reuse its setup pattern):

```python
def test_hooks_install_wires_both_editor_hooks(tmp_path: Path, monkeypatch):
    """hooks install copies both editor hook scripts and registers both matchers."""
    import json
    import subprocess
    from reins.commands.hooks_install import cmd_hooks_install

    subprocess.run(["git", "init", "-q", str(tmp_path)], check=True)
    monkeypatch.chdir(tmp_path)
    assert cmd_hooks_install() == 0

    assert (tmp_path / ".reins/hooks/enforce-doc-templates.sh").is_file()
    assert (tmp_path / ".reins/hooks/block-raw-kb-git.sh").is_file()

    settings = json.loads((tmp_path / ".claude/settings.json").read_text())
    matchers = {e["matcher"] for e in settings["hooks"]["PreToolUse"]}
    assert matchers == {"Write|Edit", "Bash"}

    cursor = json.loads((tmp_path / ".cursor/hooks.json").read_text())
    commands = {e["command"] for e in cursor["hooks"]["preToolUse"]}
    assert commands == {
        ".reins/hooks/enforce-doc-templates.sh",
        ".reins/hooks/block-raw-kb-git.sh",
    }

    copilot = json.loads((tmp_path / ".github/hooks/reins.json").read_text())
    bashes = {e["bash"] for e in copilot["hooks"]["preToolUse"]}
    assert bashes == {
        ".reins/hooks/enforce-doc-templates.sh",
        ".reins/hooks/block-raw-kb-git.sh",
    }
```

Run: `uv run pytest tests/test_init_hook.py::test_hooks_install_wires_both_editor_hooks -v`
Expected: FAIL (block-raw-kb-git.sh not installed).

- [ ] **Step 2: Generalize the constants**

In `src/reins/commands/hooks_install.py` replace `_HOOK_SCRIPT`, `_CLAUDE_ENTRY`, `_CURSOR_ENTRY`, `_COPILOT_ENTRY` (lines 16-24) with:

```python
# Editor PreToolUse hook scripts: (script filename, Claude Code tool matcher)
_EDITOR_HOOK_SCRIPTS = (
    ("enforce-doc-templates.sh", "Write|Edit"),
    ("block-raw-kb-git.sh", "Bash"),
)

def _claude_entry(script: str, matcher: str) -> dict:
    return {
        "matcher": matcher,
        "hooks": [{"type": "command", "command": f"{_SCRIPT_DEST}/{script}"}],
    }

def _cursor_entry(script: str, matcher: str) -> dict:
    return {"command": f"{_SCRIPT_DEST}/{script}", "matcher": matcher}

def _copilot_entry(script: str, _matcher: str) -> dict:
    return {"type": "command", "bash": f"{_SCRIPT_DEST}/{script}"}
```

Keep `_SCRIPT_DEST` as-is. Check for other usages first: `grep -rn "_CLAUDE_ENTRY\|_HOOK_SCRIPT" src tests`.

- [ ] **Step 3: Loop over both scripts in `_install_editor_hooks`**

Replace the single-file copy block (lines 113-129) with:

```python
    claude_entries: list[dict] = []
    cursor_entries: list[dict] = []
    copilot_entries: list[dict] = []

    for script, matcher in _EDITOR_HOOK_SCRIPTS:
        src = editor_hooks_src / script
        if not src.is_file():
            console.warn(f"SKIP: {script} — source not found")
            continue

        dest = dest_dir / script
        if dest.is_file() and dest.read_text() == src.read_text():
            console.warn(f"SKIP: {script} — already up to date")
        else:
            shutil.copy2(src, dest)
            dest.chmod(dest.stat().st_mode | stat.S_IEXEC)
            console.success(f"INSTALLED: {script}")

        claude_entries.append(_claude_entry(script, matcher))
        cursor_entries.append(_cursor_entry(script, matcher))
        copilot_entries.append(_copilot_entry(script, matcher))

    # Merge hook entries into each editor's config
    _merge_claude_settings(root / ".claude" / "settings.json", claude_entries)
    _merge_cursor_settings(root / ".cursor" / "hooks.json", cursor_entries)
    _merge_copilot_settings(root / ".github" / "hooks" / "reins.json", copilot_entries)
```

The merge functions already take lists and dedupe by matcher/command/bash — no changes needed there.

- [ ] **Step 4: Run tests**

Run: `uv run pytest tests/test_init_hook.py tests/test_guardrail_hook.py -v`
Expected: all pass, including the new test.

- [ ] **Step 5: Install into this repo and verify**

```bash
reins hooks install
cat .claude/settings.json
```
Expected: PreToolUse now has both `Write|Edit` and `Bash` matcher entries; `.reins/hooks/block-raw-kb-git.sh` exists.

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/hooks_install.py tests/test_init_hook.py .claude/settings.json
git commit -m "feat(hooks): install block-raw-kb-git editor hook alongside doc-templates guard"
```

### Task 6: Platforms — remove Gemini, add Codex

**Files:**
- Modify: `src/reins/commands/init.py:46-58` (registries), `:465-487` (`_prompt_platforms`), `_install_platform_instructions` (~`:490`)
- Delete: `platform-instructions/gemini.md`
- Modify: `AGENTS.md` (skills note)
- Test: `tests/test_init_platforms.py`

- [ ] **Step 1: Update tests first**

In `tests/test_init_platforms.py`, change `test_init_generates_all_platforms` from `["claude", "cursor", "copilot", "gemini"]` to `["claude", "cursor", "copilot", "codex"]`; assert `GEMINI.md` is NOT created and no codex-specific file is created. Add:

```python
def test_codex_platform_installs_no_extra_file(tmp_path: Path):
    """Codex reads AGENTS.md natively — selecting it must not write any new file."""
    from reins.commands.init import _install_platform_instructions

    before = set(tmp_path.rglob("*"))
    _install_platform_instructions(tmp_path, "myrepo", ["codex"])
    after = set(tmp_path.rglob("*"))
    assert before == after
```

Run: `uv run pytest tests/test_init_platforms.py -v` — expected: FAIL.

- [ ] **Step 2: Update the registries**

In `src/reins/commands/init.py` remove the `"gemini"` entries from both `PLATFORM_FILES` and `PLATFORM_TEMPLATES` (do NOT add codex entries — it has no file).

- [ ] **Step 3: Update the picker**

In `_prompt_platforms`, replace `("gemini", "Gemini CLI", False)` with `("codex", "Codex", False)`.

- [ ] **Step 4: Special-case codex in `_install_platform_instructions`**

At the top of the platform loop body, before the template lookup:

```python
        if platform == "codex":
            console.success("Codex: uses AGENTS.md (already installed)")
            continue
```

- [ ] **Step 5: Delete the template and sweep references**

```bash
git rm platform-instructions/gemini.md
grep -rin "gemini" src tests AGENTS.md GETTING-STARTED.md platform-instructions
```
Expected after fixes: no remaining `gemini` references (README handled in Task 12).

- [ ] **Step 6: Add the AGENTS.md note**

In `AGENTS.md`, find the section describing skills and add one line after the skills-location sentence:

```markdown
Platforms without a skill-invocation tool (e.g. Codex) read skills directly from the skills directory: each skill is a `SKILL.md` you follow as instructions.
```

- [ ] **Step 7: Run tests and commit**

Run: `uv run pytest tests/test_init_platforms.py tests/test_multi_repo_init.py -v`
Expected: pass.

```bash
git add -A && git commit -m "feat(platforms): add Codex (AGENTS.md-native), remove Gemini support"
```

### Task 7: Canonical `.agents/skills/` with `.claude/skills` symlink

**Files:**
- Modify: `src/reins/commands/init.py:45` (`SKILLS_ASSET`), `_copy_skills` (~`:365`), `_check_skill_collisions` (~`:312`)
- Modify: `src/reins/commands/update.py:39,153` (+docstrings at `:35,146-147`)
- Modify: `src/reins/manifest.py:19-24` (`MANAGED_ASSETS`)
- Modify: `pyproject.toml` (force-include)
- Repo layout: `git mv .claude/skills .agents/skills` + symlink
- Modify: `AGENTS.md`, `platform-instructions/*.md`, `GETTING-STARTED.md` (path references)
- Test: `tests/test_skill_copy.py`, `tests/test_assets.py`, `tests/test_update.py`, `tests/test_manifest.py`

- [ ] **Step 1: Convert this repo's layout**

```bash
mkdir .agents
git mv .claude/skills .agents/skills
ln -s ../.agents/skills .claude/skills
git add .claude/skills .agents
git status --short | head -30
```
Expected: renames staged plus a new `.claude/skills` symlink. (Claude Code follows the symlink, so the session keeps its skills.)

- [ ] **Step 2: Update the wheel packaging**

In `pyproject.toml` `[tool.hatch.build.targets.wheel.force-include]`, change:

```toml
".claude/skills" = "reins/_data/skills"
```
to
```toml
".agents/skills" = "reins/_data/skills"
```

- [ ] **Step 3: Update failing tests to the new contract**

In `tests/test_skill_copy.py::test_init_copies_expected_skills`, assert the new layout:

```python
    skills = tmp_path / ".agents" / "skills"
    assert (skills / "using-reins" / "SKILL.md").is_file()
    link = tmp_path / ".claude" / "skills"
    assert link.is_symlink()
    assert link.resolve() == skills.resolve()
```

Also update any `".claude/skills"` literals in `tests/test_assets.py`, `tests/test_update.py`, `tests/test_manifest.py` (find with `grep -rn "claude/skills" tests/`).

Run: `uv run pytest tests/test_skill_copy.py -v` — expected: FAIL (code still installs to .claude/skills).

- [ ] **Step 4: Update init.py**

Line 45:
```python
SKILLS_ASSET = ".agents/skills"
```

Rewrite `_copy_skills` destination + add the symlink helper (keep the existing skill-name listing and `_check_skill_collisions(all_skills)` tail; change the success message to say `.agents/skills/`):

```python
def _copy_skills(_r_root: Path, target_dir: Path) -> None:
    """Copy skills to <target>/.agents/skills/ and link .claude/skills to it."""
    skills_src = get_asset_path("skills")
    if skills_src is None:
        skills_src = get_asset_path(SKILLS_ASSET)
    if skills_src is None:
        console.warn("No skills directory found — skipping")
        return

    skills_dest = target_dir / SKILLS_ASSET
    skills_dest.mkdir(parents=True, exist_ok=True)
    shutil.copytree(skills_src, skills_dest, dirs_exist_ok=True)
    _link_claude_skills(target_dir)
```

```python
def _link_claude_skills(target_dir: Path) -> None:
    """Make .claude/skills a symlink to ../.agents/skills (copy fallback).

    Claude Code only reads .claude/skills; Codex/Cursor/Copilot read
    .agents/skills natively. A pre-existing REAL .claude/skills dir is
    left untouched (never delete user content) — warn instead.
    """
    claude_dir = target_dir / ".claude"
    claude_dir.mkdir(parents=True, exist_ok=True)
    link = claude_dir / "skills"
    rel_target = Path("..") / ".agents" / "skills"

    if link.is_symlink():
        return  # already linked
    if link.is_dir():
        console.warn(
            ".claude/skills already exists as a real directory — left in place.\n"
            "  Canonical skills now live in .agents/skills/. Remove the old\n"
            "  directory and re-run 'reins update' to switch to the symlink."
        )
        return
    try:
        link.symlink_to(rel_target, target_is_directory=True)
        console.success("Linked .claude/skills -> .agents/skills")
    except OSError:
        shutil.copytree(target_dir / SKILLS_ASSET, link, dirs_exist_ok=True)
        console.warn(
            "Symlinks unavailable (Windows without developer mode?) — copied\n"
            "  skills to .claude/skills instead. Keep both in sync via 'reins update'."
        )
```

In `_check_skill_collisions`, also probe `~/.agents/skills` the same way `~/.claude/skills` is probed (union both sets of user-level skill names before comparing).

- [ ] **Step 5: Update update.py and manifest.py**

`src/reins/commands/update.py`:
- Line 39 probe tuple: `("skills", ".agents/skills", ".claude/skills", "AGENTS.md")`
- Line 153 asset_probes entry: `(["skills", ".agents/skills", ".claude/skills"], ".agents/skills"),`
- Update the docstrings mentioning `.claude/skills` at lines 35 and 146-147.

`src/reins/manifest.py` `MANAGED_ASSETS`: change `".claude/skills"` to `".agents/skills"`.

- [ ] **Step 6: Run the affected suites**

Run: `uv run pytest tests/test_skill_copy.py tests/test_assets.py tests/test_update.py tests/test_manifest.py tests/test_init_platforms.py tests/test_source_of_truth.py tests/test_output_conventions.py -v`
Expected: pass. These structural suites may enumerate skill paths — fix any `.claude/skills` literals they assert on.

- [ ] **Step 7: Update doc references**

`grep -rn "claude/skills" AGENTS.md GETTING-STARTED.md platform-instructions/ .agents/skills/using-reins/SKILL.md` — change prose references to `.agents/skills/`. Platform templates say "reins skills in `.claude/skills/`" — change to `.agents/skills/`.

- [ ] **Step 8: Wheel smoke test**

```bash
uv build 2>/dev/null && unzip -l dist/*.whl | grep -c "_data/skills"
rm -rf dist
```
Expected: a non-zero count (skills shipped from the new path).

- [ ] **Step 9: Full suite + commit**

Run: `uv run pytest -q`
Expected: all pass.

```bash
git add -A && git commit -m "feat!: canonical .agents/skills install location with .claude/skills symlink"
```

### Task 8: Scaffold only enforced docs — trim plan template set

**Files:**
- Modify: `src/reins/kb_seed.py:75-76`
- Delete (kb): `kb/reins/exec-plans/_template/decisions.md`, `_template/progress.md`, `_template/retro.md`
- Delete (kb): `kb/reins/exec-plans/active/feature-beta-cleanup/{decisions,progress,retro}.md` (our own scaffolded stubs)
- Test: `tests/test_kb_seed.py`

- [ ] **Step 1: Update kb_seed test**

In `tests/test_kb_seed.py`, find assertions about `_template` contents; assert `plan.md` exists and `decisions.md`/`progress.md` do NOT:

```python
    template = tmp_path / "myrepo" / "exec-plans" / "_template"
    assert (template / "plan.md").is_file()
    assert not (template / "progress.md").exists()
    assert not (template / "decisions.md").exists()
```

Run: `uv run pytest tests/test_kb_seed.py -v` — expected: FAIL.

- [ ] **Step 2: Remove the seed writes**

In `src/reins/kb_seed.py` delete lines 75-76:

```python
    (template / "progress.md").write_text("# Progress\n")
    (template / "decisions.md").write_text("# Decisions\n")
```

Run: `uv run pytest tests/test_kb_seed.py -v` — expected: PASS.

- [ ] **Step 3: Remove the templates and our stubs from the kb**

```bash
rm kb/reins/exec-plans/_template/decisions.md kb/reins/exec-plans/_template/progress.md kb/reins/exec-plans/_template/retro.md
rm kb/reins/exec-plans/active/feature-beta-cleanup/decisions.md kb/reins/exec-plans/active/feature-beta-cleanup/progress.md kb/reins/exec-plans/active/feature-beta-cleanup/retro.md
```
(`_template/retro.md` goes away because `reins retro create` generates its own content — Task 9 — and plan create must not scaffold an empty retro.)

- [ ] **Step 4: Verify plan create scaffolds only plan.md**

```bash
git checkout -q -b tmp-scaffold-check && reins plan create >/dev/null && ls kb/reins/exec-plans/active/tmp-scaffold-check/
```
Expected output: `plan.md` only. Clean up:

```bash
rm -r kb/reins/exec-plans/active/tmp-scaffold-check
git checkout -q feature/beta-cleanup && git branch -q -D tmp-scaffold-check
```

- [ ] **Step 5: Publish kb + commit**

```bash
reins kb publish
git add kb src/reins/kb_seed.py tests/test_kb_seed.py
git commit -m "feat(plan): scaffold only plan.md — drop decisions/progress/retro stubs"
```

### Task 9: Retro rework — active-dir creation, registry sections, complete-time warning

**Files:**
- Modify: `src/reins/commands/doc_create.py:83-96` (`_create_retro`)
- Modify: `src/reins/commands/doc_show.py` (`cmd_retro_show`)
- Modify: `src/reins/commands/plan.py:179-181` (`cmd_plan_complete` retro warning)
- Test: `tests/commands/test_retro.py` (new)

- [ ] **Step 1: Write failing tests**

Create `tests/commands/test_retro.py` (use the `submodule_repo` fixture; `repo_slug()` is `"unknown"` for remoteless repos):

```python
"""Tests for retro create/show/complete-warning lifecycle."""

from __future__ import annotations

import subprocess
from pathlib import Path

from reins.commands.doc_create import cmd_retro_create
from reins.commands.plan import cmd_plan_complete

def _setup_branch_with_plan(repo: Path, branch: str) -> Path:
    subprocess.run(["git", "checkout", "-q", "-b", branch], cwd=repo, check=True)
    active = repo / "kb" / "unknown" / "exec-plans" / "active" / branch
    active.mkdir(parents=True)
    (active / "plan.md").write_text(
        f"# Execution Plan: {branch}\n\n**Status:** complete\n"
    )
    return active

def test_retro_create_targets_active_plan_dir(submodule_repo: Path, monkeypatch):
    active = _setup_branch_with_plan(submodule_repo, "feature-r")
    monkeypatch.chdir(submodule_repo)
    assert cmd_retro_create() == 0
    retro = active / "retro.md"
    assert retro.is_file()
    text = retro.read_text()
    for section in ("What Went Well", "What Could Be Improved", "Lessons Learned", "Action Items"):
        assert f"## {section}" in text

def test_retro_create_without_plan_uses_completed_dir(submodule_repo: Path, monkeypatch):
    subprocess.run(["git", "checkout", "-q", "-b", "feature-noplan"], cwd=submodule_repo, check=True)
    monkeypatch.chdir(submodule_repo)
    assert cmd_retro_create() == 0
    assert (
        submodule_repo / "kb" / "unknown" / "exec-plans" / "completed" / "feature-noplan" / "retro.md"
    ).is_file()

def test_plan_complete_warns_on_empty_retro(submodule_repo: Path, monkeypatch, capsys):
    _setup_branch_with_plan(submodule_repo, "feature-empty-retro")
    monkeypatch.chdir(submodule_repo)
    cmd_retro_create()
    capsys.readouterr()  # discard create output
    assert cmd_plan_complete() == 0
    out = capsys.readouterr().out
    assert "retro" in out.lower()
    assert "reins retro create" in out

def test_plan_complete_quiet_on_filled_retro(submodule_repo: Path, monkeypatch, capsys):
    active = _setup_branch_with_plan(submodule_repo, "feature-good-retro")
    monkeypatch.chdir(submodule_repo)
    cmd_retro_create()
    retro = active / "retro.md"
    retro.write_text(
        retro.read_text().replace(
            "## What Went Well\n\n- ", "## What Went Well\n\n- Shipped it cleanly"
        )
    )
    capsys.readouterr()
    assert cmd_plan_complete() == 0
    out = capsys.readouterr().out
    assert "No retro captured" not in out
    # retro traveled with the plan dir
    assert (
        submodule_repo / "kb" / "unknown" / "exec-plans" / "completed" / "feature-good-retro" / "retro.md"
    ).is_file()
```

Run: `uv run pytest tests/commands/test_retro.py -v` — expected: FAIL (retro goes to completed/, sections differ).

- [ ] **Step 2: Rewrite `_create_retro`**

In `src/reins/commands/doc_create.py` replace `_create_retro` (lines 83-96):

```python
def _create_retro(repo_dir: Path, title: str, author: str) -> Path:
    branch = current_branch() or "unknown"
    # Prefer the active plan dir (retro travels to completed/ at archive time);
    # fall back to the completed path when there is no active plan.
    active_dir = branch_doc_path("plan", repo_dir, branch).parent
    if active_dir.is_dir():
        target = active_dir / "retro.md"
    else:
        target = branch_doc_path("retro", repo_dir, branch)
    target.parent.mkdir(parents=True, exist_ok=True)
    heading = title.strip() if title.strip() else f"Retro: {branch}"
    sections = "".join(
        f"\n## {s}\n\n- \n" for s in REGISTRY["retro"].required_sections
    )
    target.write_text(_provenance(heading, author) + sections)
    return target
```

- [ ] **Step 3: Make `cmd_retro_show` probe active first**

In `src/reins/commands/doc_show.py` replace `cmd_retro_show`:

```python
def cmd_retro_show(branch: str | None = None, full: bool = False) -> int:
    repo_dir = _repo_dir()
    if repo_dir is None:
        return 1
    branch = branch or current_branch()
    if not branch:
        console.error("no branch given and none checked out")
        return 1
    active = branch_doc_path("plan", repo_dir, branch).parent / "retro.md"
    target = active if active.is_file() else branch_doc_path("retro", repo_dir, branch)
    if not target.is_file():
        console.error(f"no retro for branch '{branch}'")
        console.next_step(REGISTRY["retro"].create_hint)
        return 1
    _print_doc(target, "retro", branch_dir_name(branch), full)
    return 0
```
(match the module's existing imports — `branch_doc_path`, `branch_dir_name`, `REGISTRY`, `_repo_dir`, `_print_doc` are already available there.)

- [ ] **Step 4: Upgrade the plan-complete warning**

In `src/reins/commands/plan.py`, add after the imports:

```python
_EMPTY_RETRO_LINE = re.compile(r"^\s*-\s*(\[ \]\s*)?(_[^_]*_)?\s*$")

def _retro_is_empty(text: str) -> bool:
    """True when a retro has no filled-in bullet content."""
    for line in text.splitlines():
        s = line.strip()
        if not s or s.startswith("#") or s.startswith("**"):
            continue
        if _EMPTY_RETRO_LINE.match(line):
            continue
        return False
    return True
```

Replace lines 179-181 (the `retro = completed_dir / "retro.md"` tip block) with:

```python
    retro = completed_dir / "retro.md"
    if not retro.is_file() or _retro_is_empty(retro.read_text()):
        console.warn("No retro captured for this branch — lessons learned will be lost.")
        console.next_step("reins retro create")
```

- [ ] **Step 5: Run tests**

Run: `uv run pytest tests/commands/test_retro.py tests/test_doc_types.py tests/commands tests/test_output_conventions.py -v`
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add src/reins/commands/doc_create.py src/reins/commands/doc_show.py src/reins/commands/plan.py tests/commands/test_retro.py
git commit -m "feat(retro): create into active plan dir, registry sections, complete-time warning"
```

### Task 10: Retro step in finishing-a-development-branch skill

**Files:**
- Modify: `.agents/skills/finishing-a-development-branch/SKILL.md`

- [ ] **Step 1: Read the skill and add the retro step**

Read `.agents/skills/finishing-a-development-branch/SKILL.md`. Before its merge/PR decision section (after verification passes, before offering integration options), insert:

```markdown
## Capture the Retro (required)

Before completing the branch, capture lessons learned:

1. Run `reins retro create` (writes `retro.md` into the branch's active plan dir).
2. Fill in every section from THIS session's actual experience — What Went Well,
   What Could Be Improved, Lessons Learned, Action Items. Concrete beats generic;
   two honest bullets beat five vague ones.
3. If a lesson is a durable, enforceable rule, also add it via
   `reins principle add "<title>"`.
4. `reins kb publish`

`reins plan complete` warns if the retro is missing or empty — do not skip this.
```

If the skill has a checklist/flowchart, add the retro step there too so it can't be skipped.

- [ ] **Step 2: Commit**

```bash
git add .agents/skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(skills): require retro capture in finishing-a-development-branch"
```

### Task 11: Kb cleanup pass

**Files (all inside `kb/`, committed via `reins kb publish`):**
- Delete: `kb/agent-tasks/`, `kb/generated/agent-jobs/`
- Move: dead plans `active/` → `completed/`
- Modify: superseded/implemented specs (status header), `kb/reins/tech-debt/cleanup-queue.md`
- Modify: `src/reins/commands/doc_create.py:211-216` (skip list)

- [ ] **Step 1: Delete agent-task remnants**

```bash
rm -r kb/agent-tasks kb/generated/agent-jobs
```

- [ ] **Step 2: Drop the dead skip-list entry**

In `src/reins/commands/doc_create.py` (lines 211-216), remove `"agent-tasks",` (keep `"generated"` — the seed's `.gitignore` still creates that concept):

```python
    repo_name = parts[kb_idx + 1]
    # Skip shared dirs (., _, generated)
    if repo_name.startswith((".", "_")) or repo_name == "generated":
        return 0
```

Run: `uv run pytest tests/test_guardrail_hook.py tests/commands -q` — expected: pass (fix any test asserting on agent-tasks).

- [ ] **Step 3: Archive dead active plans**

`feature-cli-noun-first-restructure` already exists in `completed/` — its `active/` copy is a duplicate; delete it:

```bash
rm -r kb/reins/exec-plans/active/feature-cli-noun-first-restructure
```

`feature-axi-output-surface` is merged (PR #28) — archive as complete:

```bash
reins plan complete feature-axi-output-surface
```

The remaining nine are abandoned:

```bash
for b in claude-add-repo-metadata-upvjd feature-cross-platform-support feature-m5-publish-hardening \
         feature-mvp feature-mvp-2 feature-remove-touched-areas fix-security-critical-mvp \
         gh-reins-pr-crawl harness-to-kb-rename multi-repo-init-test; do
  d="kb/reins/exec-plans/active/$b"
  sed -i 's/\*\*Status:\*\* *\(planning\|in-progress\)/**Status:** abandoned/' "$d/plan.md"
  mv "$d" "kb/reins/exec-plans/completed/$b"
done
ls kb/reins/exec-plans/active/
```
Expected: only `feature-beta-cleanup` remains active.

- [ ] **Step 4: Delete untouched stub files from archived plan dirs**

A stub still contains the old template placeholders. Check each before deleting:

```bash
for f in kb/reins/exec-plans/completed/*/decisions.md; do
  grep -q '## \[Date\]' "$f" && rm "$f"
done
for f in kb/reins/exec-plans/completed/*/progress.md; do
  grep -q '\[One line: what' "$f" && rm "$f"
done
for f in kb/reins/exec-plans/completed/*/retro.md; do
  grep -qE '^- .*[A-Za-z]' "$f" || rm "$f"
done
```
Then eyeball what's left: `ls kb/reins/exec-plans/completed/*/` — real content (e.g. `feature-axi-output-surface/decisions.md`, `python-cli-rewrite/*`) must survive. If a deletion looks wrong, restore via `reins kb git checkout -- <path>`.

- [ ] **Step 5: Mark superseded/implemented specs**

For each spec below, edit the `**Status:**` line (if one lacks a Status line, add it after `**Author:**`).

Status `superseded` (feature cut or direction abandoned): `agent-task-dispatch.md`, `agent-runner-layer.md`, `mvp-roadmap.md`, `m2-init-gh-integration.md`, `m4-skills-unification.md`, `m5-publish-sync-hardening.md`, `m6-polish-trust-signals.md`, `team-presentation-reins-and-agentic-dev.md`, `team-presentation-implementation-plan.md`, `comparison-akmnitt-approach.md`, `block-dangerous-commands-go-rewrite.md`, `pr-history-crawling.md`, `mvp-unified-attach.md`, `multi-editor-hook-support.md` (verify: if it describes the CURRENT editor-hook design, mark `implemented` instead).

Status `implemented` (shipped, kept for history): `harness-to-kb-rename.md`, `harness-to-kb-rename-handoff.md`, `centralized-doc-type-registry.md`, `cli-noun-first-restructure.md`, `agents-md-template-and-population.md`, `skills-fork-and-template-docs.md`, `submodule-management.md`, `submodule-pointer-staleness-fix.md`, `2026-02-22-submodule-pointer-staleness-fix-plan.md`, `remove-touched-areas-compute-overlap-on-demand.md`, `reins-update-command.md`, `update-superpowers-skill.md`, `reins-init-rework.md`, `init-rework-findings.md`, `multi-repo-init-findings.md`, `agent-native-output-surface-axi-principles.md`.

Leave untouched (living docs): `core-beliefs.md`, `reins-design.md`, `reins-implementation.md`, `smoke-test-design.md`, `harness-auto-load-and-skill-triggers.md`, `reclaim-spec-noun-for-sdd-sense-design-to-spec-spec-to-prd.md`, `harness-engineering-quick-wins-principle-provenance-surprise.md`, `index.md`, and the beta-cleanup spec itself.

Before marking each, skim its first 20 lines; if the classification above is wrong for one, use your judgment and note it in the PR description.

- [ ] **Step 6: Prune cleanup-queue.md**

In `kb/reins/tech-debt/cleanup-queue.md` (the queue's own convention says completed rows are removed during cleanup passes — so delete resolved rows):
- Delete rows SEC-01..05 (Status completed).
- Delete SEC-07 (setup-hooks.sh — file deleted), SEC-09 (job.json — agent dispatch deleted), TEST-05 (commands/agent.py — deleted).
- Delete TEST-03 (post_checkout tests — resolved by Task 3) and TEST-04 (pre_push — verify: tests/commands/internal/test_pre_push.py already exists, so this row was stale; delete).
- Keep: SEC-06, SEC-08, ERR-01..03, OUT-01, TEST-01, TEST-02.

- [ ] **Step 7: Publish and bump pointer**

```bash
reins kb lint
reins kb publish
git add kb src/reins/commands/doc_create.py
git commit -m "chore(kb): archive dead plans, mark superseded specs, prune queue, drop agent-task remnants"
```
If `reins kb lint` flags the moved/edited docs, fix per its remediation messages before publishing.

### Task 12: Docs — README rewrite, HUMANS.md, GETTING-STARTED

**Files:**
- Rewrite: `README.md`
- Rename: `HELLO.md` → `HUMANS.md`
- Verify: `GETTING-STARTED.md`, `pyproject.toml`

- [ ] **Step 1: Rename HELLO.md**

```bash
git mv HELLO.md HUMANS.md
```
(Michael rewrites the content himself later — carry the text as-is.)

- [ ] **Step 2: Rewrite README.md**

Replace the full README with this structure (write real prose, not placeholders; source facts from GETTING-STARTED.md, `reins help` output, and the skills dir listing):

1. **Title + pitch** (3-4 sentences): reins is a Skill Set + CLI for managing an AI knowledgebase and enforcing a spec-driven development workflow across a team. Humans steer, agents execute; the kb (a git submodule) encodes intent, constraints, and context; hooks and the CLI keep it honest. Link to HUMANS.md ("a hand-written intro from the author").
2. **Status**: beta candidate; API surface still settling.
3. **Quick start**: `pip install reins` / `uv tool install reins`, `cd your-repo`, `reins init` — then the daily loop: `reins kb sync` → work with skills (brainstorming → spec → plan → execute → retro) → `reins kb publish`.
4. **The workflow** (core section): spec-driven development as enforced — protected kb paths force docs through `reins <type> create`; plan per branch; retro at finish; golden principles; overlap detection across branches. Doc-types table (spec, prd, plan, retro, debt, idea, principle) with create commands.
5. **The CLI**: command table (reuse the table from `.agents/skills/using-reins/SKILL.md`, kept in sync).
6. **The skill set**: one line per skill group — workflow skills (using-reins, brainstorming, writing-plans, executing-plans, subagent-driven-development, finishing-a-development-branch), discipline skills (test-driven-development, systematic-debugging, verification-before-completion, requesting/receiving-code-review), maintenance (update-superpowers, writing-skills, using-git-worktrees, dispatching-parallel-agents, populate-agents-md). Note the superpowers fork attribution (ATTRIBUTION.md).
7. **Platforms**: Claude Code, Cursor, GitHub Copilot, Codex — AGENTS.md is the universal entry point; skills live in `.agents/skills/` (open standard) with a `.claude/skills` symlink for Claude Code; editor hooks installed for all three hook-capable editors.
8. **Repository structure**: short tree of what actually exists post-cleanup (src/, .agents/skills/, hooks/, editor-hooks/, linters/, platform-instructions/, upgrades/, kb/, tests/).
9. **Kb as submodule**: why (shared context across branches/contributors), the sync/publish model, `reins mode disable|incognito` escape hatches.
10. **Contributing + References**: keep the Harness Engineering inspiration link and license.

Delete entirely: extension system section, scripts section, MCP servers section, adoption tiers, workflows/doc-gardening CI references, "7 skills" table.

- [ ] **Step 3: Sweep GETTING-STARTED.md and remaining references**

```bash
grep -n "scripts/\|extensions\|gemini\|HELLO\|claude/skills\|agent-tasks" GETTING-STARTED.md README.md AGENTS.md pyproject.toml
```
Fix any hits (GETTING-STARTED should need at most the skills-path and platform-list touch-ups). Confirm pyproject force-include lists only shipped dirs (AGENTS.md, .agents/skills, hooks, editor-hooks, linters, upgrades, platform-instructions).

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "docs: rewrite README around beta scope; HELLO.md -> HUMANS.md"
```

### Task 13: Final verification

- [ ] **Step 1: Full quality gates**

```bash
uv run pytest -q
uv run ruff check
uv run pyright
reins kb lint
```
Expected: all clean. Fix anything that isn't.

- [ ] **Step 2: End-to-end smoke test in a throwaway repo**

```bash
mkdir -p /tmp/claude-1000/-workspace-reins/e8ca1b08-90a1-4729-b3a5-0ff7e520f7c1/scratchpad/smoke
cd /tmp/claude-1000/-workspace-reins/e8ca1b08-90a1-4729-b3a5-0ff7e520f7c1/scratchpad/smoke
git init -q smoketest && cd smoketest
uv run --project ~/dev/reins reins init
```
Walk the interactive prompts (local kb, default platforms + codex). Verify:
- `.agents/skills/` populated; `.claude/skills` is a symlink into it
- `AGENTS.md` present; `CLAUDE.md` present; no `GEMINI.md`
- `.claude/settings.json` has both PreToolUse matchers (`Write|Edit`, `Bash`)
- `.git/hooks/post-checkout` is the thin stub
- `git checkout -b feature-x` prints the `reins plan create` suggestion
- `reins plan create` scaffolds only `plan.md`; `reins retro create` writes `retro.md` into the active plan dir
Then `cd ~/dev/reins`.

- [ ] **Step 3: Update the using-reins skill CLI table if drifted**

`reins help` vs the table in `.agents/skills/using-reins/SKILL.md` — they must match (no gemini, no removed commands).

- [ ] **Step 4: Commit any final fixes, publish plan progress**

```bash
git add -A && git commit -m "chore: final beta-cleanup verification fixes" || true
reins kb publish
```

## Dependencies

- Task 7 (skills move) before Task 10 and 12 (they edit files under `.agents/skills/`).
- Task 8 before Task 9 (template retro removed so retro tests aren't polluted by scaffolded stubs).
- Within Task 11, Step 1 (delete kb/agent-tasks) before Step 2 (drop its skip-list entry).
- Everything else is order-independent but the listed order minimizes rebase pain.
