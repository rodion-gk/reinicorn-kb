---
type: plan
title: 'Execution Plan: feat-skillset-adapters'
slug: feat-skillset-adapters
lifecycle: done
status: complete
created: 2026-08-18
author: Michael Biehl
branch: feat-skillset-adapters
ticket: N/A
spec: reinicorn/specs/skill-base-agnostic-reinicorn-adapter-infrastructure-for-ext.md
---

# Execution Plan: feat-skillset-adapters

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Goal

Plan 1 of the skill-base-agnostic spec: the adapter infrastructure and the superpowers cutover. Build the declarative patch engine (`src/reinicorn/skillset/`), the `rcorn skills` CLI family with lockfile and transactional installs, the registry-driven wiring renderer, the bundled `superpowers` adapter, the methodology-neutral `using-reinicorn` rewrite, and the migration that removes the vendored superpowers forks from the repo and from user projects. Plan 2 (separate branch) covers the `mattpocock-skills` adapter and the `authoring-skillset-adapters` skill.

**Branch flow:** this branch is stacked on `feat-registry-doc-types-stage2` (PR #51) because the wiring renderer consumes stage-1/2 registry fields (`help_text`, `create_verb`, `addressing`). PR into `main` after #51 merges (rebase onto main then), or into the registry integration branch if #51 is still open at finish time.

**Architecture:** a new `src/reinicorn/skillset/` package with five focused modules — `adapter.py` (parse + validate `adapter.yaml` at the boundary), `fetch.py` (commit-pinned tarball fetch, cache, digest), `engine.py` (canonical-order patch application into a staging tree), `lockfile.py` (`.reinicorn/skillset-lock.json`), `wiring.py` (registry-driven wiring doc), plus `installer.py` (the transaction that composes them). CLI surface is one hand-wired `skills` group in `cli.py` dispatching to `commands/skills_cmds.py`. Bundled adapters live in a new top-level `adapters/` directory shipped as package assets.

**Tech Stack:** Python 3.12, argparse, PyYAML (existing dep), `git apply` via subprocess, uv, pytest.

## Global Constraints

- Spec is the contract: `reinicorn/specs/skill-base-agnostic-reinicorn-adapter-infrastructure-for-ext.md` (approved). Where this plan and the spec disagree, the spec wins.
- Commit-SHA-only pins: `source.commit` must match `^[0-9a-f]{40}$`; tags are never accepted as pins (golden-principle-4 error tells the author to resolve the tag).
- Canonical patch order, always: `exclude` → `patch` series (listed order) → `append` (listed order) → `override`. Contradictions (patching an excluded or overridden file) are load-time `AdapterError`s.
- Every error message: what went wrong, where, how to fix (golden principle 4). All user-facing errors go through `reinicorn.console.error`.
- Validate at boundaries (golden principle 1): `load_adapter` and `read_lock` are the only places adapter/lockfile shapes are checked; everything downstream trusts typed objects. No YOLO dict probing (principle 3).
- No network in CI: every test fetches via a `file://` tarball fixture (monkeypatched `tarball_url`); integration tests use vendored fixture trees under `tests/skillset/fixtures/`.
- Doc-type knowledge comes only from `reinicorn.doc_types.REGISTRY`; branch on enum members with `is`, never string comparison.
- Red-green TDD; conventional commits; frequent commits.
- Gate per task: `uv run pytest tests/ -v`, `uv run ruff check src/reinicorn tests`, `uv run pyright src/reinicorn`, `bash tests/run-all.sh`.
- Implementer subagents: never push, never run the installed rcorn CLI against this repo's kb, don't touch `kb/`.

## Acceptance Criteria

- [x] `rcorn skills install superpowers` on a fixture project produces patched skills in the configured skills dir, `.reinicorn/skillset-lock.json`, the generated wiring doc, and the compatibility link — atomically (any induced failure leaves the project byte-identical).
- [x] Same adapter + same pin ⇒ byte-identical output (double-install idempotence test).
- [x] A patch that does not apply fails loudly naming the patch file, target, and remediation; nothing is written.
- [x] Wiring doc renders one row per `REGISTRY` type (phantom-type test passes); unknown non-optional wiring keys error listing actual registry types; optional-absent keys are skipped and reported.
- [x] Skills dir and compatibility link are configurable via `REINICORN_SKILLS_DIR` / `REINICORN_SKILLS_LINK` (`none` disables the link).
- [x] The repo contains zero third-party skill text: the 14 superpowers forks, `update-superpowers`, and the forks' `ATTRIBUTION.md` are deleted; `.agents/skills/` ships only `using-reinicorn` and `populate-agents-md`; adapter-installed skills are gitignored and the repo dogfoods `rcorn skills install superpowers`.
- [x] `using-reinicorn` is methodology-neutral: no superpowers skill names, no hardcoded doc nouns; points at the generated wiring doc; states the no-skills-listed fallback rule.
- [x] `rcorn update` on a legacy project detects the old forks via the manifest and offers the superpowers-adapter migration; locally modified forks are never silently deleted.
- [x] Full gate green; PR body cites the spec and discloses behavior changes.

## File Structure

```
adapters/superpowers/adapter.yaml         # bundled adapter definition
adapters/superpowers/patches/*.patch      # git patches vs pinned upstream
adapters/superpowers/appends/*.md         # Reinicorn trailer blocks
adapters/superpowers/files/ATTRIBUTION.md
src/reinicorn/skillset/__init__.py
src/reinicorn/skillset/adapter.py         # AdapterError, dataclasses, load_adapter
src/reinicorn/skillset/fetch.py           # tarball_url, fetch_source, cache+digest
src/reinicorn/skillset/engine.py          # patch_touched_paths, build_staging
src/reinicorn/skillset/lockfile.py        # SkillsetLock, read_lock, write_lock
src/reinicorn/skillset/wiring.py          # render_wiring, write_wiring
src/reinicorn/skillset/installer.py       # install_adapter, update_adapter (transaction)
src/reinicorn/commands/skills_cmds.py     # cmd_skills_{install,status,update,list}
tests/skillset/test_{adapter,fetch,engine,lockfile,wiring,installer}.py
tests/commands/test_skills_cmds.py
tests/skillset/fixtures/upstream-tree/    # minimal fake skill set (2 skills)
```

## Tasks

### Task 1: Adapter model and boundary validation (`adapter.py`)

**Files:**
- Create: `src/reinicorn/skillset/__init__.py` (empty), `src/reinicorn/skillset/adapter.py`
- Test: `tests/skillset/__init__.py` (empty), `tests/skillset/test_adapter.py`

**Interfaces:**
- Produces (all later tasks consume these exact types):

```python
class AdapterError(Exception):
    """Agent-readable adapter failure (message carries what/where/fix)."""

@dataclass(frozen=True)
class AdapterSource:
    repo: str        # "owner/name"
    commit: str      # 40-hex sha (the pin)
    annotation: str  # human label, e.g. "v5.0.6" — never used for fetching

@dataclass(frozen=True)
class WiringEntry:
    skills: tuple[str, ...]
    optional: bool = False

@dataclass(frozen=True)
class Adapter:
    name: str
    source: AdapterSource
    skills: dict[str, str]              # upstream dir path -> installed skill name
    patches: tuple[str, ...]            # adapter-relative *.patch, listed order
    appends: dict[str, tuple[str, ...]] # installed name -> adapter-relative .md blocks
    excludes: tuple[str, ...]           # upstream-relative file paths to drop
    overrides: dict[str, str]           # installed-relative path -> adapter-relative file
    files: dict[str, str]               # installed-relative path -> adapter-relative file
    wiring: dict[str, WiringEntry]
    root: Path                          # adapter dir, resolves the relative paths above

def load_adapter(adapter_dir: Path) -> Adapter: ...
```

- [ ] **Step 1: Write failing tests** in `tests/skillset/test_adapter.py`. Build a `make_adapter_dir(tmp_path, yaml_text, extra_files=...)` helper that writes `adapter.yaml` plus referenced patch/append files. Cover: (a) happy path parses the full shape below; (b) tag-as-pin (`commit: v5.0.6`) raises `AdapterError` matching `commit-SHA` and `resolve the tag`; (c) missing referenced patch file raises naming the path; (d) wiring shorthand `spec: [brainstorming]` and expanded `prd: {skills: [x], optional: true}` both parse to `WiringEntry`; (e) empty `skills:` raises; (f) unknown top-level key raises (fail closed).

Canonical fixture YAML (also the documented shape — keep in the module docstring):

```yaml
name: demo
source:
  repo: acme/skills
  commit: 0123456789abcdef0123456789abcdef01234567
  annotation: v1.0.0
skills:
  skills/alpha: alpha
  skills/nested/beta: beta
patches:
  - patches/alpha-kb-paths.patch
appends:
  alpha:
    - appends/alpha-reinicorn.md
excludes:
  - skills/alpha/scratch.md
overrides:
  beta/references/template.md: overrides/template.md
files:
  ATTRIBUTION.md: files/ATTRIBUTION.md
wiring:
  spec: [alpha]
  prd:
    skills: [alpha]
    optional: true
```

- [ ] **Step 2: Run to verify failure** — `uv run pytest tests/skillset/test_adapter.py -v`. Expected: FAIL (module missing).

- [ ] **Step 3: Implement `adapter.py`.** `load_adapter` reads `adapter_dir / "adapter.yaml"` with `yaml.safe_load`, validates every field explicitly (regexes: repo `^[\w.-]+/[\w.-]+$`, commit `^[0-9a-f]{40}$`), rejects unknown top-level keys, verifies each referenced adapter-relative file exists under `root` (and refuses `..`/absolute components — path-safety at the boundary), normalizes wiring shorthand/expanded forms. Error template:

```python
raise AdapterError(
    f"Adapter '{name}' at {adapter_dir}: source.commit '{commit}' is not a "
    f"40-hex commit SHA.\n  Tags are not valid pins — resolve the tag to its "
    f"commit and pin that (see spec: skill-base-agnostic adapter source rules)."
)
```

- [ ] **Step 4: Run to verify pass**, then full gate.
- [ ] **Step 5: Commit** — `feat(skillset): adapter model and boundary validation`

### Task 2: Commit-pinned fetch with cache and digest (`fetch.py`)

**Files:**
- Create: `src/reinicorn/skillset/fetch.py`
- Test: `tests/skillset/test_fetch.py`

**Interfaces:**
- Consumes: `AdapterSource`, `AdapterError` (Task 1); `sha256_file` from `reinicorn.manifest`.
- Produces:

```python
def tarball_url(source: AdapterSource) -> str:
    return f"https://codeload.github.com/{source.repo}/tar.gz/{source.commit}"

def default_cache_dir() -> Path:  # $REINICORN_CACHE_DIR or ~/.cache/reinicorn/skillsets

def fetch_source(
    source: AdapterSource, cache_dir: Path, *, expected_digest: str | None = None,
) -> tuple[Path, str]:
    """Return (extracted tree root, archive sha256). Uses the cached tarball
    when present; downloads via urllib otherwise. If expected_digest is given
    (from the lockfile) and the archive digest differs, AdapterError."""
```

- [ ] **Step 1: Write failing tests.** Build a fixture tarball from `tests/skillset/fixtures/upstream-tree/` (create the fixture in this task: `skills/alpha/SKILL.md` with a `docs/plans/` string to patch later, `skills/alpha/scratch.md`, `skills/nested/beta/SKILL.md`, `skills/nested/beta/references/template.md`) whose top-level directory is named `skills-<sha>` like codeload produces. Monkeypatch `fetch.tarball_url` to return `file://<fixture.tar.gz>`. Cover: (a) fetch extracts and returns the inner tree root + digest; (b) second call hits the cache (delete the fixture file first — still succeeds); (c) `expected_digest="0"*64` mismatch raises `AdapterError` naming expected vs actual; (d) extraction uses `filter="data"` (tarfile member with absolute path raises).
- [ ] **Step 2: Verify failure.**
- [ ] **Step 3: Implement.** Cache key `<cache_dir>/<owner>__<name>__<commit>.tar.gz`; download with `urllib.request.urlopen` to a `.part` file then rename; digest with `sha256_file`; extract to a `tempfile.mkdtemp` dir with `tarfile.extractall(filter="data")`; return the single top-level directory. Network errors wrap in `AdapterError` naming the repo, commit, and URL.
- [ ] **Step 4: Verify pass + gate.**
- [ ] **Step 5: Commit** — `feat(skillset): commit-pinned source fetch with cache and digest`

### Task 3: Patch engine with canonical order (`engine.py`)

**Files:**
- Create: `src/reinicorn/skillset/engine.py`
- Test: `tests/skillset/test_engine.py`

**Interfaces:**
- Consumes: `Adapter`, `AdapterError` (Task 1).
- Produces:

```python
def patch_touched_paths(patch_text: str) -> set[str]:
    """Upstream-relative paths a unified diff touches (from 'diff --git a/X b/X')."""

def validate_patch_targets(adapter: Adapter) -> None:
    """AdapterError if any patch touches an excluded file or a file whose
    installed location is overridden (canonical-order contradiction)."""

def build_staging(adapter: Adapter, source_tree: Path, staging: Path) -> dict[str, str]:
    """Apply exclude -> patch -> append -> override -> files into `staging`
    (installed layout). Returns {installed rel path: sha256}."""
```

- [ ] **Step 1: Write failing tests** using the Task 2 fixture tree (as a plain directory — no fetch involved). Author a real fixture patch in the test (heredoc string written to the adapter dir) that rewrites `docs/plans/` to `kb/{repo}/exec-plans/` in `skills/alpha/SKILL.md`. Cover: (a) staging contains `alpha/SKILL.md` (patched content) and `beta/...` per the `skills` mapping; (b) excluded `alpha/scratch.md` absent; (c) append block present at end of `alpha/SKILL.md` separated by one blank line; (d) override replaced `beta/references/template.md`; (e) `files` entry landed at staging root; (f) a patch with stale context raises `AdapterError` matching the patch filename and `rebase this adapter`; (g) `validate_patch_targets` rejects a patch touching an excluded path; (h) returned hash map covers exactly the staged files; (i) running `build_staging` twice into two dirs yields identical hash maps (determinism).
- [ ] **Step 2: Verify failure.**
- [ ] **Step 3: Implement.** Copy `source_tree` to a temp worktree; delete excludes; apply each patch with `subprocess.run(["git", "apply", "--whitespace=nowarn", patch_path], cwd=worktree, capture_output=True, text=True)` — on nonzero returncode raise:

```python
raise AdapterError(
    f"Adapter '{adapter.name}': patch {rel_patch} does not apply to "
    f"{adapter.source.repo}@{adapter.source.commit[:12]}.\n"
    f"  git apply said: {proc.stderr.strip()}\n"
    f"  How to fix: run the authoring-skillset-adapters skill to rebase this adapter."
)
```

then map `skills` dirs into `staging/<installed name>`, append blocks (`existing.rstrip() + "\n\n" + block.rstrip() + "\n"`), copy overrides and files, and hash everything with `sha256_file`.
- [ ] **Step 4: Verify pass + gate.**
- [ ] **Step 5: Commit** — `feat(skillset): patch engine with canonical exclude→patch→append→override order`

### Task 4: Lockfile (`lockfile.py`) and config keys

**Files:**
- Create: `src/reinicorn/skillset/lockfile.py`
- Modify: `src/reinicorn/identity.py` (add `SKILLSET_LOCK_FILE_NAME = "skillset-lock.json"`, `SKILLS_DIR_KEY = "REINICORN_SKILLS_DIR"`, `SKILLS_LINK_KEY = "REINICORN_SKILLS_LINK"`)
- Modify: `src/reinicorn/config.py` (add `skills_dir(root) -> Path` and `skills_link(root) -> Path | None`)
- Test: `tests/skillset/test_lockfile.py`, `tests/test_config.py` (extend)

**Interfaces:**
- Consumes: `config_get` (existing), `STATE_DIR_NAME`.
- Produces:

```python
@dataclass(frozen=True)
class SkillsetLock:
    adapter: str
    repo: str
    commit: str
    archive_sha256: str
    files: dict[str, str]                # skills-dir-relative path -> sha256
    wiring: dict[str, WiringEntry]       # persisted so update/init can re-render
                                         # the wiring doc without the adapter dir
                                         # (local-path adapters aren't resolvable
                                         # later); serialized as
                                         # {type: {skills: [...], optional: bool}}

def read_lock(repo_root: Path) -> SkillsetLock | None   # None if missing/malformed (warn)
def write_lock(repo_root: Path, lock: SkillsetLock) -> Path
# config.py
def skills_dir(root: Path | None = None) -> Path      # default Path(".agents/skills"), relative to root
def skills_link(root: Path | None = None) -> Path | None  # default .claude/skills; "none" -> None
```

- [ ] **Step 1: Failing tests** — round-trip write/read; malformed JSON → `None`; missing keys → `None` (boundary validation mirrors `read_manifest`); `skills_dir` honors `REINICORN_SKILLS_DIR=tools/skills` in `.reinicorn-config`; `skills_link` returns `None` for `REINICORN_SKILLS_LINK=none`.
- [ ] **Step 2: Verify failure.** **Step 3: Implement** (lockfile lives at `<root>/.reinicorn/skillset-lock.json`). **Step 4: Verify pass + gate.**
- [ ] **Step 5: Commit** — `feat(skillset): lockfile and configurable skills dir/link`

### Task 5: Wiring renderer (`wiring.py`)

**Files:**
- Create: `src/reinicorn/skillset/wiring.py`
- Test: `tests/skillset/test_wiring.py`

**Interfaces:**
- Consumes: `REGISTRY`, `Addressing`, `TitleSource` from `reinicorn.doc_types` (stage-1/2 fields: `key`, `create_verb`, `help_text`, `addressing`, `title_source`); `WiringEntry` (Task 1).
- Produces:

```python
def render_wiring(wiring: dict[str, WiringEntry] | None) -> tuple[str, list[str]]:
    """(markdown, skipped_optional_keys). AdapterError on a non-optional key
    absent from REGISTRY, listing sorted(REGISTRY) in the message."""

def write_wiring(repo_root: Path, wiring: dict[str, WiringEntry] | None) -> Path:
    """Render and write <skills_dir>/using-reinicorn/references/skillset-wiring.md."""
```

Rendered shape — one row per registry type (registry is the row source, wiring only fills the skills column):

```markdown
# Skillset Wiring

Generated by rcorn — do not edit. Regenerated by `rcorn update` and `rcorn skills`.

| Doc type | Create with | Invoke first |
|---|---|---|
| spec | `rcorn spec create "<title>"` | brainstorming |
| plan | `rcorn plan create` | writing-plans, executing-plans |
| idea | `rcorn idea create "<text>"` | — |
```

The create-command cell derives from the registry row: TITLE → `rcorn {key} {create_verb} "<title>"`, FREE_TEXT → `rcorn {key} {create_verb} "<text>"`, NONE → `rcorn {key} {create_verb}`.

- [ ] **Step 1: Failing tests** — (a) with `wiring=None` every registry key appears with `—` in the skills column; (b) phantom type: insert a synthetic slug-addressed row into a copied REGISTRY (monkeypatch) and assert it gets a row with no wiring change; (c) unknown non-optional key `bogus` raises listing the actual keys; (d) optional-absent key is skipped and returned in `skipped`; (e) `write_wiring` creates parent dirs and returns the path under the configured skills dir.
- [ ] **Step 2–4: red, implement, green + gate.**
- [ ] **Step 5: Commit** — `feat(skillset): registry-driven wiring doc renderer`

### Task 6: Transactional installer (`installer.py`)

**Files:**
- Create: `src/reinicorn/skillset/installer.py`
- Test: `tests/skillset/test_installer.py`

**Interfaces:**
- Consumes: everything above, plus `read_manifest` (native-file ownership) and `_link_claude_skills`'s logic from `commands/init.py`.
- Produces:

```python
def maintain_link(repo_root: Path) -> None:
    """Extracted from init._link_claude_skills, generalized to config paths:
    make skills_link a relative symlink to skills_dir (copy-and-warn fallback,
    real-dir left untouched with warning, no-op when link is None)."""

def install_adapter(adapter: Adapter, repo_root: Path, *, cache_dir: Path | None = None) -> None:
    """Fetch -> validate -> stage -> collision-check -> commit transaction:
    skill files + lockfile + wiring doc + link together; any failure restores
    the previous state exactly. Raises AdapterError."""

def update_adapter(adapter: Adapter, repo_root: Path, *, force: bool = False,
                   cache_dir: Path | None = None) -> list[str]:
    """Reinstall against the existing lock. Inventory diff: files in the old
    lock but not the new staging are removed when hash-unmodified, preserved
    when locally modified. Locally modified files that the new staging would
    overwrite abort (listed) unless force. Returns preserved-drift paths."""
```

- [ ] **Step 1: Failing tests** (fixture project = `tmp_path` with a git-less layout, manifest listing a fake native `using-reinicorn/SKILL.md`): (a) fresh install writes skills, lock, wiring doc, link; (b) double install is byte-identical (hash the whole skills dir); (c) collision with the manifest-listed native skill name raises and writes nothing; (d) collision with an unmanaged existing dir raises and writes nothing; (e) induced failure (monkeypatch `write_lock` to raise) leaves the project byte-identical to pre-install (snapshot/compare); (f) update where new staging drops a file: unmodified → removed, locally modified → preserved and returned; (g) update over a locally modified file that changed upstream aborts without `--force`, listing the path.
- [ ] **Step 2–3: red, implement.** Transaction mechanics: compute the full write-set first; back up every path that will be created/replaced/removed into a temp dir; apply; on any exception restore backups and re-raise. Ownership check: a staging target may only overwrite paths present in the old lock's `files`; anything else existing on disk (native per manifest, or unmanaged) is a collision `AdapterError` naming the path and the owner.
- [ ] **Step 4: green + gate.** Also refactor `init.py` `_link_claude_skills` to delegate to `maintain_link` (its two call sites keep behavior; existing init tests stay green).
- [ ] **Step 5: Commit** — `feat(skillset): transactional adapter installer with ownership checks`

### Task 7: `rcorn skills` CLI group

**Files:**
- Create: `src/reinicorn/commands/skills_cmds.py`
- Modify: `src/reinicorn/cli.py` (hand-wired `skills` group + 4 `_DISPATCH` rows, after the mode group)
- Modify: `pyproject.toml` (force-include `"adapters" = "reinicorn/_data/adapters"`)
- Test: `tests/commands/test_skills_cmds.py`, `tests/test_cli_shape.py` (extend)

**Interfaces:**
- Produces: `cmd_skills_install(name_or_path)`, `cmd_skills_status()`, `cmd_skills_update(ref=None, force=False)`, `cmd_skills_list()` — all `-> int`. Adapter resolution: an existing directory path wins; otherwise `get_asset_path(f"adapters/{name}")`; miss → error listing bundled adapter names.

Parser block (mirrors the house style of the other hand-wired groups):

```python
skills_p = sub.add_parser("skills", help="Skill-set adapter management (install, status, update, list)")
skills_sub = skills_p.add_subparsers(dest="skills_command")
skills_sub.required = True
skills_install_p = skills_sub.add_parser("install", help="Install a skill-set adapter")
skills_install_p.add_argument("adapter", help="Bundled adapter name or path to an adapter directory")
skills_sub.add_parser("status", help="Installed adapter, pin, and local drift")
skills_update_p = skills_sub.add_parser("update", help="Re-fetch and re-apply the installed adapter")
skills_update_p.add_argument("--ref", dest="ref", default=None, help="New commit SHA to pin")
skills_update_p.add_argument("--force", action="store_true", help="Overwrite locally modified files")
skills_sub.add_parser("list", help="List bundled adapters")
```

- [ ] **Step 1: Failing tests** — dispatch pins (patch `reinicorn.skillset.installer.install_adapter` etc. and assert calls, matching `test_dispatch.py` style); `skills status` with no lock prints `no adapter installed` and exits 0; `--ref` with a non-SHA errors before any fetch; shape test asserts the group + verbs exist.
- [ ] **Step 2–4: red, implement (thin command layer: resolve adapter, call installer, catch `AdapterError` → `console.error(str(e)); return 1`), green + gate.**
- [ ] **Step 5: Commit** — `feat(cli): rcorn skills group (install/status/update/list)`

### Task 8: Bundled `superpowers` adapter

**Files:**
- Create: `adapters/superpowers/adapter.yaml`, `adapters/superpowers/patches/*.patch`, `adapters/superpowers/appends/*.md`, `adapters/superpowers/files/ATTRIBUTION.md`
- Test: `tests/skillset/test_superpowers_adapter.py`

**Interfaces:**
- Consumes: the current forks in `.agents/skills/` (source of truth for intended content), the upstream cache `~/.claude/plugins/cache/claude-plugins-official/superpowers/<latest>/`, `replacements.yaml` in `.agents/skills/update-superpowers/`.

- [ ] **Step 1: Resolve the pin.** Read the forks' `upstream:` frontmatter version (e.g. `superpowers/v5.0.6`); resolve the matching tag on the upstream GitHub repo to a commit SHA: `gh api repos/obra/superpowers/git/ref/tags/v<version> -q .object.sha` (dereference annotated tags via `git/tags/<sha>` if `.object.type == "tag"`). Record SHA + `annotation: v<version>`.
- [ ] **Step 2: Generate content.** For each of the 14 forked skills (every `.agents/skills/` dir except `using-reinicorn`, `populate-agents-md`, `update-superpowers`): split the local fork at the first `^## Reinicorn ` heading — the tail becomes `appends/<skill>-reinicorn.md`; diff the upstream `skills/<skill>/SKILL.md` against the head (fork minus trailer, minus the `upstream:` frontmatter line) — a non-empty diff becomes `patches/<skill>.patch` with `a/skills/<skill>/...` paths. Write `adapter.yaml`: `skills:` maps `skills/<skill>` → `<skill>` for all 14; wiring `spec: [brainstorming]`, `plan: [writing-plans, executing-plans, subagent-driven-development]`; `files: {ATTRIBUTION.md: files/ATTRIBUTION.md}` (move the current `.agents/skills/ATTRIBUTION.md` content, dropping the fork-specific wording).
- [ ] **Step 3: Failing-then-passing test.** `test_superpowers_adapter.py`: `load_adapter(repo_root / "adapters/superpowers")` succeeds (schema-valid, all referenced files present, patch targets consistent via `validate_patch_targets`). No network: this test validates the definition, not the fetch.
- [ ] **Step 4: Live verification (manual, once, not in CI).** `rcorn skills install adapters/superpowers` into a scratch project; diff every installed skill against the current fork — byte-identical except the dropped `upstream:` frontmatter line (that metadata now lives in the lockfile). Fix patches until clean. Record the diff-clean evidence in the PR body.
- [ ] **Step 5: Commit** — `feat(adapters): bundled superpowers adapter (patches, trailers, attribution)`

### Task 9: Methodology-neutral `using-reinicorn` rewrite

**Files:**
- Modify: `.agents/skills/using-reinicorn/SKILL.md`
- Test: `tests/test_assets.py` (extend with content assertions)

- [ ] **Step 1: Failing content tests** — assert the shipped `using-reinicorn/SKILL.md` (a) does not mention `brainstorming`, `writing-plans`, `superpowers`, or any forked-skill name; (b) references `references/skillset-wiring.md`; (c) contains the fallback sentence about the creation command sufficing.
- [ ] **Step 2: Rewrite.** Keep: the skill-invocation discipline (EXTREMELY-IMPORTANT block, red-flags table), platform adaptation, CLI quick reference (unchanged this branch; registry stage 5 genericizes it), doc-creation rule, golden principles pointer, user-instructions precedence. Replace the "Skill Priority" section and every named-skill reference with:

```markdown
## Doc-Type Wiring

Before authoring any registered doc type, consult
`references/skillset-wiring.md` (generated — lists every doc type, its
creation command, and the skill(s) to invoke first). Invoke the listed
skill(s) before creating the doc. When a doc type lists no skills — no
skill-set adapter is installed — the creation command alone is the
contract: create the doc directly via the CLI.

Process skills (debugging, planning, review) come from your installed
skill set; when a task matches one, invoke it before acting. Reinicorn
takes no position on methodology — only on where docs live and how they
are created.
```

- [ ] **Step 3: green + gate. Step 4: Commit** — `feat(skills): methodology-neutral using-reinicorn`

### Task 10: Remove the forks; dogfood the adapter

**Files:**
- Delete: the 14 forked skill dirs, `.agents/skills/update-superpowers/`, `.agents/skills/ATTRIBUTION.md`
- Modify: `.gitignore` (adapter-installed skill dirs), `tests/test_skill_copy.py` (delete; superseded by Task 8's test + `tests/test_assets.py`), `tests/test_assets.py` (native set = `using-reinicorn`, `populate-agents-md`)
- Modify: `src/reinicorn/manifest.py` — `_collect_files` skips paths listed in the skillset lock (adapter files are not `rcorn update`-managed)
- Modify: `src/reinicorn/commands/update.py` — `_collect_package_files` result no longer contains fork files (they left the assets), and the sync loop skips lock-owned paths

- [ ] **Step 1: Failing tests** — `test_assets.py` asserts the bundled skills asset contains exactly the two native skills; `tests/test_manifest.py` (extend): with a lockfile owning `.agents/skills/brainstorming/SKILL.md`, `_collect_files` excludes it while native files stay.
- [ ] **Step 2: Delete + implement.** Gitignore entries: one line per adapter-installable path is unmaintainable — ignore everything under `.agents/skills/` except the native two:

```gitignore
.agents/skills/*
!.agents/skills/using-reinicorn/
!.agents/skills/populate-agents-md/
```

- [ ] **Step 3:** Run `rcorn skills install superpowers` in this repo (dogfood); verify `git status` stays clean and the installed skills load.
- [ ] **Step 4: green + full gate. Step 5: Commit** — `feat!: remove vendored superpowers forks; repo dogfoods the adapter`

### Task 11: Legacy migration in `rcorn update` + upgrade note

**Files:**
- Modify: `src/reinicorn/commands/update.py` (migration detect/offer, before the version-equality early return, after the submodule migration block)
- Create: `upgrades/v<version>.md` where `<version>` is the next unreleased version from `src/reinicorn/_version.py`
- Test: `tests/commands/test_update.py` (extend)

**Interfaces:**
- Consumes: `read_manifest`, `read_lock`, `install_adapter`, `get_asset_path("adapters/superpowers")`.

- [ ] **Step 1: Failing tests** — fixture repo whose manifest lists `.agents/skills/brainstorming/SKILL.md` (a legacy fork) and no skillset lock: (a) `cmd_update` prompts (`input` patched) and on `y` calls `install_adapter` with the superpowers adapter, removes hash-clean legacy fork files, and never re-adds them from assets; (b) a hash-drifted legacy fork file is preserved and warned, never deleted; (c) a repo with a lockfile present is not prompted (already migrated); (d) answer `n` leaves the forks in place (adapter-less opt-out) and records `REINICORN_SKILLSET_MIGRATION=declined` via the existing `config_set` (add the key to `identity.py`); the prompt is skipped when that key or a lockfile is present.
- [ ] **Step 2–3: red, implement.** Detection: manifest `files` contains a path under the skills dir whose top-level dir is not a current bundled native skill and no lock exists. Removal honors the same rule as everywhere: hash-match → delete, drift → preserve + warn.
- [ ] **Step 4:** Write `upgrades/v<next>.md` (house format): Skills — forks replaced by `rcorn skills install superpowers`; update-superpowers removed; Breaking — `.agents/skills` content ownership split.
- [ ] **Step 5: green + full gate. Commit** — `feat(update): migrate legacy superpowers forks to the adapter`

### Task 12: Wiring regeneration hooks, docs, and finish

**Files:**
- Modify: `src/reinicorn/commands/update.py` and `src/reinicorn/commands/init.py` — after asset sync, call `write_wiring(repo_root, lock.wiring if (lock := read_lock(repo_root)) else None)` so the wiring doc always exists (registry-only when no adapter installed).
- Modify: `platform-instructions/` copies that enumerate the superpowers skills (grep for skill names; update to reference the wiring doc)
- Test: `tests/commands/test_update.py`, `tests/commands/test_init_platforms.py` (extend: wiring doc exists post-run)

- [ ] **Step 1: Failing tests; Step 2: implement; Step 3: green + full gate** (`uv run pytest tests/ -v`, `ruff`, `pyright`, `bash tests/run-all.sh`).
- [ ] **Step 4:** Self-review the branch diff against the spec's acceptance list; update this plan's checkboxes; `rcorn kb publish`.
- [ ] **Step 5:** PR into `main` (or the registry integration branch if #51 is unmerged), body: spec citation, gate evidence, Task 8's diff-clean evidence, breaking-change disclosure. Do not merge without review (public-repo process).

## Dependencies

- **Stacked on** `feat-registry-doc-types-stage2` (PR #51) — wiring renderer needs stage-1/2 registry fields. Rebase onto `main` once #51 lands.
- Overlaps to watch: `feat-unified-kb-doc-frontmatter` (doc frontmatter — no file overlap expected); registry stage 5 will later genericize the `using-reinicorn` CLI table this plan leaves alone.
- Plan 2 (`mattpocock-skills` adapter + `authoring-skillset-adapters` skill) builds on this branch's engine; not started here.
