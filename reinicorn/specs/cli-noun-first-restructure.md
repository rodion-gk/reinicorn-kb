---
type: spec
title: CLI noun-first restructure
slug: cli-noun-first-restructure
lifecycle: done
status: implemented
created: 2026-04-25
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# CLI noun-first restructure

## Problem

The reins CLI surface mixes two organizational patterns and has accreted shortcuts that produce real bugs and reduce legibility.

**Inconsistency in doc creation.** Doc creation is reachable through three different shapes:

- `reins doc create <type> "title"` — generic form, accepts seven types
- `reins plan create` — top-level shortcut for plans
- `reins idea "text"` — top-level shortcut for ideas

The `plan` and `idea` shortcuts are not aliases. `reins idea` delegates to the same code path as `reins doc create idea`, but `reins plan create` and `reins doc create plan` are independent implementations with divergent behavior:

| | `reins plan create` | `reins doc create plan "title"` |
|---|---|---|
| Title arg | no (uses branch) | required |
| Template source | reads `_template/*.md` from kb with substitutions | hard-coded inline string in Python |
| Overlap check | yes (`check_overlap`) | no |
| Files created | whatever's in `_template/` | always plan.md + progress.md + decisions.md |

This is a latent correctness bug: which plan you get depends on which command you typed.

**Inconsistency at the top level.** The CLI mixes verb-first (`reins sync`, `reins publish`, `reins lint`, `reins enable`, `reins disable`, `reins incognito`) with noun-first (`reins plan ...`, `reins kb ...`, `reins hooks ...`, `reins doc ...`). Both shapes coexist for similar things — `reins sync` operates on the kb but isn't under `reins kb`; `reins enable` is a mode toggle but isn't grouped with the other toggles.

**Legibility for human/agent collaboration.** Agents are the primary CLI users, but humans watch their command streams. When an agent runs `reins idea "foo"`, a human reading the transcript has to know "idea" is a doc type. When an agent runs a self-documenting form like `reins idea create "foo"`, the structure is obvious at a glance. The current shortcuts hide the type and degrade legibility.

## Design Goals

When this is done:

1. There is exactly one canonical command for any given action. No two commands produce the same artifact via different code paths.
2. Every command is self-documenting: reading it (without help text) reveals what object is being acted on.
3. The CLI is uniformly noun-first: the top-level command names an object; subcommands are verbs scoped to that object.
4. Tab-completion can be type-scoped without a hand-maintained matrix.
5. The structure accommodates future per-type lifecycle verbs (e.g. `design supersede`, `debt resolve`) without restructuring.
6. The current divergent `plan create` / `doc create plan` behavior collapses into a single implementation.

## Design

### Principle

**Noun-first throughout.** Every top-level subcommand names an object (a doc type, the kb, modes, hooks). Verbs live one level down, scoped to the object they act on. The exceptions are commands that operate on reins-the-tool itself and have no natural noun (`init`, `update`, `feedback`, `help`).

This generalizes the pattern that `reins plan` already uses (`plan create`, `plan status`, `plan complete`) to every doc type, and brings the rest of the CLI into the same shape.

### Final command tree

**Doc-type groups** — each doc type is a top-level noun group with its own verbs.

```
reins design    create <title>
reins spec      create <title>
reins debt      create <title>
reins idea      create <text>          # text becomes the captured content
reins plan      create                 # uses current branch; no title
reins plan      status
reins plan      complete [branch]
reins retro     create                 # uses current branch; no title (default heading from branch, matching plan)
reins principle add <title>            # appends to golden-principles.md
```

`principle` uses `add` rather than `create` because it appends an entry to a single file rather than creating a new file. This minor inconsistency is preferred over a misleading `create` verb.

**Kb group** — operations on the kb (intentionally higher-level than raw git).

```
reins kb sync
reins kb publish
reins kb status
reins kb lint
reins kb list
reins kb remove-scope <name> [--force]
reins kb git <args...>                 # raw git passthrough; was `reins kb-git`
```

The verbs `sync` and `publish` are kept (rather than `pull`/`push`) because they are intentionally higher-level than the underlying git operations.

**Mode group** — consolidates three previously top-level toggles.

```
reins mode enable
reins mode disable
reins mode incognito
reins mode status                      # NEW: report the active mode
```

**Top-level commands** — operate on reins-the-tool, no natural noun.

```
reins init [--kb-url URL | --local | --create-remote] [--slug X] [--kb-name X]
reins hooks install
reins update [--diff X]
reins feedback [text]
reins help
```

**Hidden / internal** — git-hook callbacks, not in argparse help.

```
_hook-check, _post-checkout, _pre-push, _post-merge   (unchanged)
_check-path <file>                                    (was: reins doc check-path)
```

`_check-path` is only invoked by hooks. Moving it under the internal-prefix convention removes it from the human-facing surface.

### Removed surfaces

The following are removed entirely with no aliases:

- `reins doc` (and all subcommands: `doc create`, `doc check-path`)
- `reins sync`, `reins publish`, `reins status`, `reins lint` (top-level forms)
- `reins enable`, `reins disable`, `reins incognito` (top-level forms)
- `reins kb-git` (replaced by `reins kb git`)
- `reins idea "text"` as a top-level shortcut (replaced by `reins idea create "text"`)
- `reins plan create` retains its name and current behavior; the duplicate code path in `doc_create.py` is deleted.

### Implementation consolidation

There is currently one creator function per doc type in `src/reins/commands/doc_create.py` (`_create_design`, `_create_plan`, `_create_spec`, etc.) plus a parallel `cmd_plan_create` in `src/reins/commands/plan.py` that uses kb templates with substitutions and runs `check_overlap`.

The restructure deletes `_create_plan` from `doc_create.py` and routes both `reins plan create` and the formerly-overloaded `reins doc create plan` through the canonical `cmd_plan_create` implementation.

After consolidation:

- `commands/plan.py` keeps `cmd_plan_create`, `cmd_plan_status`, `cmd_plan_complete`.
- `commands/doc_create.py` exposes one `cmd_<type>_create` per type plus a shared `_create_typed` helper. The divergent `_create_plan` is deleted.

**Scope note:** the spec originally proposed extending the `_template/`-reading + substitutions pattern from `cmd_plan_create` to every doc type (with shared helpers). The implementation collapsed only the divergent plan path; non-plan creators still use hard-coded inline strings. Adding `_template/` support to design/spec/debt/retro/principle is deferred as follow-up work and tracked separately.

### CLI dispatcher

`src/reins/cli.py` is rewritten so each top-level group adds its own subparser with type-scoped verbs. The single `_dispatch` function fans out by `(group, verb)`. Every group's required `<verb>` makes argparse error messages list valid verbs automatically.

### User documentation

`README.md`, `GETTING-STARTED.md`, kb skills, kb index files, and platform-instructions templates that reference any removed command must be updated. This is mechanical search-and-replace work but spans several files.

## Non-Goals

This design explicitly does not cover:

- **New doc types or new lifecycle verbs.** The structure makes `design supersede`, `debt resolve`, `idea promote`, `spec approve`, etc. natural to add later, but none are added here.
- **Changes to doc templates or kb directory layout.** The `design-docs/`, `product-specs/`, `exec-plans/`, etc. directories and their template contents are unchanged.
- **Changes to git-hook behavior.** Hook callbacks (`_hook-check`, `_post-checkout`, `_pre-push`, `_post-merge`) are unchanged in behavior; only `doc check-path` is renamed to `_check-path`.
- **`reins init` flag restructuring.** `init` is a top-level command that operates on reins itself; its flags are out of scope.
- **Backwards-compat aliases.** A hard break is acceptable because the user base is effectively zero. Old commands are removed, not deprecated.
- **Tab-completion configuration.** This design enables clean type-scoped completion via `argcomplete` or `shtab`, but adding the completion scripts is a separate task.
