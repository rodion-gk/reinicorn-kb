---
type: spec
title: 'review setup: reconcile existing doc-review ruleset bypass actors'
slug: review-setup-reconcile-existing-doc-review-ruleset-bypass-ac
lifecycle: active
status: draft
created: 2026-07-24
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# review setup: reconcile existing doc-review ruleset bypass actors

Closes #15.

## Problem

`cmd_review_setup` treats a ruleset named `reinicorn-doc-review` as fully
installed by name alone (`src/reinicorn/commands/review.py`, ~line 465). The
required push-capable RepositoryRole bypass actors `{2, 4, 5}` (maintain, write,
admin) are only sent when *creating* a ruleset.

A repo configured by an earlier Reinicorn version can retain the same-named
ruleset with a missing maintain-role bypass (e.g. only `{4, 5}`). Re-running
setup reports `doc-review ruleset already installed` and does nothing, leaving
maintain-role collaborators unable to make routine `kb publish` pushes.

## Design Goals

- Detect a same-named ruleset whose bypass actors are missing a required role,
  instead of trusting the name.
- `--force` repairs the drift by merging in the required actors, without
  removing unrelated user-added bypass actors.
- A fully-compliant ruleset (including extra user actors) stays a no-op.
- When the token can't inspect bypass actors, warn — never assume current, never
  overwrite blindly.
- Keep the cleanup-token guidance tied to the ruleset being *active*, not to it
  being fully current.
- No whole-ruleset content hash — that would flag benign user customization and
  make upgrades needlessly destructive.

## Design

### Required actors

Factor the required bypass tuples out of `_RULESET` as
`_REQUIRED_BYPASS = {(a["actor_id"], a["actor_type"], a["bypass_mode"]) for a in
_RULESET["bypass_actors"]}` — `{(2,"RepositoryRole","always"), (4,...),
(5,...)}` — so drift detection and creation share one source of truth.

### Reconciliation in `cmd_review_setup`

When the ruleset listing contains a name match, retain its `id` from the list
response and fetch its full config: `gh api repos/{repo}/rulesets/{id}`.

1. Boundary-validate the detail response. If `bypass_actors` is absent (token
   lacks permission to inspect them), warn with a remediation step and make no
   change — treat as "installed" for the cleanup-token hint but not as verified
   current.
2. Build the installed tuple set. If `_REQUIRED_BYPASS` is a subset → compliant,
   info "already installed", no-op (extra user actors preserved).
3. If required actors are missing (drift):
   - Without `--force`: warn the ruleset is outdated and direct the operator to
     `rcorn review setup --force`. Do not modify.
   - With `--force`: PUT the merged actor set
     (`gh api repos/{repo}/rulesets/{id} --method PUT`) — installed actors
     unioned with the required ones — preserving user-added actors, and report
     success.

The cleanup-token hint fires whenever the ruleset is active (created, compliant,
drifted, or permission-opaque), matching the "tied to active, not current"
goal.

### GitHub interaction (`src/reinicorn/commands/review.py`)

Setup already calls `github.run_gh("api", ...)` directly; keep that style for
the detail-fetch and PUT. Parse JSON at the boundary; on an unparseable detail
response, treat as permission-opaque (warn, no change).

## Non-Goals

- Reconciling rule *parameters* (approval count, dismiss-stale, merge methods) —
  only the bypass-actor drift that breaks pushes is in scope.
- Whole-ruleset hashing or replacing user-customized rules.
- Removing bypass actors, ever — reconciliation is additive (union) only.

## Acceptance Criteria

- A same-named ruleset with `{4, 5}` is detected as outdated (warn without
  `--force`).
- `review setup --force` upgrades it to include `{2, 4, 5}` without removing
  unrelated existing bypass actors.
- A fully-compliant ruleset (including extra user actors) remains a no-op.
- Missing `bypass_actors` in the detail response produces a remediation warning
  and no update.
- Tests cover the list, detail-fetch, drift, forced update, and
  insufficient-permission paths.
