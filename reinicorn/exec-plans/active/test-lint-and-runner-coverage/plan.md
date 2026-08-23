---
type: plan
title: 'Execution Plan: test/lint-and-runner-coverage'
slug: test-lint-and-runner-coverage
lifecycle: active
status: planning
created: 2026-07-25
author: Michael Biehl
branch: test/lint-and-runner-coverage
ticket: N/A
spec: specs/test-coverage-reports.md
---

# Execution Plan: test/lint-and-runner-coverage

## Goal
Close the two worst coverage gaps found once measurement landed (P1 and P2 of
the coverage analysis): `commands/lint.py` at 0% and `linter/runner.py` at
45%. Together ~60 statements, and both are live production code.

Stacked on `feat/coverage-reports` (PR #20), which adds the coverage tooling
this branch is measured by.

## Acceptance Criteria
- [ ] `src/reinicorn/commands/lint.py` at 100%
- [ ] `src/reinicorn/linter/runner.py` at >=95%, including the external `.sh`
      rule loop (lines 76-117) which currently has zero coverage
- [ ] Total coverage rises from 85.69%; `fail_under` raised to the measured
      floor, rounded down
- [ ] No production code changed — tests and config only
- [ ] `pytest`, `ruff`, `pyright` green

## Approach
- **Why lint.py reads 0%:** nothing imports it. It is reachable only through
  `cli._DISPATCH[("kb","lint")]`, which lazy-imports via
  `importlib.import_module`, and no test dispatches `kb lint`. Not a
  subprocess-coverage artifact — the one test that shells out to the CLI only
  runs `--help` and bad-verb paths.
- **Why runner.py sits at 45%:** the `kb_repo` fixture writes
  `linters/.lint-config.json` but creates no `linters/rules/` directory, so
  `runner.py:75`'s `is_dir()` guard is always False and the entire 42-line
  external-`.sh` loop is dead to the suite. All three configured rules also
  pass against the fixture, so every failure and summary path is unreached.
  The real repo ships `linters/rules/kb/provenance.sh` and
  `scripts/shellcheck.sh`, both enabled — this loop runs in CI for real.
- Do **not** modify the `kb_repo` fixture: ~10 test files depend on it. Write
  per-test lint configs and rule scripts into the test's own tree.
- Trigger rule failures with real inputs (a broken markdown link in
  `AGENTS.md` makes `kb/cross-links` produce diagnostics) rather than mocking
  rule classes. The point is to cover the runner's handling of a failing
  rule, which mocks would fake away.
- External `.sh` rules are tiny generated scripts in the test tree; no real
  linter binaries, no network.

## Tasks
- [ ] `tests/commands/test_lint.py` (new): outside-repo guard returning 1,
      and a passing run through `kb_repo` returning 0
- [ ] Cover the `kb lint` dispatch entry so the lazy import is exercised
- [ ] Extend `tests/linter/test_runner.py`:
  - [ ] disabled builtin rule -> skipped counter (41-42)
  - [ ] `max_days_stale` on a rule that takes no kwargs -> `TypeError`
        fallback (52-53)
  - [ ] failing builtin at `error` and at `warning` severity (61-70)
  - [ ] error and warning summary blocks (130-139)
  - [ ] external `.sh` passing, failing at both severities (76-117)
  - [ ] `.sh` whose name collides with a builtin -> skipped (80-81)
  - [ ] `.sh` present but unconfigured / disabled (84-86)
  - [ ] non-executable `.sh` -> `PermissionError` arm (96-101)
- [ ] Measure, then raise `fail_under` in `pyproject.toml` to the new floor
      and update the number in `CONTRIBUTING.md`
- [ ] Verify: full suite green, both target modules at goal

## Dependencies
Stacked on `feat/coverage-reports` (PR #20). PR targets that branch; GitHub
retargets to `main` when #20 merges. No overlap with the other active plans.
