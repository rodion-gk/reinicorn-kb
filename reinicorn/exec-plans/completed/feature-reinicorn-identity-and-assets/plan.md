---
type: plan
title: Reinicorn Identity and Asset Ownership Implementation Plan
slug: feature-reinicorn-identity-and-assets
lifecycle: done
status: complete
created: 2026-07-17
author: Michael Biehl
origin: ai-assisted
branch: feature-reinicorn-identity-and-assets
spec: kb/reinicorn/specs/reinicorn-public-release-program-and-identity-migration.md
---

# Reinicorn Identity and Asset Ownership Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete public-release Gate 1 by replacing the Reins runtime identity with Reinicorn, exposing only the `rcorn` CLI, separating source and downstream AGENTS ownership, and moving KB navigation into a scope README.

**Architecture:** Rename the Python namespace first, then centralize runtime identity in `reinicorn.identity`. Keep project instructions user-owned by rendering a dedicated template once, while package-managed skills, hooks, linters, workflows, and platform instructions remain manifest-managed. Persist an explicit KB scope override so the private archive remote can remain named `reins` while the working KB scope becomes `reinicorn`.

**Tech Stack:** Python 3.12+, stdlib `argparse`, hatchling/hatch-vcs, uv, pytest, Ruff, Pyright, Git submodules, shell hooks, GitHub Actions.

## Goal

Deliver Gate 1 as a clean identity and ownership cutover while keeping every
checkpoint independently testable and reviewable.

## Global Constraints

- Public brand, repository, distribution, and Python package are `reinicorn`; the sole executable is `rcorn`.
- State lives under `.reinicorn/`; repository configuration is `.reinicorn-config`; configuration keys and secrets use `REINICORN_*`.
- Provide no `reins` executable, import alias, configuration fallback, environment fallback, deprecated skill alias, or hidden compatibility path.
- Root `AGENTS.md` is source/user-owned and must never be recorded in `.reinicorn/manifest.json` or considered by `rcorn update`.
- The generic downstream template is `templates/AGENTS.md`, packaged as `reinicorn/_data/templates/AGENTS.md` and rendered only when the target has no `AGENTS.md`.
- `kb/<scope>/README.md` is the team-owned KB map; initialization creates it only when absent and never updates it.
- Do not rewrite private historical KB prose merely to remove the old name; Gate 4 decides which historical records enter the fresh public export.
- Do not publish to PyPI in this plan.
- Every behavior change follows red-green TDD and every task ends with its focused tests and a conventional commit.

## Acceptance Criteria

- [ ] `import reinicorn`, `python -m reinicorn`, and `rcorn --version` work.
- [ ] Package metadata exposes exactly `rcorn = "reinicorn.cli:main"` and no `reins` script or namespace.
- [ ] Runtime state, config, mode, hooks, workflow secrets, and messages use only the final identity.
- [ ] `rcorn init` renders a generic user-owned `AGENTS.md` and creates `kb/<scope>/README.md` without managing either file afterward.
- [ ] Re-running init and running update preserve existing AGENTS and KB README files byte-for-byte.
- [ ] The Reinicorn source repository has a populated source-specific `AGENTS.md` that points to `kb/reinicorn/README.md`.
- [ ] The active KB scope is `kb/reinicorn/`, with an explicit scope override while the private parent remote retains its archived name.
- [ ] Runtime, packaging inputs, current release-facing docs, and built archives pass the legacy-identity scan.
- [ ] Full pytest, Ruff, Pyright, structural shell tests, package build, and extracted-wheel smoke tests pass.

## Dependencies and cutover rule

- PR #9 approved the governing spec.
- The physical KB scope move in Task 7 is shared across every branch. Before it runs, `rcorn kb status` must show no uncoordinated active work, or the owner must explicitly approve the coordinated cutover. Do not copy the scope or leave both `reins/` and `reinicorn/` live.
- The private `.gitmodules` URL remains private until Gate 4 creates `reinicorn-kb`; it is excluded from the Gate 1 public-artifact scan.

---

## Tasks

Execute the following nine tasks in order. Task 7 is the coordinated shared-KB
cutover and must not run until its explicit precondition is satisfied.

### Task 1: Rename the Python namespace and executable

**Files:**
- Rename: `src/reins/` -> `src/reinicorn/` (all Python modules and package data paths)
- Create: `src/reinicorn/identity.py`
- Create: `tests/test_identity.py`
- Modify: `pyproject.toml`
- Modify: `tests/test_cli_shape.py`
- Modify: every Python test returned by `rg -l 'from reins|import reins|reins\.' tests --glob '*.py'`

**Interfaces:**
- Produces: `reinicorn.identity.PRODUCT_NAME`, `CLI_NAME`, `STATE_DIR_NAME`, `CONFIG_FILE_NAME`, `ENV_PREFIX`, all `str` constants.
- Produces: distribution entry point `rcorn = "reinicorn.cli:main"`.
- Consumes: existing CLI behavior and command dispatch without semantic changes.

- [ ] **Step 1: Write failing identity and entry-point tests**

Create `tests/test_identity.py`:

```python
from __future__ import annotations

import tomllib
from pathlib import Path

from reinicorn.identity import (
    CLI_NAME,
    CONFIG_FILE_NAME,
    ENV_PREFIX,
    PRODUCT_NAME,
    STATE_DIR_NAME,
)

def test_public_identity_contract() -> None:
    assert PRODUCT_NAME == "Reinicorn"
    assert CLI_NAME == "rcorn"
    assert STATE_DIR_NAME == ".reinicorn"
    assert CONFIG_FILE_NAME == ".reinicorn-config"
    assert ENV_PREFIX == "REINICORN_"

def test_pyproject_exposes_only_rcorn() -> None:
    data = tomllib.loads(Path("pyproject.toml").read_text())
    assert data["project"]["name"] == "reinicorn"
    assert data["project"]["scripts"] == {"rcorn": "reinicorn.cli:main"}
```

Update `tests/test_cli_shape.py` so its subprocess helper invokes
`[sys.executable, "-m", "reinicorn", *args]`, is named `_rcorn`, and asserts
`rcorn --version` begins with `rcorn `.

- [ ] **Step 2: Run the tests to verify the old namespace fails**

Run:

```bash
uv run pytest tests/test_identity.py tests/test_cli_shape.py -v
```

Expected: collection fails with `ModuleNotFoundError: No module named 'reinicorn'`.

- [ ] **Step 3: Move the package and update imports mechanically**

Run:

```bash
git mv src/reins src/reinicorn
rg -l -0 'from reins|import reins|reins\.' src/reinicorn tests --glob '*.py' | xargs -0 sed -i 's/from reins/from reinicorn/g; s/import reins/import reinicorn/g; s/reins\./reinicorn\./g'
```

Create `src/reinicorn/identity.py`:

```python
"""Canonical runtime identity for Reinicorn."""

PRODUCT_NAME = "Reinicorn"
CLI_NAME = "rcorn"
STATE_DIR_NAME = ".reinicorn"
CONFIG_FILE_NAME = ".reinicorn-config"
ENV_PREFIX = "REINICORN_"
```

Update `src/reinicorn/__init__.py`, `src/reinicorn/__main__.py`, and
`src/reinicorn/cli.py` to import from `reinicorn`, set `prog=CLI_NAME`, display
`Reinicorn`, and render the version as `f"{CLI_NAME} {__version__}"`.

Update `pyproject.toml` exactly:

```toml
[project]
name = "reinicorn"

[project.urls]
Repository = "https://github.com/mnbiehl/reinicorn"

[project.scripts]
rcorn = "reinicorn.cli:main"

[tool.hatch.version]
source = "vcs"

[tool.hatch.build.hooks.vcs]
version-file = "src/reinicorn/_version.py"

[tool.hatch.build.targets.wheel]
packages = ["src/reinicorn"]
```

Change all force-include destinations and Ruff/Pyright paths from `reins` to
`reinicorn`. Rename `reins_root()` to `reinicorn_root()` and
`reins_source_repo()` to `reinicorn_source_repo()` in their definitions, imports,
patch targets, and tests; do not retain wrappers.

- [ ] **Step 4: Run namespace and CLI tests**

Run:

```bash
uv run pytest tests/test_identity.py tests/test_cli_shape.py tests/test_meta.py tests/test_git.py -v
uv run pyright src/reinicorn
```

Expected: all selected tests pass and Pyright reports zero errors.

- [ ] **Step 5: Run the full suite to expose missed import paths**

Run:

```bash
uv run pytest tests/ -q
uv run ruff check src/reinicorn tests
```

Expected: all tests pass and Ruff reports no violations.

- [ ] **Step 6: Commit the namespace cutover**

```bash
git add pyproject.toml src/reinicorn tests
git commit -m "refactor(identity): rename Python package and CLI"
```

---

### Task 2: Centralize state, configuration, manifest, and KB scope

**Files:**
- Modify: `src/reinicorn/identity.py`
- Modify: `src/reinicorn/config.py`
- Modify: `src/reinicorn/manifest.py`
- Modify: `src/reinicorn/mode.py`
- Modify: `src/reinicorn/commands/init.py`
- Modify: `src/reinicorn/commands/status.py`
- Modify: `src/reinicorn/commands/plan.py`
- Modify: `src/reinicorn/commands/home.py`
- Modify: `src/reinicorn/commands/review.py`
- Modify: `src/reinicorn/commands/idea.py`
- Modify: `src/reinicorn/commands/doc_create.py`
- Modify: `src/reinicorn/commands/doc_show.py`
- Modify: `src/reinicorn/kb.py`
- Modify: `tests/test_config.py`
- Modify: `tests/test_manifest.py`
- Modify: `tests/test_mode.py`
- Modify: `tests/test_init_manifest.py`
- Modify: `tests/test_init_slug_override.py`
- Modify: command tests patching `repo_slug` for KB paths

**Interfaces:**
- Produces: `reinicorn.config.kb_scope(root: Path | None = None) -> str`.
- Produces: `.reinicorn/manifest.json` with `reinicorn_version` and package-owned files only.
- Produces: `.reinicorn/mode` for enabled/disabled/incognito state.
- Consumes: `reinicorn.git.repo_slug()` only as the fallback when no configured KB scope exists.

- [ ] **Step 1: Write failing final-path tests**

Update `tests/test_manifest.py` to assert:

```python
manifest_path = repo / ".reinicorn" / "manifest.json"
data = json.loads(manifest_path.read_text())
assert data["reinicorn_version"] == "0.1.0"
assert "AGENTS.md" not in data["files"]
```

Update `tests/test_config.py` to create `.reinicorn-config`, and add:

```python
def test_kb_scope_prefers_configured_scope(tmp_path: Path) -> None:
    (tmp_path / ".reinicorn-config").write_text(
        "REINICORN_KB_SCOPE=reinicorn\n"
    )
    assert kb_scope(tmp_path) == "reinicorn"
```

Update `tests/test_mode.py` to assert `set_mode("incognito", tmp_path)` writes
`.reinicorn/mode`. Extend `test_init_slug_override` to assert the explicit slug is
persisted as `REINICORN_KB_SCOPE=custom-name`.

- [ ] **Step 2: Run the focused tests to verify old paths fail**

```bash
uv run pytest tests/test_config.py tests/test_manifest.py tests/test_mode.py tests/test_init_manifest.py tests/test_init_slug_override.py -v
```

Expected: failures reference `.reins-config`, `.reins/manifest.json`,
`reins_version`, `.reins-mode`, and missing `kb_scope`.

- [ ] **Step 3: Implement the final state and scope contract**

Extend `src/reinicorn/identity.py`:

```python
MANIFEST_FILE_NAME = "manifest.json"
MODE_FILE_NAME = "mode"
KB_SCOPE_KEY = "REINICORN_KB_SCOPE"
TICKET_PATTERN_KEY = "REINICORN_TICKET_PATTERN"
STALE_THRESHOLD_KEY = "REINICORN_STALE_THRESHOLD"
```

In `src/reinicorn/config.py`, read only `CONFIG_FILE_NAME` and add:

```python
def kb_scope(root: Path | None = None) -> str:
    """Return the configured KB scope, falling back to the origin-derived slug."""
    configured = config_get(KB_SCOPE_KEY, root=root)
    if configured:
        return configured
    from reinicorn.git import repo_slug
    return repo_slug()

def config_set(key: str, value: str, root: Path) -> None:
    """Set one KEY=value entry while preserving unrelated config lines."""
    path = root / CONFIG_FILE_NAME
    lines = path.read_text().splitlines() if path.is_file() else []
    replacement = f"{key}={value}"
    output: list[str] = []
    replaced = False
    for line in lines:
        if line.partition("=")[0].strip() == key:
            output.append(replacement)
            replaced = True
        else:
            output.append(line)
    if not replaced:
        output.append(replacement)
    path.write_text("\n".join(output) + "\n")
```

Make `cmd_init(..., slug=...)` persist the validated explicit scope with
`config_set(KB_SCOPE_KEY, slug, cwd)`. Replace KB-path uses of `repo_slug()` with
`kb_scope()` in the files listed above; keep origin parsing in `git.repo_slug()`.

In `manifest.py`, build paths from `STATE_DIR_NAME`, require
`{"reinicorn_version", "files"}`, write `reinicorn_version`, and remove
`AGENTS.md` from `MANAGED_ASSETS`. In `mode.py`, read and write
`root / STATE_DIR_NAME / MODE_FILE_NAME`, creating the state directory before write.

- [ ] **Step 4: Run focused state and scope tests**

```bash
uv run pytest tests/test_config.py tests/test_manifest.py tests/test_mode.py tests/test_init_manifest.py tests/test_init_slug_override.py tests/commands/test_status.py tests/commands/test_plan.py -v
```

Expected: all selected tests pass.

- [ ] **Step 5: Prove old runtime paths are absent**

```bash
rg -n '\.reins|REINS_|reins_version' src/reinicorn tests/test_config.py tests/test_manifest.py tests/test_mode.py tests/test_init_manifest.py
```

Expected: no matches.

- [ ] **Step 6: Commit runtime identity**

```bash
git add src/reinicorn tests
git commit -m "refactor(config): adopt Reinicorn state and scope identity"
```

---

### Task 3: Rename installed skills, hooks, workflows, and platform instructions

**Files:**
- Rename: `.agents/skills/using-reins/` -> `.agents/skills/using-reinicorn/`
- Rename: `workflows/reins-doc-review-cleanup.yml` -> `workflows/reinicorn-doc-review-cleanup.yml`
- Rename: `.github/hooks/reins.json` -> `.github/hooks/reinicorn.json`
- Modify: `.agents/skills/ATTRIBUTION.md`
- Modify: `.agents/skills/brainstorming/SKILL.md`
- Modify: `.agents/skills/executing-plans/SKILL.md`
- Modify: `.agents/skills/requesting-code-review/SKILL.md`
- Modify: `.agents/skills/update-superpowers/SKILL.md`
- Modify: `.agents/skills/update-superpowers/replacements.yaml`
- Modify: `.agents/skills/writing-plans/SKILL.md`
- Modify: `hooks/post-checkout`
- Modify: `hooks/post-merge`
- Modify: `hooks/pre-push`
- Modify: `editor-hooks/block-raw-kb-git.sh`
- Modify: `editor-hooks/enforce-doc-templates.sh`
- Modify: `platform-instructions/claude.md`
- Modify: `platform-instructions/copilot.md`
- Modify: `platform-instructions/cursor.md`
- Modify: `linters/README.md`
- Modify: `linters/framework/run-lints.sh`
- Modify: `linters/rules/kb/provenance.sh`
- Modify: `linters/rules/scripts/shellcheck.sh`
- Modify: `upgrades/README.md`
- Modify: `src/reinicorn/commands/hooks_install.py`
- Modify: `src/reinicorn/commands/review.py`
- Modify: `tests/test_skill_copy.py`
- Modify: `tests/test_init_hook.py`
- Modify: `tests/commands/test_hooks_install.py`
- Modify: `tests/test_assets.py`
- Modify: `tests/commands/test_review.py`

**Interfaces:**
- Produces: installed skill name `using-reinicorn` and hook directory `.reinicorn/hooks`.
- Produces: Copilot hook config `.github/hooks/reinicorn.json`.
- Produces: workflow asset `workflows/reinicorn-doc-review-cleanup.yml` using `REINICORN_INSTALL_TOKEN` and `__REINICORN_REPO__`.

- [ ] **Step 1: Change tests to require only final asset names**

In `tests/test_skill_copy.py`, require:

```python
assert (target / ".agents/skills/using-reinicorn/SKILL.md").is_file()
assert not (target / ".agents/skills/using-reins").exists()
```

In hook tests, require `.reinicorn/hooks/...` and
`.github/hooks/reinicorn.json`. In `tests/test_assets.py`, resolve
`workflows/reinicorn-doc-review-cleanup.yml` and assert:

```python
assert "REINICORN_INSTALL_TOKEN" in text
assert "__REINICORN_REPO__" in text
assert "rcorn _review-cleanup" in text
assert "reins" not in "\n".join(
    line for line in text.splitlines() if not line.lstrip().startswith("#")
).lower()
```

- [ ] **Step 2: Run asset tests to verify they fail on old names**

```bash
uv run pytest tests/test_skill_copy.py tests/test_init_hook.py tests/commands/test_hooks_install.py tests/test_assets.py tests/commands/test_review.py -v
```

Expected: failures show missing `using-reinicorn`, `.reinicorn/hooks`, and renamed workflow/config files.

- [ ] **Step 3: Rename and rewrite package-owned assets**

```bash
git mv .agents/skills/using-reins .agents/skills/using-reinicorn
git mv workflows/reins-doc-review-cleanup.yml workflows/reinicorn-doc-review-cleanup.yml
git mv .github/hooks/reins.json .github/hooks/reinicorn.json
```

Replace user-facing commands with `rcorn`, product prose with `Reinicorn`, state
paths with `.reinicorn`, the skill trigger with `using-reinicorn`, the workflow
placeholder with `__REINICORN_REPO__`, and its secret with
`REINICORN_INSTALL_TOKEN` across the exact files listed in this task.

In `hooks_install.py`, set:

```python
MARKER = "# --- reinicorn hooks below ---"
_SCRIPT_DEST = ".reinicorn/hooks"
_COPILOT_CONFIG = ".github/hooks/reinicorn.json"
```

Update `pyproject.toml` force-includes only where renamed asset paths require it.
Do not create old-name symlinks or wrapper skills.

- [ ] **Step 4: Run focused asset tests and shell checks**

```bash
uv run pytest tests/test_skill_copy.py tests/test_init_hook.py tests/commands/test_hooks_install.py tests/test_assets.py tests/commands/test_review.py -v
bash tests/run-all.sh
```

Expected: selected pytest tests pass; shell/structural runner exits 0.

- [ ] **Step 5: Scan package-owned assets for legacy identity**

```bash
rg -n -i '\breins\b|\.reins|REINS_' .agents hooks editor-hooks workflows platform-instructions linters upgrades src/reinicorn/commands/hooks_install.py
```

Expected: no matches.

- [ ] **Step 6: Commit installed-asset identity**

```bash
git add .agents .github/hooks hooks editor-hooks workflows platform-instructions linters upgrades pyproject.toml src/reinicorn tests
git commit -m "refactor(assets): rename installed Reinicorn integrations"
```

---

### Task 4: Split the downstream AGENTS template from the source file

**Files:**
- Create: `templates/AGENTS.md`
- Modify: `pyproject.toml`
- Modify: `src/reinicorn/commands/init.py`
- Modify: `src/reinicorn/commands/update.py`
- Modify: `src/reinicorn/manifest.py`
- Modify: `tests/test_assets.py`
- Modify: `tests/commands/test_init.py`
- Modify: `tests/test_init_slug_override.py`
- Modify: `tests/test_manifest.py`
- Modify: `tests/test_update.py`

**Interfaces:**
- Produces: template asset lookup name `templates/AGENTS.md`.
- Produces: rendered destination `target/AGENTS.md`, created only when absent.
- Consumes: `get_asset_path(name: str) -> Path | None` without changing its contract.

- [ ] **Step 1: Write failing ownership tests**

Add to `tests/commands/test_init.py`, using its existing `existing_repo` and
`seeded_bare` fixtures:

```python
def test_init_renders_generic_agents_once_and_preserves_it(
    existing_repo: Path, seeded_bare: Path, tmp_path: Path
) -> None:
    template = tmp_path / "templates" / "AGENTS.md"
    template.parent.mkdir()
    template.write_text("# {repo}\n\nRead `kb/{repo}/README.md`.\n<!-- UNPOPULATED -->\n")
    with patch("reinicorn.commands.init.get_asset_path", return_value=template), \
         patch("reinicorn.commands.init.cmd_hooks_install", return_value=0), \
         patch("reinicorn.commands.init._prompt_platforms", return_value=[]):
        assert cmd_init(
            kb_url=str(seeded_bare), cwd=existing_repo, slug="sample"
        ) == 0
    agents = existing_repo / "AGENTS.md"
    assert "kb/sample/README.md" in agents.read_text()
    assert "<!-- UNPOPULATED" in agents.read_text()
    agents.write_text("# User owned\n")
    with patch("reinicorn.commands.init.cmd_hooks_install", return_value=0):
        assert cmd_init(cwd=existing_repo) == 0
    assert agents.read_text() == "# User owned\n"
```

Update manifest/update tests to assert `AGENTS.md` is absent from manifest files,
absent from `_collect_package_files()`, and remains byte-for-byte unchanged after
`cmd_update()`.

- [ ] **Step 2: Run ownership tests to verify current management fails**

```bash
uv run pytest tests/commands/test_init.py tests/test_init_slug_override.py tests/test_manifest.py tests/test_update.py tests/test_assets.py -v
```

Expected: failures show root `AGENTS.md` is still used as the asset and is still manifest/update-managed.

- [ ] **Step 3: Create the generic template and rewire init**

Create `templates/AGENTS.md` with this stable structure:

```markdown
<!-- UNPOPULATED: Run the populate-agents-md skill to complete this file. -->
# {repo}

<!-- Describe what this project does and who it serves. -->

## Build and test

<!-- Record the exact build, lint, type-check, and test commands. -->

## Project conventions

<!-- Record naming, architecture, and review conventions. -->

## Knowledge base

Read and follow `kb/{repo}/README.md` before planning or changing code. It is the
canonical map for principles, architecture, specs, plans, quality, and debt.

Use `rcorn` for every KB operation. Never manage the KB submodule with raw Git.
```

Set `AGENTS_ASSET = "templates/AGENTS.md"` in `init.py`, but keep the destination
name `AGENTS.md`. Package it with:

```toml
"templates/AGENTS.md" = "reinicorn/_data/templates/AGENTS.md"
```

Remove every AGENTS probe and collection branch from `commands/update.py`; remove
`AGENTS.md` from `manifest.MANAGED_ASSETS`. Missing template errors must say:

```text
Missing packaged template 'templates/AGENTS.md'. Reinstall Reinicorn, then rerun 'rcorn init'.
```

- [ ] **Step 4: Run ownership and asset tests**

```bash
uv run pytest tests/commands/test_init.py tests/test_init_slug_override.py tests/test_manifest.py tests/test_update.py tests/test_assets.py -v
```

Expected: all selected tests pass.

- [ ] **Step 5: Commit the ownership split**

```bash
git add templates/AGENTS.md pyproject.toml src/reinicorn tests
git commit -m "refactor(init): make downstream AGENTS user-owned"
```

---

### Task 5: Seed a GitHub-rendered KB README without taking ownership

**Files:**
- Modify: `src/reinicorn/kb_seed.py`
- Modify: `tests/test_kb_seed.py`
- Modify: `tests/test_multi_repo_init.py`
- Modify: `tests/test_init_slug_override.py`

**Interfaces:**
- Produces: `kb/<scope>/README.md` on first scope creation.
- Preserves: any pre-existing scope README byte-for-byte.

- [ ] **Step 1: Write failing seed and preservation tests**

Add to `tests/test_kb_seed.py`:

```python
def test_seed_creates_scope_readme_map(tmp_path: Path) -> None:
    generate_seed_tree(tmp_path, "my-project")
    text = (tmp_path / "my-project" / "README.md").read_text()
    for target in (
        "golden-principles.md",
        "architecture/",
        "specs/",
        "prds/",
        "exec-plans/active/",
        "quality-scores.md",
        "tech-debt/",
    ):
        assert target in text
    assert "rcorn kb sync" in text
    assert "rcorn kb publish" in text

def test_seed_preserves_existing_scope_readme(tmp_path: Path) -> None:
    scope = tmp_path / "my-project"
    scope.mkdir()
    readme = scope / "README.md"
    readme.write_text("# Team map\n")
    generate_seed_tree(tmp_path, "my-project")
    assert readme.read_text() == "# Team map\n"
```

- [ ] **Step 2: Run seed tests to verify README behavior is absent**

```bash
uv run pytest tests/test_kb_seed.py tests/test_multi_repo_init.py tests/test_init_slug_override.py -v
```

Expected: README creation test fails because no scope README exists.

- [ ] **Step 3: Implement create-once README generation**

In `kb_seed.py`, write only when absent:

```python
readme = scope / "README.md"
if not readme.exists():
    readme.write_text(
        f"# {repo_slug} knowledge base\n\n"
        "This file is the canonical map for humans and agents.\n\n"
        "| Topic | Location |\n|---|---|\n"
        "| Golden principles | `golden-principles.md` |\n"
        "| Architecture | `architecture/` |\n"
        "| Approved specs | `specs/` |\n"
        "| Product requirements | `prds/` |\n"
        "| Active plans | `exec-plans/active/` |\n"
        "| Quality scores | `quality-scores.md` |\n"
        "| Technical debt | `tech-debt/` |\n\n"
        "Use `rcorn kb sync` before work and `rcorn kb publish` after KB changes.\n"
        "Create protected documents only through their `rcorn <type> create` command.\n"
    )
```

Do not add README to any package manifest or update collector.

- [ ] **Step 4: Run seed and init tests**

```bash
uv run pytest tests/test_kb_seed.py tests/test_multi_repo_init.py tests/test_init_slug_override.py -v
```

Expected: all selected tests pass.

- [ ] **Step 5: Commit KB map generation**

```bash
git add src/reinicorn/kb_seed.py tests/test_kb_seed.py tests/test_multi_repo_init.py tests/test_init_slug_override.py
git commit -m "feat(kb): seed a scope README map"
```

---

### Task 6: Populate the Reinicorn source instructions and final local config

**Files:**
- Rename: `.reins-config` -> `.reinicorn-config`
- Modify: `.reinicorn-config`
- Modify: `AGENTS.md`
- Modify: `.gitignore`
- Modify: `.github/workflows/test.yml`
- Modify: `.github/workflows/lint-kb.yml`
- Modify: `.github/workflows/lint-architecture.yml`
- Test: `tests/test_public_identity.py`

**Interfaces:**
- Produces: source-owned `AGENTS.md` pointing to `kb/reinicorn/README.md`.
- Produces: `REINICORN_KB_SCOPE=reinicorn` so KB commands work while the private origin remains archived as `reins`.

- [ ] **Step 1: Add a failing source-identity structural test**

Create `tests/test_public_identity.py`:

```python
from __future__ import annotations

from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent

def test_source_agents_is_populated_and_points_to_reinicorn_scope() -> None:
    text = (ROOT / "AGENTS.md").read_text()
    assert "<!-- UNPOPULATED" not in text
    assert "kb/reinicorn/README.md" in text
    assert "uv run rcorn" in text

def test_source_config_pins_reinicorn_scope() -> None:
    text = (ROOT / ".reinicorn-config").read_text()
    assert "REINICORN_KB_SCOPE=reinicorn" in text
    assert "REINS_" not in text
```

- [ ] **Step 2: Run the structural test to verify source files are still old**

```bash
uv run pytest tests/test_public_identity.py -v
```

Expected: failure because `.reinicorn-config` does not exist and root AGENTS remains unpopulated.

- [ ] **Step 3: Rename config and populate source AGENTS**

```bash
git mv .reins-config .reinicorn-config
```

Rename every key to `REINICORN_*`, every command to `rcorn`, and add:

```text
REINICORN_KB_SCOPE=reinicorn
```

Replace root `AGENTS.md` with a source-specific map containing:

```markdown
# Reinicorn

Reinicorn is a Python CLI and workflow skill set for spec-driven development with
AI coding agents. It serves engineering teams that keep intent, architecture,
plans, and quality controls in a shared Git-backed knowledge base.

## Build and test

- Python 3.12+; dependencies and environments are managed with uv.
- Run the CLI from source with `uv run rcorn`.
- Run tests with `uv run pytest tests/ -v`.
- Run lint with `uv run ruff check src/reinicorn tests`.
- Run type checking with `uv run pyright src/reinicorn`.
- Run structural and shell checks with `bash tests/run-all.sh`.

## Knowledge base

Read and follow `kb/reinicorn/README.md` before planning or changing code. Use
`rcorn` for every KB operation; never manage the KB submodule with raw Git.

## Project conventions

- Runtime identity constants live in `reinicorn.identity`.
- KB document-type paths and behavior come from `reinicorn.doc_types.REGISTRY`.
- Validate external input at boundaries and keep one concern per file.
- stdout is the agent-facing result surface; stderr is progress/debug only.
- Follow red-green TDD for behavior changes and use conventional commits.
```

Update workflow commands and tracked config paths to `rcorn`, `.reinicorn`, and
`src/reinicorn`. Do not change `.gitmodules` in Gate 1.

- [ ] **Step 4: Run source identity and CI-configuration tests**

```bash
uv run pytest tests/test_public_identity.py tests/test_source_of_truth.py tests/test_cli_shape.py -v
uv run ruff check src/reinicorn tests
```

Expected: tests and Ruff pass.

- [ ] **Step 5: Commit source-owned instructions**

```bash
git add .reinicorn-config AGENTS.md .gitignore .github tests/test_public_identity.py
git commit -m "docs(agents): populate Reinicorn source instructions"
```

---

### Task 7: Coordinate and publish the KB scope cutover

**Files:**
- Rename in KB: `kb/reins/` -> `kb/reinicorn/`
- Create if absent: `kb/reinicorn/README.md`
- Modify: parent submodule pointer `kb`

**Interfaces:**
- Consumes: `reinicorn.config.kb_scope()` and `.reinicorn-config` from Tasks 2 and 6.
- Produces: the sole live project scope `kb/reinicorn/`.

- [ ] **Step 1: Verify the shared cutover precondition**

Run:

```bash
uv run rcorn kb sync
uv run rcorn kb status
uv run rcorn plan show feature/reinicorn-identity-and-assets --full
```

Expected: the KB is aligned with `origin/main`, this plan is readable, and there is
no uncoordinated active branch. If another active branch is still changing
`kb/reins/`, stop and obtain the owner's explicit cutover approval before Step 2.

- [ ] **Step 2: Move the scope through the CLI escape hatch**

```bash
uv run rcorn kb git mv reins reinicorn
```

If the moved scope has no README, generate the exact README content from Task 5 at
`kb/reinicorn/README.md`. Do not retain a `kb/reins` symlink, redirect, or copy.

- [ ] **Step 3: Verify all KB commands resolve the configured scope**

```bash
uv run rcorn spec show reinicorn-public-release-program-and-identity-migration --full
uv run rcorn plan show feature/reinicorn-identity-and-assets --full
uv run rcorn kb lint
```

Expected: spec and plan resolve under `kb/reinicorn`; lint has zero error-severity failures. Existing warning families remain Gate 3 work.

- [ ] **Step 4: Publish the shared KB cutover**

```bash
uv run rcorn kb publish
```

Expected: `Kb pushed to remote main.` and KB `main` equals `origin/main`.

- [ ] **Step 5: Commit the parent pointer**

```bash
git add kb
git commit -m "chore(kb): rename project scope to reinicorn"
```

---

### Task 8: Rename current release-facing documentation and enforce the legacy scan

**Files:**
- Modify: `README.md`
- Modify: `GETTING-STARTED.md`
- Modify: `HUMANS.md` only if the scan finds current identity prose there
- Modify: `src/reinicorn/doc_types.py`
- Modify: user-facing strings in `src/reinicorn/**/*.py`
- Modify: `tests/test_public_identity.py`
- Modify: output expectations throughout `tests/`

**Interfaces:**
- Produces: a deterministic Gate 1 scan over runtime code, package inputs, installed assets, workflows, and current release docs.
- Excludes: `kb/`, `.gitmodules`, `presentation/`, caches, build output, and private historical material governed by Gate 4.

- [ ] **Step 1: Extend the structural test to fail on legacy public identity**

Add to `tests/test_public_identity.py`:

```python
PUBLIC_PATHS = (
    Path("src/reinicorn"),
    Path("templates"),
    Path(".agents"),
    Path("hooks"),
    Path("editor-hooks"),
    Path("linters"),
    Path("platform-instructions"),
    Path("workflows"),
    Path(".github/workflows"),
    Path("README.md"),
    Path("GETTING-STARTED.md"),
    Path("pyproject.toml"),
)

def test_release_inputs_contain_no_legacy_identity() -> None:
    offenders: list[str] = []
    for relative in PUBLIC_PATHS:
        path = ROOT / relative
        files = [path] if path.is_file() else [p for p in path.rglob("*") if p.is_file()]
        for file in files:
            text = file.read_text(errors="ignore")
            if "reins" in text.lower() or ".reins" in text or "REINS_" in text:
                offenders.append(str(file.relative_to(ROOT)))
    assert offenders == []
```

- [ ] **Step 2: Run the scan to collect the remaining current-tree offenders**

```bash
uv run pytest tests/test_public_identity.py::test_release_inputs_contain_no_legacy_identity -v
```

Expected: failure lists every remaining release-input file containing the legacy identity.

- [ ] **Step 3: Rename current documentation and output strings**

Use `Reinicorn` for prose, `reinicorn` for repository/distribution/import names,
and `rcorn` for commands. Replace unavailable PyPI instructions with a source-only
installation section (the inner fence is literal README content):

````markdown
## Install from source

```bash
git clone https://github.com/mnbiehl/reinicorn.git
cd reinicorn
uv tool install .
rcorn --version
```
````

Do not claim a PyPI release. Update doc-type create hints, errors, `next:` output,
workflow invocations, examples, and tests to `rcorn`. Keep private history and
`.gitmodules` outside this task.

- [ ] **Step 4: Run identity, CLI-output, and documentation tests**

```bash
uv run pytest tests/test_public_identity.py tests/test_cli_shape.py tests/test_output_conventions.py tests/test_console.py tests/commands/test_home.py tests/commands/test_status.py tests/commands/test_review.py -v
```

Expected: all selected tests pass and the legacy scan reports no offenders.

- [ ] **Step 5: Commit release-facing identity**

```bash
git add README.md GETTING-STARTED.md HUMANS.md src/reinicorn tests .github/workflows pyproject.toml
git commit -m "docs(identity): adopt Reinicorn release-facing names"
```

---

### Task 9: Verify initialization, update ownership, and built artifacts end to end

**Files:**
- Modify: `tests/test_public_identity.py`
- Modify: `tests/test_assets.py`
- Modify: `tests/test_update.py`
- Modify: `tests/commands/test_init.py`
- Modify: implementation files only when a failing Gate 1 verification test exposes a spec gap

**Interfaces:**
- Consumes: all Gate 1 behavior from Tasks 1-8.
- Produces: executable evidence for clean init, byte-preserving re-init/update, and final package contents.

- [ ] **Step 1: Add the final update ownership regression test**

Add to `tests/test_update.py`, using its existing
`_setup_repo_with_manifest()` and `_setup_package_assets()` helpers:

```python
def test_update_never_reclaims_user_owned_maps(tmp_path: Path) -> None:
    from reinicorn.commands.update import cmd_update

    repo = _setup_repo_with_manifest(tmp_path)
    agents = repo / "AGENTS.md"
    readme = repo / "kb" / "sample" / "README.md"
    readme.parent.mkdir(parents=True)
    agents.write_text("# User instructions\n")
    readme.write_text("# Team KB map\n")
    before = (agents.read_bytes(), readme.read_bytes())

    assets = _setup_package_assets(tmp_path)
    with patch("reinicorn.commands.update._get_package_version", return_value="99.0.0"), \
         patch("reinicorn.commands.update._get_repo_root", return_value=repo), \
         patch("reinicorn.commands.update._get_asset_sources", return_value=assets):
        assert cmd_update() == 0

    assert (agents.read_bytes(), readme.read_bytes()) == before
    manifest = json.loads((repo / ".reinicorn/manifest.json").read_text())
    assert "AGENTS.md" not in manifest["files"]
    assert not any(name.startswith("kb/") for name in manifest["files"])
```

Together with Task 4's re-init regression, this proves both commands preserve the
two ownership boundaries without adding a production-only test hook.

- [ ] **Step 2: Run the end-to-end ownership tests**

```bash
uv run pytest tests/commands/test_init.py tests/test_update.py tests/test_manifest.py tests/test_assets.py tests/test_public_identity.py -v
```

Expected: all selected tests pass.

- [ ] **Step 3: Build wheel and source archive**

```bash
build_dir=$(mktemp -d)
uv build --out-dir "$build_dir"
find "$build_dir" -maxdepth 1 -type f -print
```

Expected: the temporary directory contains `reinicorn-*.whl` and
`reinicorn-*.tar.gz`; no `reins-*` archive exists. Keep `$build_dir` for Step 4.

- [ ] **Step 4: Inspect and smoke-test the wheel in isolation**

```bash
python -m zipfile -l "$build_dir"/reinicorn-*.whl
tmpdir=$(mktemp -d)
uv venv "$tmpdir/venv"
uv pip install --python "$tmpdir/venv/bin/python" "$build_dir"/reinicorn-*.whl
"$tmpdir/venv/bin/python" -c 'import reinicorn; print(reinicorn.__version__)'
"$tmpdir/venv/bin/rcorn" --version
test ! -e "$tmpdir/venv/bin/reins"
```

Expected: archive paths use `reinicorn/`; the generic template and all managed
asset groups are present; import and `rcorn` succeed; no `reins` executable exists.

- [ ] **Step 5: Run the complete Gate 1 verification suite**

```bash
uv run pytest tests/ -v
uv run ruff check src/reinicorn tests
uv run pyright src/reinicorn
bash tests/run-all.sh
uv run rcorn kb lint
git diff --check
```

Expected: pytest has zero failures; Ruff and Pyright have zero errors; shell checks
exit 0; KB lint has zero error-severity failures; `git diff --check` prints nothing.

- [ ] **Step 6: Commit final regression coverage**

```bash
git add tests src/reinicorn pyproject.toml
git commit -m "test(release): verify Reinicorn identity and asset ownership"
```

- [ ] **Step 7: Publish final plan progress and record evidence**

Mark every completed checkbox with the actual command results, then run:

```bash
uv run rcorn kb publish
uv run rcorn plan status
git status --short --branch
```

Expected: KB is published, the plan reports complete task progress, and the source
branch contains only intentional committed changes plus the current KB pointer if it
has not yet received its final parent commit.
