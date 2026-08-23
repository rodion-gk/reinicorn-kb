---
type: spec
title: Remove the kb submodule
slug: remove-the-kb-submodule
lifecycle: active
status: approved
created: 2026-07-26
author: Michael Biehl
origin: ai-assisted
human_validated: false
review_pr: https://github.com/crystldm/reinicorn-kb/pull/8
---
# Remove the kb submodule

## Problem

The kb is a git submodule at `kb/`, so the parent repo records a specific kb
commit. Nothing wants that commit.

`ensure_kb_on_main()` (`src/reinicorn/kb.py:82`) runs on sync, publish, review,
and post-checkout, and forces the kb onto `main`. `stage_kb_pointer()`
(`kb.py:92`) then runs after those same operations and does `git add kb`, so
the next parent commit carries whatever `main` happened to be at that moment.
The recorded SHA therefore means "kb main as of some arbitrary past commit" —
and anyone who checked out that SHA deliberately would be dragged back to
`main` by the next `rcorn` command. `.gitmodules` already declares
`branch = main`, which is the real intent leaking through a mechanism built for
pinning.

Four concrete costs:

1. **Pointer-bump PRs.** `main` is protected (`main-pr-gate`, `main-safety`;
   PR + status checks required, bypass limited to a repository role), so
   refreshing a stale pointer means a PR whose entire content is a 40-byte SHA.
   #25 is one.
2. **Drift, then dangling pointers.** The pointer only advances when someone
   happens to commit in the parent, so every kb-only publish leaves it behind.
   Worse, doc-creating commands commit to the kb locally without pushing, so a
   parent commit can reference kb commits that exist nowhere else; CI then fails
   at `submodules: recursive` checkout. Filed as #21.
3. **CI lints a stale kb.** All four workflows (`test`, `lint-kb`,
   `lint-architecture`, `doc-gardening`) use `submodules: recursive`, which
   checks out the *pinned* commit. On the #22 run that was four commits behind,
   so `Lint Kb` and doc-gardening never saw the newest spec drafts.
4. **It multiplies per consumer repo.** `rcorn init` runs `setup_submodule` for
   every adopting repo. One shared kb, N independently drifting pointers, N
   streams of bump commits.

### The pointer cannot be made inert

The prior draft of this spec proposed neutralizing the pointer with
`submodule.kb.ignore = all` rather than removing the submodule. That premise is
false. Measured with `git ls-files -s` / `rev-parse HEAD:docs` plumbing rather
than `git status` (which `ignore = all` itself suppresses, and which produced
the original false reading):

| mode | drift visible in `status` | pointer after `git add -A` + `git commit` |
|---|---|---|
| default | yes | **moved** |
| `ignore = all` | no | **moved** |
| `git update-index --assume-unchanged kb` | no | frozen |
| `git update-index --skip-worktree kb` | no | frozen |

`ignore = all` suppresses *display* in `status` and `diff` and changes nothing
about staging or committing. An explicit `git add kb` stages the new SHA under
it too. It is strictly worse than the default: the pointer still drifts into
every commit, and now nothing warns you.

This matters beyond the discarded draft, because `setup_submodule()`
(`src/reinicorn/submodule.py:132`) **already writes `ignore = all`** into
`.gitmodules` on every `rcorn init`, and prints "Kb added as submodule tracking
main branch (ignore=all)". Every repo initialized by current `rcorn` has the
blindfold. This repo's own `.gitmodules` predates that line and does not.

The two flags that do freeze the pointer are index bits, not `.gitmodules`
config: they live in `.git/index`, so they are per-clone, invisible to
teammates, not committable, and would have to be re-applied by `rcorn` on every
fresh clone and every worktree. That is a local-state workaround for a
structural problem.

### `ensure_kb_on_main()` is broken in two ways

Separately, the guard that keeps the kb usable is broken. It is:

```python
r = run_git("symbolic-ref", "--short", "HEAD", check=False, cwd=kb_dir)
if r.returncode != 0 or r.stdout.strip() != "main":
    run_git("checkout", "main", check=False, cwd=kb_dir)
```

It never fetches, and it discards both the exit code and the output. Two
verified failure modes:

- **It can move the kb backwards.** From a detached HEAD at `origin/main`'s tip,
  `git checkout main` lands on the *local* `main` ref, which may be stale.
  Reproduced: content silently reverted from `v2` to `v1`. Git prints "Your
  branch is behind 'origin/main'"; `check=False` throws it away.
- **It can strand commits off-branch.** If the checkout fails — uncommitted kb
  work conflicting with `main` — it exits 1 and HEAD stays detached.
  `commit_kb()` swallows that and proceeds to `git add -A` and `git commit`
  anyway. Reproduced: the commit lands, is not an ancestor of local `main`, and
  `push origin main` does not carry it. Recoverable only via the reflog. A doc
  would appear committed and silently never publish.

Nothing detaches the kb today, which is the only reason this has not bitten.
Removing the submodule makes the kb an ordinary clone that `rcorn` fully owns,
which is the right moment to fix this rather than carry it forward.

### What the submodule genuinely buys

Being fair to the mechanism, four things are lost by removing it:

1. **A declared, machine-readable link.** `.gitmodules` states that this repo
   relates to that repo at that path, discoverable without reading docs.
2. **`git clone --recurse-submodules` bootstrap** in one command.
3. **GitHub renders `kb/` as a linked submodule** in the web UI.
4. **A tracking boundary git enforces.** Git will not track the kb's files into
   the parent tree.

(1) is already duplicated: `REINICORN_KB_REMOTE` in `.reinicorn-config` records
the same URL, and `rcorn` reads that file already. (2) delivers a *stale* kb by
construction — the pinned commit — so the one git-native bootstrap hands new
contributors old docs with no signal that they are old. (3) is cosmetic. (4) is
real and is addressed in §3.

## Design Goals

- No pointer exists, so it cannot drift, dangle, or generate bump PRs.
- CI always reads the current kb `main`, never a snapshot.
- A fresh clone reaches a working, current kb in one documented command.
- No normal workflow commits anything under `kb/` into the parent repo, and
  CI rejects anything that slips past the local guards.
- `commit_kb()` never commits unless the kb is genuinely on an up-to-date
  `main`; if it cannot get there, it fails loudly.
- Existing repos are migrated automatically, without a manual checklist.

## Design

### 1. `kb/` becomes a gitignored clone

`rcorn init` clones the kb remote to `kb/` as an ordinary repository on `main`,
adds `kb/` to `.gitignore`, and writes no `.gitmodules` entry. `setup_submodule()`
in `src/reinicorn/submodule.py` is replaced by a `setup_kb_clone()` that keeps
the parts worth keeping — empty-remote detection and seeding
(`is_remote_empty`, `seed_remote`), URL validation, stale-state cleanup — and
drops `git submodule add` and the two `git config -f .gitmodules` calls.

### 2. Discovery moves behind `get_kb_dir()`

`get_kb_dir()` (`kb.py:36`) is the single seam. Today it parses `.gitmodules`
for `path =`. It becomes: `<root>/kb` when that directory contains a `.git`,
with `KB_DIR_NAME` still the constant. The path-traversal guard at `kb.py:52-64`
becomes unnecessary, because the path is no longer repository-controlled input —
that guard exists specifically because `.gitmodules` is attacker-controllable in
a PR.

This is why the change is smaller than the file count suggests. Roughly twenty
call sites across `commands/` reach the kb through `get_kb_dir()` /
`require_kb_dir()` and need no edit at all. `_parse_kb_submodule_path()` and
`init.py:_has_kb_submodule_path()` are deleted.

The kb URL is read from `REINICORN_KB_REMOTE` in `.reinicorn-config`, which
already exists and is already documented as "Set this when using the submodule
layout" — that comment gets updated, the key does not move.

### 3. The boundary: `.gitignore` plus a pre-commit guard

Without a submodule entry, git does *not* sweep the kb's files into the parent —
verified: `git add -A` over an untracked nested repo commits a single `160000`
gitlink and zero files under it. The real failure mode is therefore not a
1000-file diff but an **orphan gitlink**: a recorded SHA with no `.gitmodules`
entry to resolve it, which reintroduces exactly the drift this spec removes and
is more confusing than the submodule was.

Three layers:

- `kb/` in `.gitignore` (parent repo, and written by `init` into consumer
  repos). This alone prevents the accident in normal use.
- A `pre-commit` hook that refuses any staged path at or under `KB_DIR_NAME`,
  with a message naming `rcorn kb publish` as the thing the user probably wants.
  `HOOK_NAMES` in `src/reinicorn/commands/hooks_install.py:14` is currently
  `("post-checkout", "post-merge", "pre-push")`; this adds a fourth, plus a new
  `hooks/pre-commit`. The install mechanism itself is unchanged.
- A CI check that fails when the parent tree tracks any path under `kb/` or a
  mode-`160000` `kb` gitlink — the server-side backstop, and the only layer a
  contributor cannot bypass.

The hook is the part worth insisting on locally: `.gitignore` is a convention
that `git add -f` and a stray `git submodule add` both defeat. But the hook is
client-side too — `git commit --no-verify` skips it — which is why the CI
check is the layer the goal above actually rests on.

### 4. Bootstrap: `rcorn kb sync` clones when absent

`cmd_sync()` gains a clone-if-missing branch before its existing fetch/merge.
`require_kb_dir()`'s error text changes from "No kb submodule found" to point at
`rcorn kb sync` for the fresh-clone case and `rcorn init` for the never-set-up
case — today both land on `rcorn init`, which is wrong for a teammate who just
cloned.

`commands/home.py:42-46` currently prints "kb: submodule not initialized" and
suggests `git submodule update --init kb`. It becomes "kb: not cloned yet" →
`rcorn kb sync`.

`post_checkout.py:50-63` currently runs `git submodule update --init` on a
fresh clone or new worktree. It clones instead, dropping the
`_kb_reference_args()` local-reference optimization or reworking it against
`git clone --reference`.

The result is one command instead of `--recurse-submodules`, and it yields
current docs rather than a stale pin.

### 5. `ensure_kb_on_main()` gains a fetch and a return value

Rewrite it to mean "the kb is on an up-to-date `main`", and to report failure:

- `git fetch origin main` first, so "main" means `origin/main` rather than a
  possibly-stale local ref.
- If HEAD is not `main`, `git checkout main`; then fast-forward local `main` to
  `origin/main`. Never move the working tree backwards.
- Return success/failure instead of discarding it. Keep uncommitted kb work
  intact — abort rather than force.

`commit_kb()` must check that result and refuse to stage or commit when it
reports failure, printing what is wrong and how to fix it (per golden
principle 4). The precondition is branch state, not a pristine worktree — the
uncommitted doc edits are exactly what `commit_kb()` exists to commit: HEAD on
`main`, local `main` not behind `origin/main`. Staging remains `git add -A`
inside the kb, which holds only kb docs, so there is no unrelated-work class
to exclude. Committing into a detached HEAD is never acceptable: the work
looks saved and is not.

Fetch cost: one network round trip per kb-writing command. `sync` and `publish`
already hit the network; `doc create` and `plan create` do not, so this makes
them slower. Throttling the fetch to once per N seconds is deliberately left out
of the first cut.

### 6. Delete the pointer plumbing

`stage_kb_pointer()` and its four call sites — `commands/sync.py:59`,
`commands/publish.py:50`, `commands/internal/post_checkout.py:63`, and
`kb.py:commit_kb` — are removed outright. There is no pointer to stage.

### 7. `repo_root()` loses superproject detection

`git.py:repo_root()` runs `git rev-parse --show-superproject-working-tree` so
that an `rcorn` command invoked from inside `kb/` resolves against the parent
project rather than the kb. That call returns empty for a plain nested clone, so
running `rcorn` from inside `kb/` would silently treat the kb as the project
root — wrong scope, wrong plans, potentially docs written into `kb/kb/`.

Replacement: after `--show-toplevel`, walk up from the result looking for a
`.reinicorn-config`; if the toplevel is a `kb` directory whose parent has one,
use the parent. This must be covered by a test that runs a command with `cwd`
inside `kb/`, because nothing else will catch the regression.

### 8. The pre-push guard gets simpler

`pre_push.py:_ensure_kb_pushed()` currently reads `git rev-parse HEAD:kb` to
learn which kb commit the parent references, then checks that SHA is an ancestor
of `origin/main` and pushes if not. With no gitlink, `rev-parse HEAD:kb` fails
and the guard returns 0 — it must be rewritten, not deleted, or #21's failure
mode returns in a new form.

The rewrite is simpler and strictly broader: push if local kb `main` is ahead of
`origin/main`, regardless of what the parent does or does not reference. That is
the invariant actually wanted — the kb is always pushed — and it no longer
depends on a pointer.

### 9. CI stops using `submodules: recursive`

All four workflows (`test.yml:24`, `doc-gardening.yml:22`, `lint-kb.yml:23`,
`lint-architecture.yml:22`) drop `submodules: recursive` from `actions/checkout`
and gain a clone step:

```yaml
      - name: Clone kb
        run: git clone --depth 1 https://github.com/crystldm/reinicorn-kb.git kb
```

This also fixes the failure CodeRabbit flagged on the discarded draft: with
`submodules: recursive`, a dangling pin fails the job during checkout, before
any later step can correct it. With no pin, there is nothing to dangle.

Fork PRs work unchanged — the kb is public and the clone is unauthenticated,
with no token and no secret.

### 10. Migration for existing repos

This is a breaking change to `rcorn init` and to every repo that already adopted
Reinicorn. `upgrades/` is currently just a README, so the release-notes path
gets its first real entry.

`rcorn init` (and `rcorn update`) detect a kb submodule by inspecting the index
for a mode-`160000` `kb` entry, not just `.gitmodules` — an orphan gitlink with
no `.gitmodules` section (the state §3 warns about) migrates too, with the
steps that assume a registered submodule (deinit, `.gitmodules` edits) skipped
when not applicable — and migrate in place:

1. Check the old `kb/` for uncommitted or unpushed work and refuse the
   migration until it is published. This runs before anything destructive:
   every later step assumes the old worktree is disposable.
2. `git submodule deinit -f kb`
3. `git rm --cached kb` and remove the `[submodule "kb"]` section from
   `.gitmodules`; delete `.gitmodules` if it becomes empty
4. `git config --remove-section submodule.kb`
5. Move `.git/modules/kb` aside or re-clone `kb/` fresh from
   `REINICORN_KB_REMOTE`
6. Add `kb/` to `.gitignore`
7. Install the `pre-commit` hook

Steps 2–4 leave staged changes in the parent that the user must commit; the
command prints exactly what to commit rather than committing on their behalf.
Step 1 is deliberately first — losing a draft to a migration would be
unforgivable, and steps 2 and 5 are where that would happen.

## Non-Goals

- **Freezing the pointer with `--skip-worktree` or `--assume-unchanged`.** Both
  verifiably work, and both are per-clone index bits: invisible to teammates,
  not committable, re-applied by `rcorn` on every clone and worktree, and
  silently absent for anyone who does not run `rcorn`. Rejected as local state
  standing in for structure. If removal proves too disruptive, this is the
  fallback.
- **Keeping `.gitmodules` as documentation** with the submodule unregistered.
  Not a coherent git state, and it would keep `get_kb_dir()` parsing a file that
  no longer describes reality.
- **A CI job that bumps the pointer** on merge or on a schedule. Moot with no
  pointer. It was also independently unattractive: it races with human merges
  (the race that produced #25), moves commit noise into `main` rather than
  removing it, and pushing to a protected `main` needs a bypass actor or an
  admin PAT stored in a public repo.
- **Historical docs-with-code reproducibility.** A meaningful pin would require
  removing `ensure_kb_on_main()`'s force-to-main entirely — a different
  philosophy about what the kb is, not a tweak. Out of scope, and this spec
  forecloses it: revisiting would mean reintroducing a submodule.
- **Fetch-throttling `ensure_kb_on_main()`.** Noted in §5; not in the first cut.
- **Multi-kb or non-`kb/` paths.** `.gitmodules` allowed a custom `path =`;
  after this, the location is `KB_DIR_NAME` by convention. No known consumer
  uses a custom path.
- **Fixing the review-lane enforcement gap** (#23) or the unpushed-kb dangling
  pointer warning (#21). #21's mechanism is replaced by §8, but the pre-push
  check that issue asks for is tracked separately.
