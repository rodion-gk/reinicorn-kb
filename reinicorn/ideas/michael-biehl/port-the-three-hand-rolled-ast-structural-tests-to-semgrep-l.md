---
type: idea
title: Port the three hand-rolled AST structural tests to semgrep; linters/structural-t
slug: port-the-three-hand-rolled-ast-structural-tests-to-semgrep-l
lifecycle: active
status: new
created: 2026-07-27
author: Michael Biehl
---

# Port the three hand-rolled AST structural tests to semgrep; linters/structural-t

## Description

Port the three hand-rolled AST structural tests to semgrep; linters/structural-tests is already CI-wired and empty

## Notes

Raised 2026-07-27 while landing the git-error seam (reinicorn#29).

### The threshold was already written down

`tests/test_source_of_truth.py`'s docstring says:

> Prefer ruff-native enforcement when the rule is expressible there (e.g. the
> sanitize_branch confinement lives in pyproject.toml as a TID251 banned-api).
> Hand-rolled AST checks like this one are a fallback for rules ruff cannot
> express. If we accumulate several more of them (here or in
> test_output_conventions.py), stop and build a proper flake8 plugin (or
> semgrep rules) instead of growing this file.

`tests/test_git_error_surface.py` is the third. Its own docstring now says not
to add a fourth by hand.

### Why ruff can't take these

Checked before hand-rolling: TID251 bans *imported names*, which is why
`reinicorn.git.sanitize_branch` works. The git-error rule is attribute access
on a local (`result.stderr`), and no ruff rule matches that shape.

### The find that makes this cheap

`linters/structural-tests/` **already exists, is wired to the `Lint
Architecture` CI workflow, and contains only `.gitkeep`.**
`.github/workflows/lint-architecture.yml:25-63` finds every file there, runs
it, counts violations, and skips cleanly when the directory is empty. So the
CI half is done — this is only about writing rules and deleting three Python
files.

Candidates to port:

| current | rule |
| --- | --- |
| `tests/test_source_of_truth.py` | doc-type strings must come from `doc_types.REGISTRY` |
| `tests/test_output_conventions.py` | no `file=sys.stderr` in `commands/` |
| `tests/test_git_error_surface.py` | only `git.py`/`github.py` may read a subprocess `.stderr` |

### Open question

Semgrep is a new dependency and a second rule language for three rules in a
small codebase. Weigh that against three bespoke AST walkers that each need
maintaining. A flake8 plugin keeps the toolchain Python-only but is more work
to write. Decide before a fourth walker appears.

### Added 2026-08-16: golden principle 15 joins the rule backlog

"No stringly-typed code" (golden principle 15) can only be enforced
generally by this work — pyright's `reportUnnecessaryComparison` covers
enum-vs-string-literal comparisons and nothing else; the full smell
(modes/kinds/states passed as bare strings, data encoded in delimited
strings) needs semantic rules. When the opengrep port lands, add rules for
principle 15 alongside the three ported walkers.

### Decided 2026-08-14: opengrep, not semgrep

The port happens via the golden-principle-enforcement spec (in review, kb#13),
and the engine is opengrep — the LF-backed LGPL-2.1 fork of Semgrep CE. Same
rule YAML, no telemetry, no commercial tier; all three rules are plain
syntactic matching, so nothing depends on semgrep-only features. One
integration difference: opengrep is not on PyPI, so CI pins a checksum-verified
release binary instead of a pip dev dependency (docker-wrapped binary as
fallback if installs prove problematic).
