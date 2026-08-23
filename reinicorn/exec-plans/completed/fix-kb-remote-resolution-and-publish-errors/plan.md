---
type: plan
title: 'Execution Plan: fix/kb-remote-resolution-and-publish-errors'
slug: fix-kb-remote-resolution-and-publish-errors
lifecycle: done
status: complete
created: 2026-07-27
author: Michael Biehl
origin: ai-assisted
human_validated: false
branch: fix/kb-remote-resolution-and-publish-errors
ticket: N/A
spec: N/A (fixes two ideas; must survive [[remove-the-kb-submodule]])
---

# Execution Plan: fix/kb-remote-resolution-and-publish-errors

## Goal

Fix two defects found on 2026-07-27:

1. **[[worktree-kb-init-uses-the-gitmodules-https-url-and-drops-the]]** — a kb
   created by the post-checkout hook takes its remote from `.gitmodules`
   (HTTPS), discarding the main checkout's SSH override. With `gh` set to
   `git_protocol: ssh` for `github.com` and no credential helper, every
   `rcorn kb publish` from that worktree fails with
   `could not read Username for 'https://github.com'`.
2. **[[rcorn-kb-publish-reports-any-post-retry-push-failure-as-conf]]** —
   `push_main_with_retry`'s callers report every non-zero push as "kb has
   conflicting changes", masking the auth failure above and suggesting a retry
   that loops forever.

Then, on review, the class behind (2) rather than the instance: six modules
each invented their own git-failure format, and two `contextlib.suppress`
blocks discarded failures entirely — including the one that installed the
broken remote in (1) with no signal at all.

## Acceptance Criteria

- [x] A kb clone created by the post-checkout hook inherits `remote.origin.url`
      from the main checkout's kb when one exists, so local overrides (SSH,
      mirror, internal host) follow into every worktree.
- [x] With no kb to inherit from, the remote is resolved from
      `REINICORN_KB_REMOTE` (then `.gitmodules` as a transitional fallback) and
      adapted to the user's configured git protocol for `github.com`.
- [x] Protocol detection reads the **host-scoped** `gh` setting; the global
      `gh config get git_protocol` is not authoritative.
- [x] HTTPS is never rewritten to SSH without positive evidence of an SSH
      preference; an SSH URL is never rewritten to HTTPS.
- [x] `rcorn kb publish` and the review lane classify a failed push into
      auth / non-fast-forward / protected-branch / unknown, and print git's
      stderr verbatim for `unknown` rather than guessing.
- [x] The auth message names the remote URL and its protocol and offers a
      concrete `rcorn kb git remote set-url` next step.
- [x] No resolution logic is submodule-specific: everything lives behind one
      seam that [[remove-the-kb-submodule]] can keep unchanged.
- [x] Exactly one module converts a git failure into text; every caller
      describes what it was doing and delegates the wording.
- [x] `run_git(check=True)` raises a `GitError` that carries cmd, exit code and
      stderr, without breaking the documented `CalledProcessError` contract.
- [x] Every `check=False` call site is triaged probe vs operation; probes stay
      as they are, operations are surfaced.
- [x] Neither `contextlib.suppress(Exception)` block discards a failure any
      more, and neither hook can fail the user's checkout or merge.
- [x] A structural test fails the build if a second module starts formatting
      git's stderr on its own.
- [x] Suite stays green and total coverage stays at or above the 87% floor.

## Approach

**Defect 1 — one resolution seam, `kb_remote.resolve_kb_remote_url(root)`.**
New module `src/reinicorn/kb_remote.py`. Resolution order:

1. `remote.origin.url` of the **main checkout's** kb clone (found by walking up
   from `git rev-parse --git-common-dir`). This is the user's own working
   configuration and is protocol-agnostic — it is what fixes the reported bug.
2. `REINICORN_KB_REMOTE` from `.reinicorn-config`.
3. The `.gitmodules` kb `url =` — transitional only, deleted with the submodule.

Whatever comes from (2) or (3) then goes through `adapt_url_to_git_protocol`,
which rewrites `https://github.com/o/r(.git)` to `git@github.com:o/r.git`
**only** when `gh config get -h github.com git_protocol` reports `ssh`. The
rewrite is deliberately one-way: `gh`'s default for an unconfigured host is
`https`, so treating that as evidence would break SSH-only users.

`post_checkout` resolves the URL *before* creating the kb and applies it with
`git remote set-url origin` afterwards. Post-[[remove-the-kb-submodule]] the
same call feeds `git clone <url>` directly — step 1 and step 2 both survive
verbatim; only step 3 is deleted with `.gitmodules`.

**Defect 2 — classify, then report.** `classify_push_failure(stderr)` in
`kb.py` returns `auth` / `non-fast-forward` / `protected` / `unknown`;
`report_push_failure(push, kb_dir)` prints the matching error and next step.
`commands/publish.py` and `commands/review.py::_push_kb_main` both call it, so
the two lanes cannot drift apart again.

**Defect 2, generalized — one seam, mechanically enforced.** The push
classifier moves into `git.py` and becomes the general one:

- `GitError(subprocess.CalledProcessError)`, raised by `run_git(check=True)`.
  Subclassing keeps the error contract documented at the top of `review.py`
  intact for every existing handler, and inherits cmd/returncode/stderr.
- `classify_failure(stderr)` / `classify_result(result_or_error)` →
  `auth` | `non-fast-forward` | `protected` | `unknown`.
- `explain_failure(action, failure, detail=…)` → message lines, and
  `report_failure(...)` → the same through `console`. `action` is what the
  caller was doing ("push kb main"); every line of git's stderr is reproduced
  under a `git:` prefix, and `unknown` gets no invented cause at all.
- Headlines stay domain-free. "kb has conflicting changes. Resolve any
  conflicts in kb/" is a *caller* detail line from `kb.py`, so git.py never
  learns kb vocabulary.

The six ad-hoc formats (`commands/review.py:79`, `submodule.py:124`,
`commands/kb_manage.py:118`, `commands/sync.py:51`, `review.py:177`,
`commands/init.py:280`) all become callers. `SubmoduleError` loses its
`stderr` attribute; the whole diagnosis lives in the message.

Enforcement: `tests/test_git_error_surface.py` walks the AST and fails if any
module outside `git.py` (git) or `github.py` (gh — already single-seamed)
reads a subprocess `.stderr`, including via `getattr`. Approved modules are
matched by root-relative path, never basename, so a future `commands/git.py`
cannot borrow the exemption; the rule is exercised against a synthetic tree to
prove it. Ruff cannot express this: TID251 bans imported names, and
`result.stderr` is attribute access on a local. This is the third hand-rolled
AST check, which is the threshold `test_source_of_truth.py` names — the file
says so and says to port all three to semgrep rather than add a fourth.

**Review round (PR #29).** Four findings, all valid:

1. CodeQL flagged `"github.com" in args`; the assertion now pins the whole
   argv, which is what its docstring claimed to guarantee anyway.
2. `apply_kb_remote_url` reported nothing when `git remote set-url` failed and
   its only caller discarded the result — the branch's own bug class. It now
   returns `updated` / `unchanged` / `failed` (a bool conflated "already
   right" with "could not set it"), reports through the seam, and
   `post_checkout` says what a failure means without failing the checkout.
3. `except (GitError, FileNotFoundError)` in `hooks_install` called every
   nonzero exit "not inside a git repository". `classify_failure` gains a
   `no-repo` class; only that earns the sentence, everything else goes to the
   seam, and a missing git binary is named as such.
4. `report_push_failure` gains `action=` and `warn=`, so the kb scope-removal
   push gets the kb diagnosis (remote, protocol, classification-specific next
   step) instead of the generic one, without turning a non-fatal step fatal.

## Tasks

- [x] Red: tests for `resolve_kb_remote_url` (inherit, config, `.gitmodules`,
      precedence) and `adapt_url_to_git_protocol` (ssh preference, https
      preference, non-GitHub host, `gh` missing).
- [x] Green: add `src/reinicorn/kb_remote.py`.
- [x] Red: test that `cmd_post_checkout` in a worktree gives the new kb the main
      checkout's origin URL, not the `.gitmodules` one.
- [x] Green: wire `resolve_kb_remote_url` into
      `commands/internal/post_checkout.py`.
- [x] Red: tests for `classify_push_failure` over real git stderr strings and
      for `report_push_failure` output per class.
- [x] Green: add both to `kb.py`; call from `publish.py` and `review.py`.
- [x] Red: tests for `GitError`, `classify_failure`/`classify_result`,
      `url_protocol`, `explain_failure`, `report_failure`.
- [x] Green: add the seam to `git.py`; `run_git` raises `GitError`.
- [x] Migrate the six ad-hoc formats and re-point their tests at the new
      (stronger) assertions.
- [x] Triage all 48 `run_git(..., check=False)` sites: 26 probes left alone,
      7 operations routed through the seam, 2 returned to a surfacing caller,
      13 deliberately best-effort and documented.
- [x] Unwrap `post_checkout`'s `suppress(Exception)` into `_init_kb`, which
      reports and returns rather than raising into the user's checkout.
- [x] Unwrap `post_merge`'s `suppress(Exception)` (it wraps `cmd_plan_complete`,
      not a git call) so a failed archive is reported, not silent.
- [x] Surface `commit_kb`'s failed commit — it returned False identically to
      "nothing to commit", so unsaved work looked like a no-op.
- [x] Red-green the enforcement test; add `tests/test_git_error_surface.py`.
- [x] Verify: pytest, ruff, pyright, `bash tests/run-all.sh`, coverage >= 87%.

## Dependencies

- [[remove-the-kb-submodule]] (kb PR #8) merges before this lands. It deletes
  `.gitmodules`, `stage_kb_pointer()`, and `.gitmodules` parsing in
  `get_kb_dir()`. Nothing here adds new `.gitmodules` coupling; the only
  `.gitmodules` read is an explicitly-labelled fallback step that #8 removes.
- Concurrent work on `src/reinicorn/frontmatter.py` / kb doc frontmatter — not
  touched here.
- `ensure_kb_on_main()`'s unchecked `git checkout main` (`kb.py`) is an
  unsurfaced operation left as-is on purpose: [[remove-the-kb-submodule]] §5
  rewrites that function outright, and fixing it here would collide.
- Follow-up, not done here: port the three hand-rolled AST checks
  (`test_source_of_truth.py` ×2, `test_git_error_surface.py`) to semgrep or a
  flake8 plugin. `linters/structural-tests/` already exists with a CI workflow
  wired to it and is currently empty — a plausible home.
