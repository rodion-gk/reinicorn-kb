---
type: spec
title: 'Golden principle enforcement: opengrep structural lints and TID251 seams'
slug: golden-principle-enforcement-semgrep-structural-lints-and-ti
lifecycle: active
status: in-review
created: 2026-08-08
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/13
---

# Golden principle enforcement: opengrep structural lints and TID251 seams

## Problem

A full mining pass over both repos' PR history (reinicorn #3–#42, reinicorn-kb #1–#12, all review threads) found five recurring bug classes that each drew the same reviewer correction on three or more independent PRs, plus one security-grade single occurrence:

1. **Under-strict matching in safety checks** — 5 instances: the N/A exemption regex matched `n/a/specs/x.md` (#30), the unconditional-exit detector missed `cmd; exit $?` (#37), the stderr-allowlist matched by basename (#29), `is_doc` matched excluded dirs anywhere in the absolute path (#30), bypass-actor reconciliation diffed by full value instead of identity keys (#18).
2. **"Cannot verify" collapsing into a conclusion before destructive actions** — `ls-remote` failure read as "branch gone" would have mass-archived plans (#30); a catch-all mapped every git failure to "not a git repo" (#29); kb#8's migration ran `submodule deinit -f` before its own safety check.
3. **One bad item aborting a whole batch** — an unguarded `plan.md` read aborting archival for every repo (#30); one non-UTF-8 hook aborting install for all hooks (#37); one malformed API entry crashing the command (#18).
4. **Guessed external behavior / naive parsing of structured formats** — invented GitHub role-id ladder (#12), assumed `gh api --paginate` output shape (#19), substring-matched `.gitmodules` (#13).
5. **"All paths do X" claims that held on one path** — the born-valid invariant held for 1 of 3 create paths until `frontmatter.render()` made the seam enforce it (#30); same shape in #13 and #42.
6. **Credential-bearing URLs printed raw** (#29, caught by CodeRabbit and CodeQL).

Mining the private-era history (mnbiehl/reins #1–#46) adds the decisive evidence: **these classes recur when the lesson lives only in a PR thread.** Private #9 shipped the direct ancestor of #30's archival bug — a `{"*"}` error sentinel whose membership check meant "archive every plan on any git failure," with a test asserting the sentinel's shape rather than the protected behavior — found, fixed, and then re-shipped as the same class eight months later. Private #23 centralized git env-stripping in `run_git`; #26 found a sibling call site bypassing it, reintroducing the same bug. Private #3's `grep -q "harness"` substring matching and `startswith("path")` key matching are the same under-strict-matching class the public era hit five more times. The one class that did *not* recur — `sanitize_branch` misuse — is the one that got an AST lint (#28). That asymmetry is this spec's whole argument.

A seventh recurring class comes from the private era's release sweep:

7. **Ship-lists that aren't verified allowlists** — the sdist had no allowlist and swept the entire private kb (~1545 files) into a published archive (#43); the wheel silently omitted attach assets so a pip install produced a broken `attach` (#4); `editor-hooks` was missing from the update-sync list so packaged hooks went permanently stale (#42).

Per this repo's promotion rule these become golden principles — but a principle without a lint is a convention, and `tests/test_git_error_surface.py` already declares the hand-rolled AST-walker suite at its limit: three walkers exist and the stated threshold is "do not add a fourth by hand — port all three to semgrep instead."

## Design Goals

- Each new principle's mechanical core has enforcement that fails CI, not a convention that has to be remembered. Where a principle also carries a process half (P18's evidence citations), the split is stated explicitly in the principle text — no principle leaves enforcement ownership implicit.
- Ruff-native (TID251) first; opengrep for what ruff cannot express; runtime pytest checks only where the rule needs imported objects (compiled regexes).
- The three existing AST walkers are ported to opengrep, honoring the threshold note (which names semgrep; opengrep is the rule-compatible fork — see phase 2); no fourth hand-rolled walker is ever added.
- Every rule's failure message cites the principle number and a fix (golden principle 4).
- Escape hatches are explicit, tagged, and greppable — never silent.

## Design

### New golden principles (added via `rcorn principle add` in phase 1)

- **P15 — Anchored, identity-precise matching in safety checks.** Allowlists, exemptions, and staleness checks match on anchored patterns and root-relative POSIX paths, keyed by identity — never substring, basename, loose word boundary, or full-value equality. Prefer non-regex forms (`str` methods, real parsers, `fullmatch`) wherever the check is expressible without a pattern; regex is for actual patterns.
- **P16 — Cannot-verify never collapses into a conclusion.** Any check gating a destructive or state-changing action distinguishes confirmed-yes / confirmed-no / cannot-verify; cannot-verify fails safe with a warning, and safety checks execute before the first destructive command.
- **P17 — Per-item fault isolation in batch loops.** Loops over independent items (repos, files, hooks, plans) wrap per-item I/O; one bad item is one skipped item plus a warning, never an aborted batch.
- **P18 — External surfaces get one wrapper and a real parser.** Structured formats are parsed with real parsers; each external CLI/API surface has a single owning module. Enforcement ownership is split: the wrapper confinement and real-parser rules are enforced by this spec (CI); the companion process rule — external IDs, enums, and output formats verified against live behavior or docs and cited in the PR — is owned by AGENTS.md and the review checklist, and is deliberately outside this spec's CI model (see Non-Goals).
- **P19 — Invariants are seams, not conventions.** An "always/never/all paths" claim routes through a single enforcing seam (with the bypass TID251-banned) or enumerates and tests every path.
- **P20 — Credentials never reach output.** URLs are printed only through a redacting display helper; userinfo is stripped before any console/log write.
- **P21 — Ship-lists are allowlists, verified against the built artifact.** Anything that enumerates what gets packaged, synced, or installed (sdist/wheel contents, `rcorn update` asset categories) is an authoritative allowlist — never "whatever git tracks" or an additive include — and CI builds the real artifact and asserts on its contents (the #43 pattern: private/dev trees absent, required assets present). The update-sync category list is asserted complete against the canonical asset-source enumeration.

### Phase 1 — no new tooling (TID251 + trivial grep tests)

- TID251 bans (pyproject.toml), each with a principle-citing msg:
  - `reinicorn.frontmatter.dumps` outside `frontmatter.py` — creation goes through the validating `render()` seam (P19).
  - `shutil.rmtree` and `shutil.copy2` outside the owning destructive seam module, with the seam exempted via per-file-ignores — the same ban-plus-exemption mechanism the existing `sanitize_branch` confinement uses (P16). Method-call shapes on locals (`p.unlink()`, `p.rename()` out of `active/`) are **not** expressible in TID251 and are explicitly deferred to the phase-3 opengrep rule. Exact ban list fixed during implementation against real call sites.
  - The raw remote-URL getter outside `git.py`; `git.py` gains `display_url()` that strips userinfo, with a `user:token@host` fixture test (P20).
- Literal-confinement checks in the P2 grep idiom (a small pytest, not an AST walker): `.gitmodules` may only appear in `git.py`; `--paginate` only in `github.py`, whose `gh_api_paginate()` owns page flattening (P18).
- Runtime anchoring test: every module-level `re.Pattern` in gate modules (`linter/`, `hooks_health.py`, `commands/internal/spec_gate.py`, `commands/internal/post_merge.py`) must start `\A` and end `\Z`. `^`/`$` are **not** accepted as anchors: under `re.MULTILINE` they match line boundaries, so `"evil\nabc"` passes a `^abc$` check — the exact bypass shape P15 exists to kill, live in the checker itself if we allowed them. Escape hatch is a module-level `UNANCHORED_OK: dict[str, str]` mapping pattern name → reason (P15). This imports compiled objects, so pytest is the right layer, not opengrep.

- **P21** (no new tooling — pytest builds the real artifacts): extend the existing sdist-content regression test (inherited from private-era #43) to also build the wheel and assert required runtime assets are present (the #4 failure mode), and add a completeness test asserting the `rcorn update` sync-category list matches the canonical asset-source enumeration (the #42 failure mode).

### Phase 1b — regex corpus cleanup (audit results, 2026-08-08)

A full audit of the 23 `re.` sites in `src/reinicorn` found two real defects, two improvements, and the seed list for `UNANCHORED_OK`:

- **Defect — `validation.py::_SAFE_NAME_RE` accepts a trailing newline.** `$` without `re.MULTILINE` still matches before a final `\n`, so `"name\n"` passes boundary validation. Convert to `re.fullmatch` on an unanchored pattern (no `$` reliance). This is the P15 bug class live in the validation boundary today.
- **Defect — user-configured `ticket_pattern` is compiled unguarded.** `plan.py` and `internal/post_checkout.py` pass the `reinicorn.ticketpattern` git-config value straight to `re.search`; a malformed pattern raises `re.error` and crashes `plan create` and the post-checkout hook. Validate at the boundary: catch `re.error`, warn with the offending value, fall back to the default pattern.
- **Improvement — `cross_links.py` scheme check** (`^(https?://|mailto:|ftp://|#)`) becomes `target.startswith(("https://", "http://", "mailto:", "ftp://", "#"))` — the non-regex form P15 prefers.
- **Improvement — `doc_create.py` principle counter** (`re.findall(r'^\d+\.', content, re.MULTILINE)`) counts any numbered list line anywhere in the doc, so a numbered list inside a principle body inflates the next principle number. Tighten to the actual principle-heading shape (`^\d+\. \*\*`).
- **`\A`/`\Z` conversions with per-pattern scope tests** (satisfies the phase-1 anchoring test). Each conversion states its intended match scope and lands with a regression test pinning that scope — it is not asserted "behavior-safe" in bulk. Intended scopes: `hooks_health._UNCONDITIONAL_EXIT` — one stripped shell statement, whole-input (test: statement with trailing newline must not match); `spec_refs.SPEC_PLACEHOLDER_RE` — one whole frontmatter value (test: `"[x]\nrest"` must not match); `spec_refs._NOT_APPLICABLE_RE` — start of a stripped value (deliberately open-ended after the `n/a` token; the anchor conversion touches only the `$` alternative inside `(?:$|\s)` → `(?:\Z|\s)`, test: `"N/A — reason"` still matches, `"n/a/specs/x.md"` still doesn't); `review_cleanup._REF_RE` — a whole ref name (test: ref with embedded newline must not match, which `\Z` tightens over `$`).
- **`UNANCHORED_OK` seed entries** (search-style by design, each with its reason): `hooks_health._REINS_DELEGATION` (staleness scan over whole hook text), `spec_refs.REF_RE` (prose reference finder), `validation._SCP_LIKE_RE` (transport-prefix check), `plan_structure`'s heading matcher (deliberate per-line `MULTILINE`).
- Clean as-is: the `git.py` URL parsers (`.strip()`ed input, not gate modules), slugify/template substitutions, `feedback.py`'s HTML-comment stripper, `plan.py._EMPTY_RETRO_LINE`, `cross_links`' markdown-link finder (accepted lint-grade imprecision).

### Phase 2 — opengrep adoption and walker port

- The engine is **opengrep** (the LF-backed LGPL-2.1 fork of Semgrep CE), not semgrep: same rule YAML and JSON/SARIF output, no telemetry, no commercial tier, and deterministic output — which the phase-3 ratcheting baseline depends on. Every rule in this spec is plain syntactic matching, so nothing here uses semgrep-only features; the walker docstrings' "port to semgrep" threshold is honored by the compatible fork.
- opengrep is not on PyPI — it ships as self-contained binaries. CI installs a pinned release with checksum verification; `tests/run-all.sh` runs `opengrep` from PATH and fails with a pointer to the install script when missing. Fallback if binary install proves problematic on some platform: a tiny docker image wrapping the same pinned binary.
- Rules live in `linters/opengrep/`; runs in `tests/run-all.sh` and the CI lint job beside ruff.
- Port the three existing walkers 1:1 (doc-type key comparisons, branch→path ownership, git/gh stderr seam), then delete the hand-rolled walkers.
- Port is behavior-preserving, demonstrated by positive **and** negative fixtures under `linters/opengrep/tests/` covering the boundary cases the walkers actually pin, not just one seeded violation each: direct `result.stderr` access, the `getattr(r, "stderr")` evasion, doc-type key comparisons both bare and inside tuple/list/set literals, an allowlist entry written as a basename (must be flagged) vs root-relative path (must pass), and allowed usage inside each seam module (must not be flagged). The hand-written walkers are deleted only after every fixture passes under the opengrep rules.

### Phase 3 — new opengrep rules

- **P15:** no `.name` equality/membership comparisons in enforcement or allowlist code paths.
- **P16:** destructive filesystem calls confined to the seam module (widens the phase-1 TID251 slice to call shapes ruff cannot express).
- **P17:** per-item I/O inside a `for` body must be wrapped by a `try` inside that loop, **and** the handler must call the shared `skip_item(item, exc)` helper (which emits the warning and continues). The match surface is structural, not a hand-kept function inventory — a listed-names surface would recreate the P21 incomplete-allowlist class (new per-item I/O silently bypassing the lint). The rule flags two shapes: (a) any call into a seam module (`git.*`, `github.*`, `frontmatter.*` — the P18 wrappers, matched by module with a wildcard, so new seam exports are covered automatically), and (b) any call to a pathlib I/O method (`read_text`, `read_bytes`, `write_text`, `open`, `iterdir`, `glob`, `stat`, `unlink`, `rename`, …) — a set bounded by the stdlib, not by this codebase, so it does not need maintenance as code grows. A `try` alone is not compliance: `except Exception: pass` and handlers that `return` or bare-`raise` abort or silence the batch and are flagged by the same rule. Probe-style loops opt out with a tagged `# per-item-ok:` comment.
- Rules adopt with a ratcheting baseline: `linters/opengrep/baseline.json`, versioned in-repo, mapping rule id → sorted list of `path:fingerprint` entries for pre-existing violations. CI compares the branch's baseline against main's and fails if any entry was added; removals are the only permitted change (same policy as the coverage floor). A suppressed finding therefore cannot be introduced by editing the baseline in the same PR that introduces the violation.

### Phase 4 — tri-state verdicts on destructive paths

- `reinicorn/verify.py` gains `Verdict` (`CONFIRMED` / `ABSENT` / `UNKNOWN`). Destructive seams take `Verdict`, not `bool`; pyright then enforces production of a verdict at every caller. Boolean verification helpers on destructive paths are converted, then TID251-banned by name.
- The typed shape **is** the ordering guarantee: the seam is callable only with a `Verdict`, and only verification helpers produce one, so destroy-before-verify is unrepresentable at type-check time — no separate ordering lint is needed once phases 1/3 have confined the raw destructive calls to the seam.
- `UNKNOWN` contract: the seam owns the warning, and the fail-safe declaration is a typed parameter, not a convention. Each destructive seam takes `on_unknown: RaisePolicy | SkipPolicy` (a two-variant union in `verify.py`, defaulting to `RaisePolicy`; `SkipPolicy` carries the item label for the warning). With `SkipPolicy` — the batch case — the seam emits the warning naming the item and the reason verification failed, and the item is skipped (which is also the P17-compliant shape). With `RaisePolicy`, or when the caller states nothing, it raises. Pyright thus enforces the declaration the same way it enforces `Verdict` production at every caller. Both paths carry named tests (`test_unknown_verdict_warns_and_skips`, `test_unknown_verdict_raises_without_handler`).
- Existing tri-state behavior (#30's "a plan with no `branch:` is never archived") is re-expressed through `Verdict` so the pattern has one home.

## Non-Goals

- The process half of the findings (PR-body conventions, empirical-verification citations, review-response style, and the tests-assert-protected-behavior-not-intermediate-signals rule from private #9 / #35, public #22, kb#11) — that is AGENTS.md / review-checklist material, handled separately from this spec. The test rule in particular has no mechanical core (a lint cannot know which assertion is the proxy), so per this spec's own design goals it is delegated, not half-enforced.
- CodeRabbit configuration or learnings — per #40, shared guidelines are fixed at the source doc, not per-reviewer.
- The managed pre-push principle-enforcement gate (existing idea: reinicorn-managed-git-hooks-for-golden-principle-enforcement) — these rules run in CI and the test suite; hook delivery is that idea's scope.
- Retrofitting every existing violation immediately — phase 3 rules land with a ratcheting baseline, not a big-bang cleanup.
