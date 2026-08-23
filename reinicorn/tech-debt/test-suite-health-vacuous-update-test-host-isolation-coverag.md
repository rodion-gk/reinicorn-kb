---
type: debt
title: 'Test suite health: vacuous update test, host isolation, coverage gaps'
slug: test-suite-health-vacuous-update-test-host-isolation-coverag
lifecycle: active
status: draft
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
category: tests
severity: medium
remediation: planned
---

# Test suite health: vacuous update test, host isolation, coverage gaps

## Impact

From the 2026-07-27 quality review. The suite is strong overall (738 tests, 87.8% with a ratcheted floor), but four defects undermine trust in exactly the refactors the review recommends:

1. **One provably vacuous test.** `tests/test_update.py:52` (`test_update_skips_locally_modified_files`) modifies `AGENTS.md`, which `_collect_package_files` never manages (`update.py:70` explicitly pops it). The file survives because update never considers it; deleting the checksum-skip logic entirely would not fail this test, and `update.py:116-128` (the "Skipped (locally modified)" path) is uncovered.
2. **Host-state dependence.** `_check_skill_collisions` reads the real `Path.home()` (`~/.claude/skills/`, `~/.agents/skills/`, `~/.claude/settings.json`); only `tests/test_init_collisions.py` patches it, so every other full-`cmd_init` test executes against the developer's actual home directory and emits machine-dependent output. Two repo-setup helpers also omit `git init -b main` and depend on the host's `init.defaultBranch`.
3. **Scaffolding drift.** Six files hand-roll their own `_init_repo`/`_git` helpers in three divergent variants while `tests/conftest.py` keeps the canonical ones private; a `make_draft` helper is re-rolled in four places.
4. **The uncovered ~12% is systematically the agent-facing error surface** the repo's golden principles protect: `cli._dispatch_internal` (the wiring every installed git hook invokes — a rename in `_INTERNAL_COMMANDS` would break all hooks with a green suite), `init._prompt_kb_source` (the entire first-run onboarding menu), feedback's gh-present path, `kb remove`'s confirm/push fallbacks, and `kb status`'s sections (whose tests stub `run_git` wholesale and assert only `"kb" in out.lower()`). Also cohesion: `tests/test_public_identity.py` is 896 lines of which ~820 are an embedded bashlex shell parser with a GNU `env -S` emulator plus meta-tests — a test-support library wearing a test file's name.

## Remediation Plan

Ordered so later refactors land on trustworthy tests:

1. Fix the vacuous test: modify a managed `SKILL.md` instead of `AGENTS.md`, assert the "Skipped (locally modified)" + `rcorn update --diff` output; add tests for the deleted-file re-add prompt (both answers). Do this before any `update.py` refactor.
2. Add an autouse conftest fixture pointing `Path.home()`/`HOME` at `tmp_path` suite-wide; add `-b main` to (or delete) the divergent repo helpers.
3. Export `git_repo`/`make_draft` factories from conftest; delete the six local copies.
4. Move the workflow shell scanner to `tests/helpers/workflow_scan.py` with its meta-tests in `tests/test_workflow_scan_helper.py`; keep only identity assertions in `test_public_identity.py`. Separately consider shrinking the scanner to the cases real workflows use.
5. Close the coverage cluster: parametrized `main(["_pre-push"])`-style tests for every internal route plus the unknown-internal and `KeyboardInterrupt` paths; `_prompt_kb_source` driven by monkeypatched `input` sequences; one failing-subprocess test each for feedback's gh path, `kb remove` fallbacks, and init's multi-repo push fallback; rewrite `test_status.py` against a real kb repo asserting each section's actual lines.
