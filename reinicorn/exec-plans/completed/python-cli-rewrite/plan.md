---
type: plan
title: 'Execution Plan: python-cli-rewrite'
slug: python-cli-rewrite
lifecycle: done
status: complete
created: 2026-02-15
author: agent (Claude Code)
branch: python-cli-rewrite
ticket: N/A
---

# Execution Plan: python-cli-rewrite

## Goal
Rewrite the reins CLI from ~2,500 lines of bash to Python. Eliminate the `jq` dependency, gain real testing with pytest, and establish a maintainable codebase that scales to planned features (PR crawl, etc.).

## Acceptance Criteria
- [x] All CLI commands ported to Python (sync, publish, status, lint, plan, idea, enable/disable/incognito, init, attach, hooks install)
- [x] Git hooks delegate to Python internal commands via thin bash wrappers
- [x] Linter framework ported with Python rule classes + external .sh fallback
- [x] Extension system uses stdlib `json` (no `jq`)
- [x] 77 tests passing with pytest
- [x] CI workflow running `uv run pytest`
- [x] Zero external Python dependencies for CLI (stdlib only)
- [x] `pyproject.toml` with proper `src/` layout and `uv` tooling

## Approach
Standard Python packaging with `src/` layout, `hatchling` build backend, `console_scripts` entry point. Centralized `git.py` module as single mock-point for all git subprocess calls. `argparse` with nested subparsers for CLI dispatch. ANSI color output respecting NO_COLOR/FORCE_COLOR/isatty.

## Tasks
- [x] Step 0: Housekeeping — amend commit with fleshed-out PR crawl idea
- [x] Step 1: Create Python project structure (`pyproject.toml`, `src/reins/`, all module stubs)
- [x] Step 2: Core modules + tests (console, config, git, harness, mode)
- [x] Step 3: CLI dispatcher + simple commands (help, version, enable/disable/incognito)
- [x] Step 4: Medium commands (idea, plan create/status, status, sync, publish)
- [x] Step 5: Git hook delegation (internal commands: hook-check, post-checkout, pre-push, post-merge)
- [x] Step 6: Scripts as Python modules (hooks_install, extensions, init_project, attach)
- [x] Step 7: Linter framework (runner, cross_links, docs_freshness, plan_structure rules)
- [x] Step 8: Test infrastructure + CI (conftest fixtures, GitHub Actions workflow)
- [x] Step 9: Cleanup (remove old bin/reins, update .gitignore, CLAUDE.md, CI workflows)

## Dependencies
None — standalone rewrite on main branch.
