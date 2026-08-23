---
type: spec
title: reins update command
slug: reins-update-command
lifecycle: done
status: implemented
created: 2026-03-07
author: Michael Biehl
origin: ai-assisted
human_validated: false
---

# reins update command

## Problem

`reins init` copies skills, hooks, linters, AGENTS.md, and doc-type
templates into the user's repo as a one-shot operation. When reins itself
is updated (via `uv` pointing at the git repo), there is no mechanism to
propagate new or improved assets into existing repos. Users are stuck on
whatever version they initialized with unless they manually re-copy files.

Additionally, reins's scaffolding evolves over time — templates gain new
sections, skills get refined, doc-type definitions expand. The community
collectively improves these, and downstream users should be informed of
changes so they can adapt their existing docs if they choose.

## Design Goals

1. A single command (`reins update`) syncs repo-local assets with the
   installed package version.
2. User modifications to assets are preserved — never silently overwritten.
3. Users are informed of what changed and why, especially for template and
   guidance changes that affect how they write harness docs.
4. The update process is safe to run repeatedly (idempotent).

## Design

### Command: `reins update`

Two phases: asset sync, then upgrade notes.

### Phase 1: Asset Sync

Compare installed package assets against the repo's copies using a manifest
file that tracks what was last installed.

**Behavior for each asset file:**

| Repo state | Manifest state | Action |
|---|---|---|
| Matches manifest checksum | Exists | Overwrite with new version |
| Differs from manifest checksum | Exists | **Skip** — user modified it |
| File missing | Exists | **Ask** — prompt user to re-add or skip |
| File missing | Missing | **Add** — new asset in this version |
| N/A | Exists but no package source | **Warn** — asset removed upstream |

On skip, print a clear message:
```
Skipped .claude/skills/brainstorming/SKILL.md (locally modified)
  Run: reins update --diff brainstorming to see upstream changes
```

**Summary output:**
```
reins update: v0.1.0 → v0.2.0
  Updated: 12 files
  Added:   2 files (new in v0.2.0)
  Skipped: 2 files (locally modified)
```

**Assets covered:**
- `.claude/skills/` — all skill directories
- `.claude/hooks/` — enforcement hooks
- `linters/` — lint rule configs
- `AGENTS.md` — agent instructions

### Phase 2: Upgrade Notes

After asset sync, display relevant upgrade notes for versions between the
user's last-installed version and the current version.

**Upgrade notes** are shipped in the package at
`src/reins/assets/upgrades/<version>.md`. Each file describes what
changed in that version's templates, skills, and guidance — and why.

Example `upgrades/v0.2.0.md`:
```markdown
# v0.2.0 Upgrade Notes

## Template changes

- `reins doc create principle` now includes an `Enforcement:` section.
  If you have existing principles, consider adding enforcement notes to them.

## Skill changes

- `brainstorming` skill now requires success criteria before proposing
  approaches. This was added based on community feedback that designs
  without clear success criteria led to scope creep.

## New doc types

- `runbook` — step-by-step operational procedures.
```

Notes are displayed inline after the asset sync summary. No automated
harness rewriting — just inform the user so they can adapt existing docs
at their discretion.

### Manifest: `.reins/manifest.json`

Tracks the installed version and checksums of all managed assets.

```json
{
  "reins_version": "0.1.0",
  "installed_at": "2026-03-07T15:30:00Z",
  "files": {
    ".claude/skills/brainstorming/SKILL.md": {
      "sha256": "abc123..."
    },
    "AGENTS.md": {
      "sha256": "def456..."
    }
  }
}
```

Written by `reins init` on first setup and updated by `reins update`
after each sync. The manifest itself is committed to the repo so teammates
share the same baseline.

### Version detection

`reins update` compares `manifest.reins_version` against the
currently installed package version (from `importlib.metadata`). If they
match, it reports "Already up to date" and exits.

### `--diff` flag

`reins update --diff <asset-name>` shows a diff between the user's
current version of a file and the upstream version from the package. Useful
for files that were skipped due to local modifications.

## Non-Goals

- **Rewriting user harness content.** The user's golden principles, design
  docs, ideas, etc. are theirs. We inform, not overwrite.
- **Rich interactive TUI.** Deferred to post-MVP.
- **Automatic conflict resolution or merge tooling.** If a file was modified,
  we skip it and let the user handle it. Merge tooling is post-MVP.
- **Skill extension framework.** A formal system for extending/overriding
  shipped skills without modifying them. Post-MVP.
- **Distribution path decisions.** This design assumes `uv` pointing at the
  git repo. Distribution method is orthogonal to the update mechanism.
