---
type: idea
title: Reinicorn-managed git hooks for golden-principle enforcement
slug: reinicorn-managed-git-hooks-for-golden-principle-enforcement
lifecycle: active
status: new
created: 2026-07-22
author: Michael Biehl
---

# Reinicorn-managed git hooks for golden-principle enforcement

## Description

Golden principles need enforcement. Sometimes that's a lint rule; sometimes
it's a git hook. Today Reinicorn ships only a fixed, hardcoded trio of git
hooks (post-checkout, post-merge, pre-push — all kb-submodule logic) and has no
general capability to manage git hooks on the user's behalf. This idea: give
Reinicorn a first-class **git-hook management** subsystem so a golden principle
can declare an enforcement mechanism (hook or lint) and Reinicorn wires it into
the user's repo.

**Key insight (resolves the "is this the product's job?" tension):** Reinicorn
manages the hook *lifecycle* — install, chain/compose, dedupe, uninstall — while
the enforcement *command* is declared by the principle, not baked into the
product. Reinicorn is not a test runner; it wires `uv run pytest tests/` into
pre-push because a principle asked it to. Hook lifecycle is a legitimate product
concern; the dev command stays out of the product surface.

This generalizes the meta-principle already in `golden-principles.md` (step 4:
"encode it in a linter or structural test") to: **enforcement is a lint rule OR
a managed hook**, and each principle declares which.

## Dependency

Waits for the frontmatter schema
([[unified-kb-doc-frontmatter-schema]], currently in review). Enforcement is
declared in a principle's **frontmatter**, so this should design against a
settled schema, not a moving one. Sketch:

```yaml
type: principle
id: P14
enforcement:
  kind: hook          # or: lint
  event: pre-push     # pre-commit | commit-msg | pre-push | ...
  command: uv run pytest tests/
  scope: this-repo    # vs universal (like kb-submodule push)
```

## First consumer (dogfood)

Golden principle #14 ("Verify against the full gate, not a subset") is the first
thing this feature should enforce: a Reinicorn-managed pre-push hook running the
CI gate on the Reinicorn repo itself. Until this exists, #14 is enforced only by
CI required checks.

## Prior art to study (research task)

Existing git-hook managers — capture how each handles install, config/declaration
format, multiple hooks per event, composition with pre-existing hooks,
cross-platform, monorepo/worktree, and escape hatches:

- **Husky** (Node ecosystem)
- **Lefthook** (Go, language-agnostic)
- **pre-commit** (Python, the framework)
- **simple-git-hooks**
- **Overcommit** (Ruby)
- **Git native** `core.hooksPath`

## Open design questions (for the eventual spec)

- Declaration format and where it lives (principle frontmatter vs a central
  config).
- Hook composition/ordering: how declared hooks compose with Reinicorn's
  built-in kb hooks via the existing chain-below-MARKER mechanism.
- Scope: per-repo-scope (kb is already repo-scoped) vs universal enforcement.
- Escape hatches: `--no-verify`, opt-out, CI-only mode.
- Uninstall / drift detection when a principle's enforcement changes.
- Cross-platform + git worktree behavior (ties into recent worktree-aware work).

## Prior art: git-hook managers (research 2026-07-22)

Research feeding the eventual spec. Reinicorn already installs a fixed trio
(`post-checkout`, `post-merge`, `pre-push`) by **appending below a marker line**
into `.git/hooks/<hook>` (`src/reinicorn/commands/hooks_install.py`,
`MARKER = "# --- reinicorn hooks below ---"`).

### 0. Git native (baseline)

- Default hooks dir is `$GIT_DIR/hooks`; each event is **one executable file** —
  no native support for multiple scripts per event.
  [githooks](https://git-scm.com/docs/githooks)
- `core.hooksPath` (Git ≥2.9) redirects the hooks dir, per-repo or global.
- A hook file without the exec bit is silently ignored.
- `init.templateDir`/`$GIT_TEMPLATE_DIR` seeds `.git/hooks` on init/clone —
  Overcommit repurposes this for auto-install-on-clone.
- **Worktrees:** linked worktrees share `.git/hooks` via `$GIT_COMMON_DIR`; no
  native per-worktree hooks. `extensions.worktreeConfig` (off by default) allows
  per-worktree `core.hooksPath` but nothing sets it automatically.

### 1. Husky (Node)

- **Install:** v9 repoints `core.hooksPath` → `.husky/_`; hooks are plain files
  in `.husky/`. `npm install` fires `prepare`. (`husky install` removed in v9.)
- **Declaration:** one shell file per hook name, no config schema.
- **Multi-command:** yes, sequential, fail on first non-zero; no parallelism.
- **Foreign hooks:** clobbers by repointing `core.hooksPath` — its most
  contentious property (issues
  [#120](https://github.com/typicode/husky/issues/120),
  [#171](https://github.com/typicode/husky/issues/171),
  [#323](https://github.com/typicode/husky/issues/323),
  [#558](https://github.com/typicode/husky/issues/558)). Broke git-lfs for
  Mastodon → considered switching to Lefthook
  ([mastodon#24884](https://github.com/mastodon/mastodon/issues/24884)).
- **Cross-platform:** forces `sh` even if a shebang is set.
- **Escape hatches:** `HUSKY=0`, `-n`/`--no-verify`.
- **Uninstall/drift:** none surfaced; a stray `core.hooksPath` is a known trap.

### 2. Lefthook (Go)

- **Install:** writes real wrappers into `.git/hooks/<name>` directly (no
  `core.hooksPath`). `lefthook install`. Standalone binary.
  [repo](https://github.com/evilmartians/lefthook)
- **Declaration:** single YAML/TOML/JSON config (many filenames), *event →
  command name → run*; gitignored `lefthook-local.yml` override.
- **Multi-command:** native, **parallel by default**; `piped: true` for
  sequential stop-on-failure.
- **Foreign hooks:** historically silent-broke on a stale foreign
  `core.hooksPath`; now detects/warns loudly
  ([#1248](https://github.com/evilmartians/lefthook/issues/1248)). No auto
  Husky→Lefthook migration.
- **Cross-platform:** native binary, no runtime dep, fast on Windows.
- **Escape hatches:** `LEFTHOOK=0`.
- **Uninstall/drift:** `lefthook uninstall`; `validate`/`dump` for inspection.

### 3. pre-commit (Python)

- **Install:** writes `.git/hooks/<name>` directly; `pre-commit install` per hook
  type (only `pre-commit` by default). [pre-commit.com](https://pre-commit.com/)
- **Declaration:** `.pre-commit-config.yaml`, `repos:` pinning provider + `rev` +
  selected `hooks:`.
- **Multi-command:** yes, sequential in declared order; `stages:` incl. a
  never-auto `manual` stage.
- **Foreign hooks:** the one with a deliberate **migration mode** — runs *both*
  your existing hook and pre-commit by default; `-f` to overwrite. Closest
  analog to Reinicorn's append-below-marker.
- **Cross-platform:** each hook runs in an isolated per-language env (its
  signature trait and main install-time cost).
- **Monorepo:** not first-class, root-only
  ([#466](https://github.com/pre-commit/pre-commit/issues/466)).
- **Escape hatches:** `SKIP=<ids>`, `--no-verify`, `manual` stage.
- **Uninstall/drift:** `pre-commit uninstall` restores exact pre-install state —
  best uninstall story surveyed.

### 4. simple-git-hooks

- **Install:** writes `.git/hooks/` directly; no auto re-install — manual re-run
  after every config change.
  [repo](https://github.com/toplenboren/simple-git-hooks)
- **Declaration:** `package.json` key or `.simple-git-hooks.json`.
- **Multi-command:** **unsupported by design** — one command per hook; README
  redirects heavier needs to Lefthook/Husky/pre-commit.
- **Foreign hooks:** overwrites target each run; `preserveUnused` only protects
  other *declared* hooks.
- **Uninstall/drift:** manual `uninstall.js`; none.

### 5. Overcommit (Ruby)

- **Install:** the outlier — uses `GIT_TEMPLATE_DIR` to seed hooks (auto-seeds
  every future clone system-wide). `overcommit --install`.
  [repo](https://github.com/sds/overcommit)
- **Declaration:** `.overcommit.yml` grouped by event; gitignored
  `.local-overcommit.yml` override.
- **Multi-command:** native, **parallel by default**, `parallelize: false` to
  serialize; `skip_if` conditional.
- **Foreign hooks:** **backs up** existing hooks on install, restored on
  uninstall — cleanest, with pre-commit.
- **Cross-platform:** Ruby-runtime dependent (heavier).
- **Escape hatches:** `SKIP=`, `ONLY=`, `required: true` (un-skippable) locks.
- **Uninstall/drift:** `overcommit --uninstall` restores exact backup.
- **Security:** README warns outright that auto-running repo-declared hooks is a
  supply-chain risk.

### Comparison table

| Dimension | Husky | Lefthook | pre-commit | simple-git-hooks | Overcommit | Git native |
|---|---|---|---|---|---|---|
| **Install mechanism** | `core.hooksPath` → `.husky/_` | writes `.git/hooks/*` | writes `.git/hooks/*` | writes `.git/hooks/*` | `GIT_TEMPLATE_DIR` seed | n/a |
| **One-time command** | `npm install` | `lefthook install` | `pre-commit install` (per type) | `npx simple-git-hooks` (manual) | `overcommit --install` | — |
| **Config format** | shell files in `.husky/` | `lefthook.yml` (+local) | `.pre-commit-config.yaml` | `package.json` key | `.overcommit.yml` (+local) | none |
| **Multi-command/event** | yes, sequential | yes, parallel default | yes, sequential | **no** | yes, parallel default | no |
| **Foreign-hook composition** | clobbers | overwrites; now warns on stale hooksPath | **migration mode** (chains) | overwrites | **backs up**, restorable | n/a |
| **Cross-platform** | forces `sh` | native binary | isolated per-lang env | thin Node CLI | Ruby-dependent | shebang+exec-bit |
| **Monorepo** | manual `cd` | multi-config/extends | root-only | none | not a focus | n/a |
| **Escape hatches** | `HUSKY=0`, `-n` | `LEFTHOOK=0` | `SKIP=`, `--no-verify`, `manual` | `--no-verify` | `SKIP=`, `ONLY=`, locks | `--no-verify` |
| **Uninstall/drift** | none; stale hooksPath trap | `uninstall`; `validate` | `uninstall` restores state | manual; none | `uninstall` restores backup | n/a |

### Design lessons for Reinicorn

1. **Stay in the pre-commit/Overcommit family (chain + restore).** Reinicorn's
   marker-append already does "chain, don't clobber" but not "capture prior
   state for a clean uninstall." Cheap fix: snapshot the pre-marker prefix at
   install so `hooks uninstall` slices back to it exactly.
2. **Stay off `core.hooksPath`.** It's the most-litigated choice surveyed
   (git-lfs/Mastodon break; Lefthook's #1248 retrofit). Reinicorn already writes
   `.git/hooks/<hook>` directly — keep that, but **detect a foreign
   `core.hooksPath` at install and warn/fail loudly**, else (e.g. Husky present)
   Reinicorn's hooks are silently ignored by git.
3. **Multi-command per event is the crux.** Independently-declared
   per-principle commands appended into one shell file = Husky's shape, where
   principle #3 failing silently blocks #4–N with no attribution. Track each
   declared command as a named unit tied to its **principle ID** so failures
   attribute back and a `hooks status` can list declared-vs-installed per
   principle.
4. **Two escape-hatch levers.** git-native `--no-verify` (blunt) + a scoped
   `REINICORN_SKIP=P14`-style per-principle skip; Overcommit's `required: true`
   maps to non-negotiable principles.
5. **Drift detection is the ecosystem's weak spot → a differentiation
   opportunity.** No surveyed tool diffs declared-config vs installed-hook.
   Because enforcement lives in principle frontmatter (structured, versioned,
   separate from the hook file), a `hooks check`/`doctor` diffing declared vs
   installed is a natural, uniquely-clean fit.
6. **Worktrees + atomicity are unsolved industry-wide** (see `prek`
   [#1672](https://github.com/j178/prek/issues/1672) — worktree-safe,
   transactional install being designed now). Reinicorn's read→decide→append is
   non-atomic; needs explicit test coverage for install/uninstall from a linked
   worktree and concurrent-install races.

### Surprising / contentious

- Husky's `core.hooksPath` takeover is the ecosystem's most-litigated decision
  ([mastodon#24884](https://github.com/mastodon/mastodon/issues/24884)).
- Husky v9 silently forces `sh` even with a different shebang.
- Lefthook's foreign-`core.hooksPath` detection was a retrofit
  ([#1248](https://github.com/evilmartians/lefthook/issues/1248)), so
  Husky→Lefthook migrations could silently no-op for years.
- pre-commit's env-provisioning slowness spawned a Rust rewrite (`prek`, ~10x
  faster install; now used by CPython/FastAPI/Airflow)
  ([j178/prek](https://github.com/j178/prek)).
- Worktree-safe transactional install is unsolved as of 2026 — get it right from
  scratch, nothing to crib.
- Overcommit's README explicitly flags auto-running repo-declared hooks as a
  supply-chain risk — **directly applicable**: principle frontmatter is repo
  content an attacker-controlled branch could edit, so the spec needs a
  trust-boundary note (confirm/audit before installing a frontmatter-declared
  command).

## Notes

Full raw research report (with every source URL) archived in the run scratchpad;
inlined above in condensed form.
