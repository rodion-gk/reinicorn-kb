---
type: spec
title: 'Doc review lane: PR-style review for gated kb docs'
slug: doc-review-lane-pr-style-review-for-gated-kb-docs
lifecycle: active
status: draft
created: 2026-07-06
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# Doc review lane: PR-style review for gated kb docs

## Problem

Code changes get PR review; the kb docs that drive them do not. Specs land on
kb main with no checkpoint, so nothing distinguishes a validated spec from a
half-formed one, and agents happily build plans on top of either.

A classic PR flow conflicts with kb philosophy: the kb submodule checkout
always stays on `main` (`ensure_kb_on_main` enforces this), publish is
fire-and-forget, and cross-branch awareness depends on everyone seeing kb main
immediately. Nobody should ever have to do branch manipulation inside `kb/` to
edit a doc.

A rendering constraint shapes the whole design: GitHub inline diff comments
only attach to hunks of a PR diff, and rename detection cannot be disabled.
Spiked on the real kb repo (PRs #2–#4, 2026-07-06):

- A branch that renames `drafts/foo.md` → `foo.md` with a status flip collapses
  to `R9x` — only the one-line hunk is commentable. Rejected.
- A PR containing **only an added file** renders every line commentable, even
  when a near-identical draft copy exists on the base branch — rename detection
  pairs adds with deletes *inside* the diff, never with base content.
- Splitting the add and the delete into separate commits does not help: the PR
  view diffs merge-base → head endpoints, then pairs the rename anyway.

So: the draft can live on main the entire time, and review still gets a full
new-file diff — as long as the draft's deletion stays out of the PR.

## Design Goals

- Reviewers use the native GitHub PR experience: inline diff comments on any
  line of the doc, recorded approvals, review-request notifications.
- The local kb checkout never leaves main; review branches are server-side
  plumbing only. Authors (and their agents) keep editing the draft in the
  ordinary working copy with the full kb in context.
- Draft content is visible on kb main from minute one (cross-branch awareness)
  but never mixes into reins surfaces with approved content unless explicitly
  requested (`--include-drafts`).
- Gating is per doc type via the registry, not hardcoded: a `gated` flag on
  `DocType`. v1 gates `spec` only.
- Enforcement is medium strength: lint warnings + skill-level checks, no hard
  CLI blocks.
- Approval integrity has three layers: GitHub dismiss-stale-approvals (where
  available), a reins divergence check at merge, and lint.

## Design

### Layout and lifecycle

Approved docs stay flat at the canonical path (`specs/<slug>.md`) — reference
stability is the asset, and "path contains `/drafts/`" is the entire
draft-detection rule. Drafts live in one annex subdirectory:

```
specs/
├── <slug>.md          # approved (or legacy pre-lane) specs — canonical path
├── drafts/<slug>.md   # working drafts, on main, never surfaced by default
└── index.md           # approved docs only
```

Frontmatter status: `draft` → `in-review` → `approved`.

1. **`reins spec create "<title>"`** writes `specs/drafts/<slug>.md` with
   `status: draft`. Publishes to kb main as today. Author + agent iterate
   directly on the draft; no branches.
2. **`reins review start <slug>`** — temp-clone of the kb repo (same machinery
   as `kb seed`), branch `review/<repo-scope>/spec-<slug>` from origin/main
   (repo-scoped to prevent cross-repo slug collisions in a shared kb), write the
   **candidate**: a copy of the draft at the final path `specs/<slug>.md` with
   `status: in-review`. Push ref, `gh pr create`, request reviewers. The
   on-main draft's frontmatter gets `status: in-review` + `review-pr: <url>`
   so `kb status` shows the review without a gh call. Prints the PR URL as
   the primary success output — humans driving the CLI see it directly,
   agents driving it surface it to the human. `review push` and
   `review status` repeat the URL in their output for the same reason.
3. **`reins review push <slug>`** — re-sync the candidate from the current
   draft and push to the review ref. Guard: the ref may only ever touch the
   single candidate file; anything else aborts.
4. **`reins review merge <slug>`** — verifies the PR is approved (gh), merges
   (squash), then one mechanical publish commit on main: flip the landed file
   to `status: approved`, stamp `approved-by` + `review-pr`, delete
   `drafts/<slug>.md`. Pulls kb main locally. Idempotent: if the PR was merged
   from the GitHub UI, `merge` performs only the cleanup half, and `kb status`
   nudges when it detects a merged-but-uncleaned review.

   **CI cleanup (preferred when available):** `review-setup` also installs a
   GitHub Actions workflow in the kb repo (`.github/workflows/`, triggered on
   merged PRs from `review/*` branches) that runs the same cleanup — flip,
   stamp, delete draft — automatically. This makes a browser-side merge a
   complete action: reviewer approves and merges, kb ends consistent, nobody
   needs a terminal. The cleanup logic is one shared implementation (an
   internal `review cleanup` entry point) invoked by both `review merge` and
   the workflow; it is idempotent and concurrency-safe (pull–rebase–retry, and
   a no-op when the flip/delete already happened), so the CLI and CI racing
   each other is harmless. The workflow's push to main requires the Actions
   app in the ruleset bypass list; where Actions is unavailable or disabled,
   the CLI/nudge path is the fallback. The workflow obtains reins via pip
   (PyPI once published; a pinned git URL until then) — it must not embed a
   copy of the cleanup logic.
5. **`reins review cancel <slug>`** — close PR, delete ref, draft back to
   `status: draft` — but the cancellation is recorded, not erased: the draft
   keeps `review-pr` (now closed) and gains `review-cancelled: <date>`.
   A cancelled-and-then-untouched draft is a first-class signal for doc
   gardening (see the docs-gardening idea docs): stale draft + cancelled
   review + no edits since ⇒ deletion candidate. Re-starting review later
   clears the marker.
6. **`reins review link <slug> <pr-url>`** — record a manually-created PR on
   the draft (`status: in-review` + `review-pr`). This is the second half of
   the no-gh escape hatch (below), and also covers "someone opened the PR by
   hand anyway".
7. **`reins review status`** — all open doc reviews: slug, status, PR link,
   reviewer state (one gh call, degrades gracefully offline).

Slug resolution: `review` commands search the `drafts/` dirs of all gated
registry types; on a cross-type collision the command errors and asks for
`--type`. Commands also accept the file path directly.

### No-gh escape hatch

`gh` gates only conveniences, never the lane: the review ref is pushed with
plain git, so every step degrades to "reins does the git half, the human does
the GitHub half" with reins printing exactly what to do:

- **start** without gh: push the ref anyway, print the PR-creation link
  (`https://github.com/<owner>/<kb-repo>/pull/new/review/<repo-scope>/spec-<slug>`), and
  instruct: create the PR there, then run `reins review link <slug> <pr-url>`.
  Agents relay the link and instruction to the human verbatim.
- **push** needs no gh at all (pure git); it reminds that approvals reset.
- **merge** without gh: can't verify approval or merge remotely — print the PR
  link and ask the human to merge in the UI. Merge detection needs no gh
  either: after `git fetch`, the candidate existing at the final path on
  origin/main means merged, so the local cleanup half (or CI) proceeds.
- **cancel** without gh: delete the ref and mark the draft cancelled (both
  git/local), print the PR link with a request to close it manually.

### Registry

`DocType` gains `gated: bool = False`. v1 sets `gated=True` on `spec`. The
drafts subpath, review command applicability, surfacing filters, and lint rules
all derive from the flag. Whether `prd` keeps existing (beta-cleanup keeps it
per the current plan) is orthogonal — gating it is a one-line flag flip.

### Surfacing rules

Mechanical rule: any path under `<type>/drafts/` is excluded from every reins
surface unless `--include-drafts` is passed.

- `reins spec show/list` (and all registry-derived doc reads): drafts excluded
  by default; with the flag, rendered with an explicit `[DRAFT]` marker.
- `specs/index.md`: approved docs only, never drafts.
- `reins kb status` / `reins home`: an "in review" section listing drafts by
  slug/status/PR — state is surfaced, content is not.

### Approval integrity

- **GitHub layer (best-effort):** a ruleset on kb `main` — require PR,
  required approvals ≥ 1, dismiss stale approvals on push — with the team in
  the bypass list so direct `kb publish` pushes keep working. Applied
  idempotently by a `review-setup` step in `reins init`/`hooks install` via
  `gh api`; if the plan/permissions don't allow it (private-repo protection
  needs a paid plan), reins reports and continues.
- **reins layer (floor):** `review merge` refuses when the draft has changed
  since the last `review push` (candidate ≠ draft ⇒ warn, require re-push +
  re-review, or explicit confirmation).
- **Lint layer:** a plan referencing a `/drafts/` path or a doc with
  `status: in-review` gets a lint warning ("building on unapproved spec").
  Legacy specs at the canonical path predating the lane are exempt — no
  migration means no retroactive warnings.
  writing-plans / executing-plans skills instruct agents to check status and
  stop for explicit user confirmation before building on a draft.

### Failure modes

- `gh` missing or unauthenticated → every verb degrades to the no-gh escape
  hatch (git half automated, GitHub half handed to the human with the exact
  link); the auth hint from `github.py` is printed alongside.
- Review ref already exists → "review in progress — use `reins review push`".
- PR merged externally → CI cleanup handles it; without CI, `review merge` =
  idempotent cleanup and `kb status` nudges.
- CI cleanup and `review merge` race → shared idempotent cleanup makes the
  loser a no-op.
- Offline → `review status` and `kb status` degrade to frontmatter-only info.
- Multi-repo kb → drafts and reviews are repo-scoped like everything else.

### Testing

- Unit tests against a local bare-repo remote (existing publish/seed
  fixtures); gh interactions mocked at the `github.py` seam.
- Registry tests for the `gated` flag; lint-rule tests for the drafts
  reference warning; surfacing tests for `--include-drafts` filtering.
- **CI workflow:** the workflow yml is a generated asset installed by
  `review-setup` — unit-test the installer (content matches the shipped
  template, idempotent re-install, refuses to clobber a user-modified file
  without `--force`). The workflow body stays a thin shell (checkout, pip
  install reins, invoke `review cleanup`), so its logic is already covered by
  the shared-cleanup unit tests: merged-PR state → flip/stamp/delete;
  already-cleaned state → no-op; concurrent-push conflict → pull–rebase–retry.
  Validate the yml structurally in the test suite (parseable YAML, expected
  trigger on merged `review/*` PRs, pinned action versions); run `actionlint`
  when available.
- One scripted end-to-end acceptance run on the real kb repo before calling
  the feature done (like the rendering spike): start → push → approve →
  UI-merge → watch the workflow perform the cleanup — then delete the
  test artifacts.
- The rename/add-only rendering behavior is documented above from the live
  spike; no automated test — it's GitHub's behavior, not ours.

## Non-Goals

- Review for continuously-published doc types (plans, retros, ideas): plans
  live on main by design, so there is no off-main content state for a PR to
  render, and a plan's natural review moment is the code PR it produces.
  If plan review is still wanted later, it is a separate design.
- Hard CLI blocks on unapproved specs (enforcement is lint + skills).
- Reviewing kb content via the parent repo's PRs.
- Migration of existing approved specs (they stay where they are; the lane
  applies to new docs).
- Non-GitHub forges (the lane is gh-based; the kb works fine without it).
