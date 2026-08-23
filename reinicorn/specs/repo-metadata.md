---
type: spec
title: repo-metadata
slug: repo-metadata
lifecycle: active
status: draft
created: 2026-03-21
author: Claude
origin: ai-assisted
human_validated: false
---

# repo-metadata

## Problem

In a multi-repo harness, an agent working in repo-A has no way to understand what repo-B is without cloning it. Each `harness/{repo}/` scope contains plans, design docs, and principles — but nothing that answers the basic question: "What is this repo, where does it live, and what's it built with?"

Agents need a lightweight, instantly-discoverable summary of each connected repo so they can reason about cross-project context without leaving the harness.

## Design Goals

- An agent reading `harness/{repo}/REPO.md` immediately knows that repo's name, purpose, remote URL, default branch, and primary stack.
- The file is auto-generated during `reins init` with minimal developer input (purpose and stack prompted; name, remote, and branch inferred from git).
- Follows existing harness conventions: markdown, uppercase filename, lives at the repo-scope root alongside golden-principles.md.

## Design

### File: `harness/{repo_slug}/REPO.md`

```markdown
# {repo_slug}

{purpose — one sentence}

| Field | Value |
|-------|-------|
| Remote | {git remote URL} |
| Default branch | {default branch} |
| Stack | {primary language/framework} |
```

### Creation flow (during `reins init`)

1. `repo_slug` and `remote_url` — already derived from git remote.
2. `default_branch` — detected from git.
3. `purpose` — prompted interactively (one-liner).
4. `stack` — prompted interactively (e.g. "Python", "TypeScript/Node", "Go").

`generate_seed_tree()` gains these parameters and writes `REPO.md` alongside golden-principles.md and quality-scores.md.

For blank/template harness seeds (no git context), purpose and stack use placeholder text matching the existing template pattern.

### What changes

1. **`harness_seed.py`** — `generate_seed_tree()` gains `purpose`, `stack`, `remote_url`, `default_branch` params and writes `REPO.md`.
2. **`commands/init.py`** — prompts for purpose and stack, passes them + git-derived fields to `generate_seed_tree()`.
3. **Tests** — update multi-repo init tests to verify `REPO.md` is created with expected content.

## Non-Goals

- No new doc type in `REGISTRY` — `REPO.md` is structural metadata, not a managed document.
- No separate CLI command — creation is part of init, not a standalone workflow.
- No linter rules — the file is informational, not enforceable.
- No cross-repo dependency tracking or service maps — that's a future concern.
